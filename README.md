
# 📊 Active Data Callout System

## Overview

The **Active Data Callout System** enhances product detail pages (PDPs) across our Salesforce Commerce Cloud (SFCC) sites by injecting live product engagement metrics directly into content assets.

This feature displays dynamic labels like "🔥 Trending" or "⭐ Popular" based on real-time product data such as views and orders. It helps customers make confident purchase decisions by showcasing product popularity.

### ✅ Key Features

- Backend-calculated thresholds (site-wide or overridden per asset)
- Reusable frontend templates via content assets
- Supports **views**, **orders**, and other metrics
- Animation effects (shine, flicker, bounce, shake)
- Fully configurable via data attributes
- Non-breaking: gracefully hides on missing or invalid data
- Supports A/B testing (e.g. `ActiveData` flag)

### 📌 Where It’s Used

- Product Detail Pages (PDPs)
- Content Assets in WYSIWYG
- A/B Test Variants

### 🔒 Permissions Required

| Role        | Access Needed                      |
|-------------|------------------------------------|
| Developer   | Code integration & cartridge edits |
| Marketing   | Content asset setup                |
| Admin       | Site Preferences in Business Manager |


## How It Works

The Active Data Callout System is composed of three main layers:

---

### 🧠 1. Backend Data Formatter (Controller + Helper)

On PDP load, the `Show` controller:
- Retrieves the product using the `pid` query string parameter
- Extracts the `activeData` object (if available)
- Passes it to the `getFormattedMetrics()` helper method

#### 🔁 Backend Helper Logic

- Formats numerical values with commas (e.g. `14,000`)
- Evaluates thresholds using global site preferences:
  - `viewsThresholdLow`
  - `viewsThresholdMedium`
  - `viewsThresholdHigh`
- Assigns a tier (`low`, `medium`, or `high`) per metric
- Returns a structured object to inject into the frontend

The resulting JSON is injected into the page:
```html
<div id="metrics-json" style="display: none;">
  ${pdict.metricsJSONString}
</div>
```

### 🎨 2. Frontend Renderer (JavaScript)

When the DOM is ready:
- Parses the JSON in `#metrics-json`
- Finds all `.social-proof` elements on the page
- Extracts attributes like `data-key`, `data-template`, `data-low`, `data-label`, etc.
- Replaces template placeholders with live metric data

#### Example Behavior

If the metric `viewsWeek = 500` and thresholds are:
- `low = 100`
- `medium = 300`
- `high = 600`

Then:
- The tier is determined as `medium`
- The label is resolved as `⭐ Popular`
- The final rendered output becomes:

```html
<span class="label-text medium">⭐ Popular</span>: 500 views
```
### ✏️ 3. Content Asset Markup

Each callout is defined directly inside a WYSIWYG content asset using the following structure:

```html
<div class="social-proof"
     data-key="weeklyViews"
     data-label="🔥 Trending Now"
     data-template="<span class='label-text ${tier}'>${label}</span>: ${value} views">
</div>
```

This setup gives content authors full control over how the callout appears. The supported attributes are:

| Attribute       | Description                                                               |
|----------------|---------------------------------------------------------------------------|
| `data-key`      | The metric to pull from backend data (e.g., `weeklyViews`, `weeklyOrders`) |
| `data-template` | The HTML template string with placeholders (`${value}`, `${tier}`…)         |
| `data-low`      | Optional: override low threshold for this asset only                      |
| `data-medium`   | Optional: override medium threshold for this asset only                   |
| `data-high`     | Optional: override high threshold for this asset only                     |
| `data-label`    | Optional: override the default label for this tier                        |
| `data-emoji`    | Optional: override the emoji for this tier                                |

[📷 Screenshot Placeholder: WYSIWYG content asset example with metric callouts]


## 🔧 Backend Setup

### ⚙️ 1. Controller & Helper Integration

The backend logic that powers the Active Data Callout system is located in two main places:

---

#### 📁 Controller: `Product-Show`

In the `server.append('Show')` route for PDP pages, the controller:

