---
name: nextdish
description: This skill should be used when the user says "/nextdish", asks to "order from NextDish", "order food", "what should I order today", "recommend a dish", "show me the NextDish menu", or mentions ordering lunch or dinner via NextDish. Handles browsing the menu, filtering by diet/price/cuisine, fetching order history for personalized recommendations, and completing checkout through the Chrome browser session.
version: 1.0.0
---

# NextDish Order Skill

Order food from NextDish via the Chrome browser session.

## Arguments

Parse the user's request for:
- `date` — delivery date, e.g. "04/05", "Sunday", "tomorrow", or a range like "Mon–Fri". Default: next available date from API.
- `meal` — "lunch", "dinner", or "both". Default: "lunch".
- `filter` — e.g. "vegetarian", "veg", "no pork", a cuisine, a dish name, or a restaurant name.
- `min_price` / `max_price` — price range per item. Default: no limit.
- `auto` — if user says "just pick one", "surprise me", or "order for me without asking", auto-select top match without confirmation.
- `recommend` / `history` — if user says "recommend based on history" or "what should I order", fetch order history first and use it to rank suggestions.
- `bulk` — if dates span multiple days, collect all date+meal pairs upfront, pick dishes for all of them (presenting a full plan if not `auto`), then execute orders sequentially.

## AppleScript ↔ JavaScript data transfer

**Critical pattern**: AppleScript drops large JS return values as `missing value`. Always use `localStorage` as a bridge:

```applescript
-- Store in JS:
execute (tab TAB of window WIN) javascript "localStorage.setItem('_key', someValue); 'done'"
-- Read back separately:
execute (tab TAB of window WIN) javascript "localStorage.getItem('_key') || 'null'"
```

Never try to return large objects or long strings directly from `execute javascript`. Filter/summarize in JS first, then store the compact result.

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

Navigate the tab to the menu page for the target date. For bulk orders (multiple dates), navigate once per date — don't re-navigate to the same date for lunch and dinner:

```applescript
tell application "Google Chrome"
  set URL of (tab TAB of window WIN) to "https://www.nextdish.com/menu?date=YYYY-MM-DD"
end tell
```

## Step 3 — Fetch the menu

**Fetch via API first** (fast, works before page finishes loading). Store results in localStorage to avoid the AppleScript `missing value` issue with large responses:

```javascript
var req = new XMLHttpRequest();
req.open("GET", "/api/menuInfo?date=YYYY-MM-DD&zipcode=95035", false);
req.withCredentials = true;
req.send();
// Don't return req.responseText directly — AppleScript drops large values.
// Instead, filter inline and store the summary:
var data = JSON.parse(req.responseText);
var dishes = [];
data.forEach(function(s){
  (s.dishes||[]).forEach(function(d){
    dishes.push(d.supplier.name + "|" + d.name + "|" + d.price + "|" + d.vegetarian + "|" + d.likePercentage + "|" + (d.topLeftTag ? d.topLeftTag.text : ""));
  });
});
localStorage.setItem("_menu", dishes.join("\n"));
"count:" + dishes.length;
```

Then read back:
```javascript
localStorage.getItem("_menu")
```

Each line: `supplier|name|price|vegetarian|likePercentage|tag`

**Important**: The `menuInfo` API returns dishes for ALL meal slots. Lunch vs dinner is determined by which DOM tab the dish card appears under — not by the API. Use the DOM tab approach in Step 7 to determine which tab to use.

**Date availability**: If the menu page shows a "Sorry, no [lunch/dinner] menu is available" message despite the API returning data, that meal slot is not open for ordering — skip it. Check using:

```javascript
var body = document.body.innerText;
var noLunch = body.includes('no lunch menu') || (body.includes('Sorry') && body.includes('lunch'));
var noDinner = body.includes('no dinner menu') || (body.includes('Sorry') && body.includes('dinner'));
```

Note: some dates have dinner-only (e.g. Sundays). Always check both tabs before concluding a date is fully unavailable.

## Step 4 — Fetch order history (when recommending)

**Check Claude memory first**: A preference profile may already be saved (see `nextdish_preferences.md` in memory). If it exists and is recent, use it directly and skip the `/user-orders` scrape — this saves ~6s and a page navigation.

Only scrape `/user-orders` if: (a) no memory profile exists, (b) the user explicitly asks for fresh history, or (c) it has been more than a week since the profile was last updated.

If scraping is needed, navigate to `/user-orders`:

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

**Switching lunch/dinner tabs**: Click the appropriate tab before scrolling. The active tab has class `active-1qhgG_sectionSelector`; inactive has `item-2ABnb_sectionSelector`.

```javascript
// Switch to Lunch tab:
var tab = Array.from(document.querySelectorAll("*")).find(function(el){
  return el.innerText && el.innerText.trim() === "Lunch" && el.tagName !== "BODY" && el.tagName !== "HTML" && el.children.length === 0;
});
tab && tab.click();

// Switch to Dinner tab (replace "Lunch" with "Dinner"):
```

**Scroll to load all cards** (critical — cards are lazy-loaded):

```javascript
window.scrollTo(0, document.body.scrollHeight);
// wait 2–3 seconds, then scroll again:
window.scrollTo(0, document.body.scrollHeight);
```

