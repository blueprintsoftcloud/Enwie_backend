# Coupon Usage Count on Cancellation: Issue & Implementation Guide

This technical document details the root cause of the coupon deduction issue during payment checkout, the cancellation architecture, and the exact backend and frontend code implementations to resolve it across codebases.

---

## 1. Problem Description & Root Cause

### The Problem
When a customer applies a coupon (especially a single-use coupon with `maxUses: 1`) and proceeds to the payment gateway:
* If the customer cancels the payment popup,
* If the payment fails or is declined, or
* If the order is cancelled by an admin or abandoned session cleanup,

the coupon remains permanently consumed. In the admin dashboard (`CouponManagement.tsx`), the coupon gets labeled as **"Expired"** even though the calendar expiration date has not arrived.

---

### Why the Count Decreased Before Payment Completed
When the customer clicks **"Pay Online"**:
1. The frontend calls `POST /api/order/create` (or checkout endpoint).
2. The backend creates an order record with `orderStatus: "PROCESSING"` and `paymentStatus: "PENDING"`.
3. In that **same initial database transaction** (before the customer even submits card/UPI credentials), the backend increments the coupon usage:
   ```ts
   if (resolvedCouponId) {
     await tx.coupon.update({
       where: { id: resolvedCouponId },
       data: { usedCount: { increment: 1 } },
     });
   }
   ```

---

### Why the Coupon Was Not Restored on Cancellation
When an order was cancelled, the system executed the inventory rollback function (`restoreStockNow()`).  
However, that function **only returned product stock** back into inventory and **completely omitted coupon restoration**. As a result:
* Product quantities were restored.
* The coupon's `usedCount` remained permanently incremented by 1.

---

### Why the Admin Dashboard Showed "Expired"
In `frontend/src/dashboard/CouponManagement.tsx`:
```ts
const isExpired = (c: Coupon) =>
  (c.expiresAt != null && new Date(c.expiresAt) < new Date()) ||
  (c.maxUses != null && c.usedCount >= c.maxUses);
```
When `maxUses` is `1`, once `usedCount` reaches `1`, the condition `usedCount >= maxUses` (`1 >= 1`) evaluates to `true`. The UI marks the coupon as **"Expired"** because all allowed usages were consumed.

---

## 2. Cancellation Architecture

An uncompleted or cancelled order can enter the cancellation pipeline through 4 distinct entry points:

1. **Customer Dismisses Payment Gateway Popup**:  
   The customer closes the Razorpay modal or clicks cancel (`POST /api/order/cancel/:id` -> `cancelOrder`).
2. **Payment Gateway Webhook Failure**:  
   The gateway notifies the server that payment failed or timed out (`POST /api/order/razorpay-webhook` -> `handleRazorpayWebhook`).
3. **Admin Manual Cancellation**:  
   An administrator cancels the order from the admin dashboard (`PUT /api/orders/update-status/:id` -> `updateStatus`).
4. **Automated Abandoned Order Cleanup**:  
   A background job periodically cleans up stale `PROCESSING` orders older than 30 minutes (`cleanupAbandonedOrders`).

All 4 pathways route to a single unified rollback function: **`restoreStockNow()`** in `backend/src/services/inventory.service.ts`.  
Placing the coupon restoration inside this function guarantees that **all 4 cancellation scenarios are restored accurately in one place**.

---

## 3. Backend Implementation

### File: `backend/src/services/inventory.service.ts`

#### Step 1: Import the `Coupon` Model
Add `Coupon` to the Mongoose models import:
```ts
import { Product, ProductVariant, Order, Coupon } from "../models/mongoose";
```

#### Step 2: Add Coupon Restoration Logic in `restoreStockNow()`
Inside `restoreStockNow(data: RestoreStockJobData)`, directly after the atomic `Order.findOneAndUpdate` claim, add the coupon rollback logic:

```ts
const restoreStockNow = async (data: RestoreStockJobData) => {
  // 1. Atomic compare-and-swap idempotency guard
  // Only the FIRST cancellation execution can flip stockRestored from false -> true.
  const claimed = await Order.findOneAndUpdate(
    { _id: data.orderId, stockRestored: { $ne: true } },
    { $set: { stockRestored: true } },
  );
  if (!claimed) {
    logger.info(`[inventory] restore-stock for order ${data.orderId} already applied — skipping (idempotent)`);
    return;
  }

  // 2. [COUPON RESTORATION FIX]
  // Decrement coupon usage if a coupon was redeemed on this cancelled order
  if (claimed.couponId) {
    try {
      await Coupon.updateOne(
        { _id: claimed.couponId, usedCount: { $gt: 0 } }, // Safety check: never drop below 0
        { $inc: { usedCount: -1 } }                        // Restores 1 usage count
      );
      logger.info(`[inventory] Restored coupon ${claimed.couponId} usage for cancelled order ${data.orderId}`);
    } catch (couponErr) {
      logger.error(`[inventory] Failed to restore coupon usage for order ${data.orderId}`, couponErr);
    }
  }

  // 3. Existing inventory stock restoration continues...
  for (const item of data.items) {
    if (item.variantId) {
      await ProductVariant.updateOne({ _id: item.variantId }, { $inc: { stock: item.quantity } });
      await syncProductFromVariants(item.productId);
    } else {
      await Product.updateOne({ _id: item.productId }, { $inc: { stock: item.quantity } });
    }
  }
};
```

---

## 4. Frontend Implementation

### File: `frontend/src/cart/CheckoutPage.tsx`

When the customer dismisses the payment modal or the payment fails, the frontend must inform the backend to release the order and trigger `restoreStockNow()`.

```tsx
let orderClosed = false;
const cancelPendingOrder = async () => {
  if (orderClosed || isPaymentSuccess.current) return;
  orderClosed = true;
  try {
    await api.post(`/order/cancel/${res.data.order.id}`);
  } catch (err) {
    console.error("Cancel failed", err);
  }
};

const options = {
  key: razorpayKey,
  amount: res.data.rzpOrder.amount,
  order_id: res.data.rzpOrder.id,
  handler: async (response: any) => {
    isPaymentSuccess.current = true;
    // ... verify payment
  },
  modal: {
    ondismiss: async function () {
      setIsRazorpayOpen(false);
      setLoading(false);
      if (isPaymentSuccess.current) return;
      // Trigger cancel when customer closes the modal without paying
      await cancelPendingOrder();
      toast.error("Payment cancelled.");
    },
  },
};

const rzp = new (window as any).Razorpay(options);

rzp.on("payment.failed", async function (response: any) {
  toast.error("Payment Failed");
  if (!isPaymentSuccess.current) {
    await cancelPendingOrder();
  }
});

rzp.open();
```

---

## 5. Verification & Safety Guarantees

1. **Idempotency (No Duplicate Decrements):**  
   The atomic check `Order.findOneAndUpdate({ _id: data.orderId, stockRestored: { $ne: true } }, { $set: { stockRestored: true } })` ensures that even if a user double-clicks "Cancel" or a webhook arrives concurrently, only **one** execution proceeds. Subsequent calls return immediately.
2. **Floor Guard at Zero (`$gt: 0`):**  
   The query filter `{ _id: claimed.couponId, usedCount: { $gt: 0 } }` ensures `usedCount` can **never become negative**, even under edge cases.
3. **Isolated Error Boundary:**  
   Coupon restoration is wrapped in an independent `try/catch` block. If the coupon was deleted by an admin, product stock restoration still completes without failure.
4. **Universal Coverage Across All 4 Pathways:**  
   Because customer dismissals, webhook failures, admin cancellations, and abandoned order cleanups all invoke `restoreStockNow()`, the coupon is guaranteed to be restored in every scenario.