- Retrieves the product via `ProductMgr.getProduct(req.querystring.pid)`
- Checks if the product has `.activeData` attached
- Calls the `getFormattedMetrics(product)` helper
- Converts the result to a JSON string and sets it on `pdict.metricsJSONString`

```js
viewData.metricsJSONString = JSON.stringify(
    productHelpers.getFormattedMetrics(product)
);
```

This exposes the structured metrics object to the frontend via:

```html
<div id="metrics-json" style="display: none;">
  ${pdict.metricsJSONString}
</div>
```
---

#### 🧰 Helper: `getFormattedMetrics(product)`

This helper:
- Pulls raw product data from `product.activeData`
- Looks up threshold values from site preferences
- Applies tier logic (`low`, `medium`, `high`)
- Formats numbers for display (e.g., `14,000`)
- Returns a structured object like:

```json
{
  "weeklyViews": {
    "value": "14,000",
    "tier": "high",
    "thresholds": {
      "low": 11,
      "medium": 31,
      "high": 66
    }
  }
}
```

If no `activeData` is present, it returns an empty object.

### 🛠️ 2. Site Preferences Configuration

Thresholds for metrics (e.g., weekly views) are managed through custom **site preferences** in Business Manager. These allow centralized control over what qualifies as "low", "medium", or "high" across all content assets — unless overridden directly in the HTML.

---

#### 🎛️ Preference Keys Used

| Preference ID            | Purpose                        | Fallback Value |
|--------------------------|--------------------------------|----------------|
| `viewsThresholdLow`      | Lower bound for "low" tier     | `11`           |
| `viewsThresholdMedium`   | Lower bound for "medium" tier  | `31`           |
| `viewsThresholdHigh`     | Lower bound for "high" tier    | `66`           |

---

#### 📍 Where to Configure

1. Go to **Merchant Tools** → **Sites** → **[Your Site]** → **Preferences** → **Custom Preferences**
2. Locate or create a preference group (e.g., `ActiveDataConfig`)
3. Add or update the following:
   - `viewsThresholdLow`
   - `viewsThresholdMedium`
   - `viewsThresholdHigh`

---

#### 📌 Example Code Usage

In the helper function, these are retrieved via:

```js
const site = Site.getCurrent();
const globalThresholds = {
    low: site.getCustomPreferenceValue('viewsThresholdLow') || 11,
    medium: site.getCustomPreferenceValue('viewsThresholdMedium') || 31,
    high: site.getCustomPreferenceValue('viewsThresholdHigh') || 66
};
```

## 🎨 Frontend Behavior & Styling

The frontend script dynamically injects product metric labels into `.social-proof` blocks using a combination of:

- JSON data from the backend
- HTML templates in `data-template`
- Tier evaluation (`low`, `medium`, `high`, or `no-data`)
- Optional styling animations via CSS classes

---

### 🧩 Default Tier Definitions

When no custom label or emoji is provided, the system uses these built-in tier values:

| Tier       | Label       | Emoji | Notes                        |
|------------|-------------|-------|------------------------------|
| `high`     | Trending    | 🔥    | Highlights hot-selling items |
| `medium`   | Popular     | ⭐    | Solid-performing products     |
| `low`      | Low         | (none) | Not drawing much attention   |
| `no-data`  | No Data     | ⚠️    | Shown only if value is missing or invalid

These defaults can be overridden using `data-label` and `data-emoji` in the HTML.

---

### 🎭 Supported Animations

CSS classes can be included in the HTML templates to make the labels more attention-grabbing:

| Class Name      | Animation           | Description                                 |
|-----------------|---------------------|---------------------------------------------|
| `flame-flicker` | 🔥 Flicker          | Emulates fire movement                      |
| `shine`         | ✨ Shine (Text Glow) | Animated shimmer effect for text            |
| `bounce-slow`   | 🎯 Bounce           | Gently lifts and drops element              |
| `pulse`         | ⚡ Pulse             | Slight pulsing of scale and opacity         |
| `shake`         | 🚨 Shake            | Horizontal vibration for urgency or warning |