Wait 2–3 seconds after the second scroll before searching.

**Find and click the add button** — use the inner `<button>` selector, NOT the container div:

```javascript
var allCards = document.querySelectorAll("[class*=single-dish-container]");
var target = null;
for (var i = 0; i < allCards.length; i++) {
  if ((allCards[i].innerText || "").includes("DISH_NAME_FRAGMENT")) {
    var btn = allCards[i].querySelector(".add-button-t9KIY_singleDishCard");
    if (btn) { target = btn; break; }
  }
}
if (target) {
  target.scrollIntoView({block: "center"});
  target.click();
  "clicked";
} else { "not found"; }
```

**Verify cart updated** — check the cart text includes the expected price:

```javascript
Array.from(document.querySelectorAll("[class*=cart]"))
  .map(function(e){ return e.innerText.replace(/\n/g, " ").trim(); })
  .filter(function(t){ return t.includes("$"); })
  .join(" | ")
```

If the wrong dish was added accidentally, remove it with:

```javascript
var card = Array.from(document.querySelectorAll("[class*=single-dish-container]"))
  .find(function(c){ return (c.innerText||"").includes("WRONG_DISH_NAME"); });
if (card) {
  var removeBtn = card.querySelector("[class*=remove-butt]");
  removeBtn && removeBtn.click();
}
```

**For same-date lunch + dinner orders**: Place lunch, complete checkout, then navigate back to `menu?date=YYYY-MM-DD`, switch to Dinner tab, scroll, and add the dinner dish. This avoids an extra page load.

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

Verify the summary shows the correct dish, price, and Order Total. Always show this to the user.

- **Non-auto mode**: ask the user to confirm before continuing.
- **Auto mode** (bulk/surprise-me): proceed immediately after showing the summary — no confirmation needed.

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

Show the final confirmation (address, payment, total).

- **Non-auto mode**: ask for explicit approval before clicking Place Order.
- **Auto mode**: proceed immediately.

Click **Place Order**:

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
- **Dish not found in DOM after scrolling**: The supplier may only appear on the other tab (lunch/dinner). Switch tabs and retry before giving up.
- **Cart didn't update after clicking**: The wrong button element may have been targeted. Ensure you're clicking `.add-button-t9KIY_singleDishCard` (the `<button>` inside the container), not `.add-button-container-1tBhm_singleDishCard` (the outer DIV).
- **localStorage returns `null` after storing**: The JS `localStorage.setItem()` call likely threw or the value was too large. Filter data in JS before storing — don't store raw API responses (can be 900KB+).
- **AppleScript returns `missing value`**: The JS returned a large or complex value. Use the localStorage bridge pattern instead of relying on the direct return value from `execute javascript`.
- **API shows dishes but page says "no menu available"**: The date is not open for ordering through the UI. Skip it.
- **Checkout page shows wrong item**: Remove it with the `[class*=remove-butt]` button, then add the correct dish.
- **Order already exists for that date/meal**: Inform user, ask if they want to add another item.
- **Place Order button missing**: Stop and tell the user something unexpected happened on the confirmation page.

## Performance tips for bulk orders

- **Fetch all menus via API before placing any orders**: Use `menuInfo` for each date upfront (inline filter to avoid large localStorage values). Build the full order plan, present it, then execute sequentially.
- **Parallel menu fetches**: For multiple dates, fire all `menuInfo` XHRs concurrently using async JS, then poll localStorage for results. Cuts a 5-date sequential fetch (~5s) down to ~1s:
  ```javascript
  ['2026-04-07','2026-04-08','2026-04-09'].forEach(function(date) {
    fetch('/api/menuInfo?date=' + date + '&zipcode=95035', {credentials:'include'})
      .then(function(r){ return r.json(); })
      .then(function(data){
        var dishes = [];
        data.forEach(function(s){ (s.dishes||[]).forEach(function(d){
          dishes.push(d.id + '|' + d.supplier.name + '|' + d.name + '|' + d.price + '|' + d.likePercentage + '|' + (d.topLeftTag?d.topLeftTag.text:''));
        }); });
        localStorage.setItem('_menu_' + date, dishes.join('\n'));
      });
  });
  ```
  Poll with `localStorage.getItem('_menu_DATE')` until all are non-null.
- **The cart has no API — DOM click is mandatory**: Clicking the add button fires zero network requests. The cart is pure React client-side state and only syncs to the server at checkout. There is no shortcut; you must navigate to the menu page and click the button in the DOM.
- **Use dish `id` from `menuInfo` for reliable DOM matching**: The API returns an `id` field (e.g. `"0845P5STS19BA"`) per dish. Store it and prefer matching by a data attribute or dish name fragment over fuzzy text search.
- **Group lunch + dinner for same date**: Navigate to the date once, place lunch order, navigate back to the same URL, switch tab, place dinner. Saves one page load per day.
- **One scroll is enough if the dish is near the top**: Only the second scroll is needed when the target dish is far down the page. For well-known suppliers that appear early (e.g. One Bowl Wonder), a single scroll often suffices.
- **Check existing orders first**: Call `/api/order/daily?date=YYYY-MM-DD` for all dates before starting. An empty response body (length 0) means no order exists. Skip dates that are already ordered.
