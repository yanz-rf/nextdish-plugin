---
name: nextdish
description: This skill should be used when the user says "/nextdish", asks to "order from NextDish", "order food", "what should I order today", "recommend a dish", "show me the NextDish menu", or mentions ordering lunch or dinner via NextDish. Handles browsing the menu, filtering by diet/price/cuisine, fetching order history for personalized recommendations, and completing checkout through the Chrome browser session.
version: 1.0.0
---

# NextDish Order Skill

Order food from NextDish via the Chrome browser session.

## Arguments

Parse the user's request for:
- `date` — delivery date, e.g. "04/05", "Sunday", "tomorrow". Default: next available date from API.
- `filter` — e.g. "vegetarian", "veg", "no pork", a cuisine, a dish name, or a restaurant name.
- `max_price` — maximum price per item. Default: no limit.
- `auto` — if user says "just pick one" or "surprise me", auto-select the highest-rated match without asking.
- `recommend` / `history` — if user says "recommend based on history" or "what should I order", fetch order history first and use it to rank suggestions.

## Step 1 — Find the NextDish Chrome tab

Run AppleScript to find a tab with `nextdish.com` in the URL across all windows:

```applescript
tell application "Google Chrome"
  set wIdx to 0
  repeat with w in windows
    set wIdx to wIdx + 1
    set tIdx to 0
    repeat with t in tabs of w
      set tIdx to tIdx + 1
      if URL of t contains "nextdish" then
        return "w:" & wIdx & " t:" & tIdx
      end if
    end repeat
  end repeat
  return "none"
end tell
```

If no tab is found, tell the user to open nextdish.com in Chrome and try again.

Store the result as `WIN` and `TAB` (e.g. `WIN=1`, `TAB=9`).

## Step 2 — Get available dates

Navigate the tab to the menu page for the target date:

```applescript
tell application "Google Chrome"
  set URL of (tab TAB of window WIN) to "https://www.nextdish.com/menu?date=YYYY-MM-DD"
end tell
```

If the user didn't specify a date, first fetch the latest available date range:

```javascript
var req = new XMLHttpRequest();
req.open("GET", "/api/menu/latestDate?zipcode=95035", false);
req.withCredentials = true;
req.send();
req.responseText
```

Parse `startDate` and `endDate` and use `startDate` (or the next day) as default.

Convert user-provided dates like "Sunday" or "04/05" to `YYYY-MM-DD` format (year is current or next occurrence).

## Step 3 — Fetch the menu

After navigating to the menu page, wait ~3 seconds for it to load, then:

```javascript
var req = new XMLHttpRequest();
req.open("GET", "/api/menuInfo?date=YYYY-MM-DD&zipcode=95035", false);
req.withCredentials = true;
req.send();
req.responseText
```

Parse the JSON array. Each item has:
- `id` — dish ID
- `name` — dish name
- `price` — price
- `vegetarian` — boolean
- `spicyLevel` — 0–3
- `likePercentage` — null or 0–100
- `supplier.name` — restaurant name
- `topLeftTag` — `{text: "Best Seller"}` or null
- `ingredientTags` — array with tag names

## Step 4 — Fetch order history (when recommending)

If the user asked for recommendations or history-based suggestions, navigate to `/user-orders` and scrape the page:

```applescript
tell application "Google Chrome"
  set URL of (tab TAB of window WIN) to "https://www.nextdish.com/user-orders"
end tell
```

Wait 3 seconds, then read `document.body.innerText` to extract past orders. The page shows 10 orders per page — click page 2 if needed:

```javascript
var p2 = Array.from(document.querySelectorAll(".ant-pagination-item")).find(el => el.innerText.trim() === "2");
p2 && p2.click();
```

Parse the scraped text into a list of `{date, meal, dishName, price}` entries. Look for patterns:
- **Repeated dishes** → strong preference signal
- **Cuisine type** (mala/spicy, noodles, Japanese, congee, etc.)
- **Meal slot preferences** (lunch vs dinner)
- **Price range** (typical spend)
- **Avoid** dishes ordered very recently (last 3 days) to encourage variety

After gathering history, navigate back to the menu page before proceeding.

## Step 5 — Filter and select