---

### 🧪 Fallback Logic

To avoid visual issues, the system automatically hides invalid or unsupported blocks:

- If `#metrics-json` is missing or malformed → No metrics rendered
- If `data-key` is not found in the JSON → Block is removed
- If the metric value is not a number → Block is removed
- If no thresholds are configured and no tier exists → Block is removed

```js
if (!key || !tpl || !metrics[key]) {
    el.remove();
    return;
}
```

## 🧱 Content Asset Usage Examples

These examples show how to use the `.social-proof` system within content assets by mixing global preferences with per-block overrides.

---

### 📌 Example 1: Full Threshold Override

This example uses custom `low`, `medium`, and `high` thresholds for the `weeklyViews` metric, overriding the default site preferences:

```html
<div class="social-proof"
     data-key="weeklyViews"
     data-low="100"
     data-medium="160"
     data-high="300"
     data-template="<div class='proof-label-box ${tier}'><span class='emoji-wrapper flame-flicker'>${emoji}</span> <span class='label-text ${tier}'>Custom Thresholds</span>: ${value} people saw this</div>">
</div>
```

**Thresholds:**

-   Low ≥ 100
    
-   Medium ≥ 160
    
-   High ≥ 300
    

Includes a custom visual template using `proof-label-box` and `flame-flicker`.

### 🔄 Example 2: Metric Swap (`weeklyOrders`)

This example uses a different backend metric — `weeklyOrders` — to display the number of recent purchases instead of product views.

```html
<div class="social-proof"
     data-key="weeklyOrders"
     data-template="<div class='proof-label ${tier}'><span class='emoji-wrapper'>${emoji}</span> <span class='label-text ${tier} shine'>${label}</span>: ${value} orders</div>">
</div>
```
**Behavior:**

-   Uses `weeklyOrders` instead of `weeklyViews`
    
-   Uses the default thresholds (if defined in backend) or falls back to low/medium/high logic
    
-   Applies the `shine` animation to the label for visual emphasis

### ⚠️ Example 3: Handling Missing or Invalid Metrics

