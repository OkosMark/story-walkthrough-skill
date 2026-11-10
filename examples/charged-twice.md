# Example: charged twice on Black Friday

The user asked: "Customers say we charged them twice on Black Friday. I do not know the payments code. Walk me through what happened, from the beginning." The store, its code, and its numbers are invented for this example. The story below is the agent's answer.

---

Here is the whole story from the start, without assuming you know the code.

## The characters

**The Pay button.** When a shopper taps Pay in the mobile app, `submitPayment()` in `mobile/src/checkout/pay.ts:34` sends a `POST /payments` request with the order number and the amount.

**The retry.** Every request from the app goes through the shared HTTP client, `apiClient` in `mobile/src/api/client.ts:58`. If a request fails with a 502, 503, or 504, the client sends it again after one second, up to two times. Its rule: *"A 504 means the request did not go through, so sending it again is safe."*

**The API gateway, the waiter.** Think of the backend as a restaurant. The gateway is the waiter: it carries each request from the app to the right service and brings the answer back. The waiter waits at most 30 seconds (`timeout: 30s` in `gateway/routes.yaml:12`). Its rule: *"If the kitchen has not answered in 30 seconds, I go back and say 504 Gateway Timeout."*

**The payment service, the kitchen.** `create_charge()` in `payments/charge.py:41` prepares the dish: it checks the order, asks the bank to charge the card, and records the charge. Its rules: *"I charge an order only if it is not paid yet. Once I start a charge, I finish it, whether anyone is still waiting or not."*

**The bank, the oven.** The card processor approves or declines each charge. The kitchen cannot finish until the oven is done. On a normal day the oven takes about 2 seconds. On a very busy day it can take more than 40.

## Chapter 1: A shopper taps Pay

At 10:02:00 on Black Friday, a shopper taps Pay for order 7731, $84.00. The waiter carries the request to the kitchen and starts its 30-second clock. The kitchen checks order 7731: not paid yet. It puts the charge in the oven.

## Chapter 2: The oven is slow

Black Friday traffic has slowed the bank down. The charge for order 7731 takes 41 seconds. **The kitchen is still cooking when the waiter's 30 seconds run out.**

## Chapter 3: The waiter gives up

At 10:02:30 the gateway stops waiting and answers the app with `504 Gateway Timeout`. It has no way to tell the kitchen to stop, so the payment service keeps waiting for the bank.

From the app's side, the payment failed. **From the kitchen's side, the payment is still going ahead.**

## Chapter 4: The same order, twice

Two things now happen at the same time.

**The retry** sees the 504 and follows its rule. At 10:02:31 it sends the same `POST /payments` for order 7731 again, and the waiter carries it to the kitchen.

**The kitchen** gets what looks like a new order. It checks order 7731: still not paid, because the first charge is still in the oven and nothing is recorded until the oven is done. **The "already paid" check passes, so the kitchen starts a second charge.** Nothing in the request says "this is the same order as before".

At 10:02:41 the oven finishes the first charge, and the kitchen records it. Nobody is waiting for that answer anymore.

At 10:02:52 the oven finishes the second charge, 21 seconds after it started and inside the waiter's 30 seconds. The kitchen records it, the waiter brings back `200 OK`, and the app shows "Payment complete".

## Chapter 5: Two charges, one receipt

The shopper sees one success screen and gets one confirmation email. The bank statement shows two charges of $84.00. The payment service log has both:

> charge succeeded order=7731 amount=84.00 charge_id=ch_9f2a
>
> charge succeeded order=7731 amount=84.00 charge_id=ch_b71c

No error appears anywhere, except one line in the gateway log that looks like an ordinary slow request:

> 504 POST /payments upstream_time=30.0s

## Why only on Black Friday, and almost only from the app

On a normal day the bank answers in about 2 seconds, so the waiter never gives up and the retry never runs. The payment service's latency dashboard shows the bank above 30 seconds only between 10:00 and 13:00 on Black Friday.

The web checkout does not use `apiClient`. Its `fetch` call in `web/src/checkout/pay.ts:22` does not retry. On a 504 it shows "We could not confirm your payment. Check your orders before you try again."

That matches the support tickets: **212 double charges, all between 10:00 and 13:00, and 209 of them from the app.** The other 3 came from web shoppers who saw the error and paid again by hand.

## Why it started now

The retry is two months old. The change that made the app retry failed requests on weak mobile networks (PR 318) added it to `apiClient`, so it covers every request the app sends. That change made sense: most failed requests are `GET` requests, such as loading a product list, and repeating those is harmless. It also covered `POST /payments`, which is not harmless to repeat. Until Black Friday the bank never took 30 seconds, so no payment ever hit a 504.

## How bad it is

- **212 shoppers were charged twice, $19,640 in total (checked).** A query on the payments table for orders with two successful charges on Black Friday returns 212 orders.
- **None of the duplicates are refunded yet (checked).** The refunds table has no entries for those charge IDs.
- **Other requests can repeat the same way (not checked).** The retry also covers `POST /orders` and `POST /gift-cards/redeem`. Whether any orders or gift card redemptions were duplicated has not been checked.

## How to fix it

**The quick fix.** Limit the retry in `apiClient` to `GET` requests. In restaurant terms: when the waiter comes back empty-handed, the app does not order again. It asks whether the first order went through, with `GET /orders/7731`. The limit: a shopper who taps Pay again by hand is still charged twice.

**The proper fix.** Give every payment an idempotency key. The app creates one number per tap of Pay and sends it in an `Idempotency-Key` header, on the first try and on every retry. The kitchen writes that number on its order rail **before** it puts the charge in the oven. When the same number comes in again, the kitchen returns the result of the first charge instead of starting a new one. This works for any repeat: the retry, the gateway, or a shopper who taps twice.

**Not a fix.** Raising the gateway timeout to 60 seconds makes the waiter more patient, but a slower day brings the same problem back.

Recommendation: ship the quick fix today, refund the 212 duplicate charges, and add the idempotency key before the next sale.