Apply filters from the user's request:
- `vegetarian` / `veg` → `d.vegetarian === true`
- `max_price` → `d.price <= max_price`
- cuisine/restaurant keyword → match against `d.supplier.name` case-insensitively
- dish name keyword → match against `d.name` case-insensitively

From the filtered list, rank by:
1. **History match** (if history was fetched): boost dishes matching the user's cuisine preferences; slightly penalize dishes ordered in the last 3 days
2. `topLeftTag` present (Best Seller / Top Rated / Most Popular) → boost
3. `likePercentage` descending (nulls last)
4. Price ascending as tiebreaker

If `auto` mode: pick the top result silently.
Otherwise: show the top 3–5 matches in a table (price, restaurant, dish, rating, tags) with a one-line rationale for each (e.g. "matches your mala preference" or "you ordered this twice before"). Ask the user to confirm which one to order.

## Step 6 — Check for existing order

Before adding to cart, check if there's already an order for this date:

```javascript
var req = new XMLHttpRequest();
req.open("GET", "/api/order/daily?date=YYYY-MM-DD", false);
req.withCredentials = true;
req.send();
req.responseText
```

If the response is non-empty JSON with existing items, tell the user and ask if they want to add to it or cancel.

## Step 7 — Add dish to cart

The page must already be on `https://www.nextdish.com/menu?date=YYYY-MM-DD`.

Find the dish card by matching the dish name in the DOM, then click its add button:

```javascript
var allCards = document.querySelectorAll("[class*=singleDishCard]");
var target = Array.from(allCards).find(card => {
  var hasAddBtn = !!card.querySelector(".add-button-container-1tBhm_singleDishCard");
  var text = card.innerText || "";
  return hasAddBtn && text.includes("DISH_NAME_FRAGMENT");
});
if (target) {
  var btn = target.querySelector(".add-button-container-1tBhm_singleDishCard");
  var evt = new MouseEvent("click", {bubbles: true, cancelable: true, view: window});
  btn.dispatchEvent(evt);
  "clicked";
} else { "not found"; }
```

Wait 1–2 seconds, then verify the cart shows the expected price:

```javascript
(document.querySelector("[class*=checkout-button]") || {innerText: ""}).innerText
```

If the cart doesn't update, retry once with the SVG child element.

## Step 8 — Checkout

Click the checkout button (the actual `<button>` element, not the container div):

```javascript
var btn = document.querySelector(".filled-button-style-3uE-__filledButton.checkout-button-34A-N_cartRightSection");
btn ? btn.click() : "no btn";
```

Wait 3 seconds. The page navigates to `/group-checkout`. Read the summary:

```javascript
document.body.innerText.substring(0, 600)
```

Confirm the summary shows:
- Correct dish and price
- Business Credit covers it (if applicable)
- Order Total

Show the summary to the user and confirm before proceeding. Do not auto-proceed past this point.

## Step 9 — Place order

Click **Next Step** on `/group-checkout`:

```javascript
var btn = Array.from(document.querySelectorAll("button")).find(b => b.innerText.includes("Next Step"));
btn && btn.click();
```

Wait 3 seconds. Read the final `/group-place-order` page:

```javascript
document.body.innerText.substring(0, 600)
```

Show the user the final confirmation (address, payment, total) and ask for explicit approval.

On approval, click **Place Order**:

```javascript
var btn = Array.from(document.querySelectorAll("button")).find(b => b.innerText.includes("Place Order"));
btn && btn.click();
```

Wait 4 seconds. Read the success page and report:
- Order number
- Dish name and price
- Delivery date and time window
- Address
- Amount charged

## Error handling

- **Tab not found**: Ask user to open nextdish.com in Chrome and enable JavaScript from Apple Events (Chrome menu → View → Developer → Allow JavaScript from Apple Events).
- **Menu empty / no matching dishes**: Tell the user no dishes match the filter for that date, list what's available, ask them to adjust.
- **Cart didn't update**: Retry the click once; if still no update, tell user to add manually.
- **Checkout page shows wrong item**: Stop and tell the user.
- **Order already exists for that date**: Inform user, ask if they want to add another item.
- **Place Order button missing**: Stop and tell the user something unexpected happened on the confirmation page.