If a metric is missing from the backend JSON or fails validation (e.g., it's not a number), the block will automatically be removed from the DOM to avoid displaying broken or misleading UI.

```html
<div class="social-proof"
     data-key="nonexistentMetric"
     data-template="<div class='proof-label ${tier}'><span class='emoji-wrapper shake'>${emoji}</span> <span class='label-text ${tier} shake'>MISSING</span>: ${value}</div>">
</div>
```
**Behavior:**

-   The system looks for `nonexistentMetric` in the JSON object
    
-   Since the key doesn't exist, the element is removed entirely
    
-   Optional: You can use `shake` to visually highlight problematic entries during QA
    

This fallback ensures content assets **fail silently and cleanly**, without confusing the shopper.

### 🖊️ Example 4: Manual Label and Emoji Overrides

This example shows how to override both the label and emoji using `data-label` and `data-emoji`, regardless of what tier the system determines.

```html
<div class="social-proof"
     data-key="weeklyViews"
     data-label="💣 Exploding Views"
     data-emoji="🔥"
     data-template="<div class='proof-label ${tier}'><span class='emoji-wrapper flame-flicker'>${emoji}</span> <span class='label-text ${tier} shine'>${label}</span> — ${value}</div>">
</div>
```

**Behavior:**

-   Replaces the tier-based label with a custom one: `💣 Exploding Views`
    
-   Forces the emoji to always be `🔥`, regardless of tier
    
-   Still uses the `tier` internally to apply styling and logic

## 🧪 Testing, Debugging & Fallback Behavior

The Active Data Callout system is designed to be fault-tolerant and easy to debug. Below are steps to verify functionality, isolate issues, and confirm expected behavior.

---

### ✅ How to Verify It’s Working

1. Inspect the page source on a PDP:
   - Confirm `#metrics-json` exists
   - Confirm it contains a valid JSON object with keys like `weeklyViews`, `weeklyOrders`, etc.

2. In the DOM, look for rendered `.social-proof` blocks:
   - Each block should contain styled HTML with text like `🔥 Trending: 1,500 views`

3. Check the frontend console (DevTools):
   - No errors from parsing or rendering
   - No warnings about missing `data-key`

---

### 🐞 Common Issues & Fixes

| Symptom                           | Likely Cause                                     | Fix                                                     |
|----------------------------------|--------------------------------------------------|----------------------------------------------------------|
| Callouts not showing             | `metrics-json` div is missing                    | Ensure `pdict.metricsJSONString` is set in controller   |
| Empty or default values shown    | `activeData` is missing on product               | Check if product has active metrics                     |
| Wrong tiers displaying           | Thresholds are misconfigured or not numeric      | Double-check site preference values or HTML attributes  |
| Callouts disappearing silently   | Invalid/missing `data-key`, or bad template      | Check that `metrics[key]` exists and templates are valid |

---

### 🧼 Fallback Logic Recap

The system **removes** a `.social-proof` block entirely if:

- `#metrics-json` is missing or malformed
- `data-key` does not exist in the JSON object
- Metric value is not a valid number
- No thresholds exist and no tier fallback is defined
- The tier is resolved to `no-data`

This ensures shoppers only see **meaningful** labels — never broken or misleading data.

## 🧪 A/B Testing Configuration (Optional)

The Active Data Callout system is optionally gated behind an A/B test flag to allow phased rollout and experimentation.

---

### 🔀 ISML Conditional

This block wraps the entire Active Data section to restrict it to test participants only:

```html
<isif condition="${dw.campaign.ABTestMgr.isParticipant('ActiveData', 'activeData')}">
    <div class="active-data">
        <div id="metrics-json" style="display: none;">
            ${pdict.metricsJSONString}
        </div>
        <div class="d-block d-md-none">
            <isslot id="active-data-slot" description="Product active data" context="global" context-object="${pdict.product.raw}" />
        </div>
    </div>
</isif>
```

### 🧪 Campaign Setup (in Business Manager)

1.  Go to **Merchant Tools** → **Online Marketing** → **A/B Tests**
    
2.  Create or locate a campaign named `ActiveData`
    
3.  Add a test variation also named `activeData`
    
4.  Assign participants, duration, and targeting rules as needed
    

----------

### 🧼 Fallback Behavior

If the current session is **not** part of the test:

-   The `metrics-json` block and `.social-proof` elements are never rendered
    
-   No JavaScript errors occur
    
-   The user sees the original content without Active Data enhancements

## ✅ Summary & Final Notes

The Active Data Callout system provides a flexible, scalable way to enhance product pages with real-time social proof. It gives developers, marketers, and decision-makers a powerful tool for improving engagement and conversions — without constant code changes.

---

### 🚀 Key Benefits

- Fully backend-driven: updates reflect live product activity
- Marketing-friendly: no deployment needed for visual edits
- Site-wide configurable via Business Manager
- Supports A/B testing for safe rollout and experimentation
- Graceful fallbacks ensure no broken content

---

### 🧰 What You Can Customize

| Layer              | What You Can Change                      |
|-------------------|-------------------------------------------|
| Backend Helper     | Add new metrics, adjust formatting logic |
| Site Preferences   | Define tier thresholds globally           |
| Content Assets     | Override labels, emojis, thresholds       |
| CSS Classes        | Extend with new animations or styles      |

---

### 📌 Next Steps (Optional)

- [ ] Add more metrics (e.g., impressions, days available)
- [ ] Enable desktop version of the callouts (currently mobile-only)
- [ ] Localize labels based on site locale
- [ ] Track clickthrough or visibility for analytics

---

### 🙋 FAQ

**Q:** What if the product has no active data?  
**A:** The callout will not display — this is intentional to avoid confusing users.

**Q:** Can I show different emojis for different tiers?  
**A:** Not natively, but you can simulate it by using template logic and `data-emoji`.

**Q:** Can this work in other parts of the site?  
**A:** Yes — as long as the controller exposes the correct JSON and `#metrics-json` is present.

---
