# Example: the stale prices

The user asked: "Walk me through the issue step by step like a story, start from the beginning because I'm not familiar with the issue." The issue was a defect in a small online shop: for a few hours after midnight, some shoppers saw yesterday's prices on product pages, then got a "price changed" warning at checkout. The shop, its code, and its numbers are invented for this example. The story below is the first answer, unchanged except that its headings are one level deeper.

## What to keep and what to fix

Keep:
- One metaphor (a whiteboard next to each web server) carried from the first section to the last, always next to the real name.
- Each character's rule written as its own thought, which makes the collision in Chapter 3 obvious.
- "Why only after midnight, and only on some servers" ties the story back to what the support tickets reported.
- "Why it started now" explains the earlier change without blaming it.

Fix:
- **"The import" was never pinned to code.** The user had to ask "Is the import the cron job or the admin upload page?". The characters section should have said: `run_import()` in `jobs/import_prices.py:18`, started by the 00:05 cron entry, plus the `broadcast_invalidate()` call at `jobs/import_prices.py:52` that sends the wipe message.
- **"Discounted items are charged correctly" was stated as fact and was wrong.** A later read of the promotions code showed that `apply_discount()` takes the base price from the cache, not the database, so discounted orders in the stale window were charged against yesterday's price. The claim should have been marked as not checked, like the newsletter bullet below it.
- **The quick fix in Chapter 8 was presented without a test.** Describing a fix in a story does not verify it. The story should have named the test that would prove it: freeze the clock at 00:10 local time on a server that runs in UTC, run the import, and check the price the product page shows.

## The story

Here is the whole story from the start, without assuming you know the code.

### The characters

**The shop.** The shop sells to customers in one country, in the UTC+3 time zone. Prices change every day, and a new day's prices take effect at local midnight.

**The Price Import.** Every night at 00:05 local time a cron job runs `run_import()` in `jobs/import_prices.py:18`. It loads the new day's price list into the database. Then it tells every web server to forget the old prices for that day. Its rule: *"A new day starts at local midnight."*

**The whiteboard.** Each web server keeps the day's prices in memory, in the price cache in `shop/prices/cache.py`, so it does not ask the database on every page view. Think of it as a whiteboard next to each server. Each day's prices go under a heading with that day's date, such as `prices:2026-03-13`. When the Price Import finishes, it sends a message to all servers that says "wipe the `prices:2026-03-14` heading".

**The key builder.** `price_cache_key()` in `shop/prices/cache.py:41` decides which heading a server reads. It takes today's date from the server's own clock. Its rule: *"Today is whatever date my clock shows."*

**The server clock.** The web servers run in UTC, three hours behind local time.

### Chapter 1: A shopper looks at prices before midnight

At 23:50 local time on 13 March, a shopper opens a product page. The server's clock says 20:50 UTC, 13 March. The key builder picks the heading `prices:2026-03-13`, finds the day's prices on the whiteboard, and shows them. Everything is correct.

### Chapter 2: The new prices arrive

At 00:05 local time on 14 March, the Price Import loads the 14 March prices into the database. It then sends its message: "wipe `prices:2026-03-14`".

Each server looks at its whiteboard. There is no `prices:2026-03-14` heading, so there is nothing to wipe. The 13 March prices stay under `prices:2026-03-13`, untouched. **The import wipes the wrong corner of the whiteboard.**

### Chapter 3: Two clocks, two dates

A shopper opens a product page at 00:20 local time. The two rules now disagree.

- **The Price Import** thinks it is 14 March, because local midnight has passed.
- **The key builder** thinks it is still 13 March, because the server clock says 21:20 UTC.

So the key builder picks `prices:2026-03-13`, finds the old prices still written there, and shows them. **The shopper sees yesterday's prices, and nothing on the server knows they are old.**

### Chapter 4: The warning at checkout

The shopper puts the item in the cart and goes to checkout. Checkout does not trust the whiteboard. `validate_cart()` in `checkout/validate.py:77` reads the price from the database, sees that it differs from the price in the cart, and logs:

> Price changed during checkout for SKU 48213: shown 19.90, current 21.50

The shopper sees "The price of an item in your cart has changed" and has to confirm the new price. Many shoppers then write to support.

### Chapter 5: Why only after midnight, and only on some servers

At 03:00 local time, the server clock reaches 00:00 UTC on 14 March. The key builder now picks `prices:2026-03-14`. That heading is empty, so the server loads the new prices from the database and writes them up. From then on, every page is correct.

A server that started after 00:05 local time had an empty whiteboard. Its first request loaded the 14 March prices from the database, so it showed correct prices from the start, even though it filed them under the wrong date.

That matches the support tickets: **complaints arrive only between 00:05 and 03:00 local time, and only from shoppers served by a server that was running before midnight.** It also explains why the nights with a 01:00 deployment had almost no complaints: the deployment restarted every server.

### Chapter 6: Why it started now

The key builder and the Price Import have used these rules for two years. Until recently the web servers ran in local time, so both rules always agreed on the date.

Three weeks ago, the change that moved the web servers to the company's standard container image (PR 412) went live. The standard image runs in UTC, like every other service in the company. That change made sense on its own: one clock for all logs and all services. But it moved the key builder's clock three hours away from the Price Import's idea of midnight, and that started this bug.

### Chapter 7: How bad is it

- **Shoppers pay the right base price (checked).** Checkout reads the price from the database, as Chapter 4 shows. The `validate_cart()` path was read end to end.
- **Discounted items are charged correctly too.** The discount is applied at checkout, after the price check.
- **The warnings are frequent (checked).** A log query shows about 1,900 "Price changed during checkout" lines per night over the past three weeks.
- **The nightly newsletter may show old prices (not checked).** The "today's deals" email goes out at 01:00 local time, inside the stale window. Where it reads its prices from has not been checked.

### Chapter 8: How to fix it

**The quick fix.** Make the Price Import wipe every `prices:` heading on the whiteboard, not only the one for the new date. Servers then reload from the database on the next request. This removes the symptom but leaves the two clocks disagreeing, so the whiteboard keeps filing prices under the wrong date.

**The proper fix.** Give the shop one definition of "today": a `business_date()` helper that returns the date in the shop's time zone. Both the key builder and the Price Import use it, so they always agree on the heading, whatever the server clock says.

Recommendation: ship the quick fix now, and move both characters to `business_date()` in the next release.
