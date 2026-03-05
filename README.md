# StackShop - Bug Fix Report

## Getting Started

```bash
yarn install
yarn dev
```

## What I Found

I started by running the app and clicking through the main flows — browsing products, filtering by category, searching, and viewing product details. That surfaced most of the bugs pretty quickly. I then looked at the code to understand the root causes and checked for related issues (like the missing TypeScript fields).

Here's everything I identified and fixed:

---

### 1. Subcategory dropdown wasn't filtering by category

When you select a category like "Tablets", the subcategory dropdown should only show subcategories for that category. Instead, it was showing every subcategory across all categories.

The issue was in `app/page.tsx` — the fetch call to `/api/subcategories` wasn't passing the `category` query param, even though the API endpoint already supports it:

```js
// before
fetch(`/api/subcategories`)

// after
fetch(`/api/subcategories?category=${encodeURIComponent(selectedCategory)}`)
```

This was probably the most impactful bug since it made the subcategory filter completely useless.

---

### 2. Product detail page was passing data through the URL

This one was a bigger architectural issue. Each product card linked to the detail page like this:

```jsx
href={{
  pathname: "/product",
  query: { product: JSON.stringify(product) },
}}
```

So the entire product object was serialized into the URL as a query string. This breaks in a few ways:
- URLs can get really long and get truncated by browsers
- You can't bookmark or share product links reliably
- The data passed from the listing page was incomplete — `featureBullets` and `retailerSku` were missing from the listing page's TypeScript interface, so they'd show up as undefined on the detail page

The fix was to create a dynamic route at `app/product/[sku]/page.tsx` that reads the SKU from the URL and fetches the full product from `/api/products/[sku]`. That API endpoint already existed in the codebase but wasn't being used. I deleted the old `app/product/page.tsx` since it's no longer needed.

---

### 3. Subcategory wasn't resetting when you switch categories

Related to bug #1. If you select Category A → Subcategory X, then switch to Category B, the subcategory X filter was still active. Since it doesn't belong to Category B, you'd get zero results with no indication of why.

Fixed by resetting `selectedSubCategory` whenever `selectedCategory` changes, before fetching the new subcategories. I also added a `key` prop to the Radix Select components so the dropdown text properly resets to the placeholder when you click "Clear Filters" (without the key, Radix keeps showing the old value even after setting state to undefined).

---

### 4. Some product images were broken

About 21 products use images from `images-na.ssl-images-amazon.com`, but `next.config.ts` only had `m.media-amazon.com` whitelisted. Next.js blocks images from unregistered domains, so those products showed broken images.

Added the missing hostname to `remotePatterns` in `next.config.ts`.

---

### 5. No error handling on fetch calls

None of the `useEffect` fetch calls had `.catch()` handlers. If the API errored or the network dropped, the products page would just show "Loading products..." forever.

Added error handling to all fetch calls. For the products fetch specifically, I added an `error` state that displays a message instead of the infinite spinner.

---

### 6. Missing price display

Every product in `sample-products.json` has a `retailPrice` field, but:
- It wasn't in the `Product` TypeScript interface in `lib/products.ts`
- It wasn't declared in the frontend interfaces either
- No price was shown anywhere in the UI

For an eCommerce app this is a pretty big miss. I added `retailPrice: number` to the interfaces and display it on both the product cards and the detail page.

---

### 7. Search triggered an API call on every keystroke

Typing "headphones" would fire 10 separate API requests. Added a simple 300ms debounce using `useRef` and a separate `debouncedSearch` state. The product fetch now triggers off the debounced value instead of the raw input.

---

## Other small things

- **Page title** was still "Create Next App" — updated to "StackShop" in `app/layout.tsx`
- **`featureBullets` access** on the product page wasn't null-safe — added optional chaining so it doesn't crash if the field is missing

## Files changed

| File | What changed |
|------|-------------|
| `app/layout.tsx` | Fixed page title |
| `app/page.tsx` | Subcategory filtering, search debounce, error handling, product links, price display, clear filters fix |
| `app/product/page.tsx` | Deleted — replaced by dynamic route |
| `app/product/[sku]/page.tsx` | New file — fetches product by SKU from API |
| `next.config.ts` | Added missing image domain |
| `lib/products.ts` | Added `retailPrice` to Product interface |
