# Custom Landing Pages

Every Yard product has a public landing page at `yard.sh/@{username}/{slug}` (also served over a seller's own hostname if they've set up a custom domain). There are two page types; this file is a complete reference for the second one.

| Page type | Who can use it | Where it's authored | Rendered how |
|---|---|---|---|
| **Built-in template** | All sellers (free + Pro) | Dashboard → product → **Listing** tab (title, tagline, cover image, description, FAQ, metadata) | Yard's own SolidJS component, consistent layout across the store |
| **Custom HTML** | **Pro only** | Dashboard → product → **Custom Page** tab, **or** the CLI (`yard page`) editing files under `.yard/landing-page/` | A bundle of HTML/CSS/JS served inside a sandboxed, full-viewport iframe |

If a product's `page_type` is `custom` and it has a published bundle, visitors see the custom page. Otherwise they see the built-in template. Toggling **Use custom page** off on the Custom Page tab keeps the draft/published bundle saved but renders the built-in template instead.

> An AI agent asked to "build a landing page for a Yard product" should default to Custom HTML only if the seller is on Pro. On the free plan, edit the Landing page-tab fields via the yard.sh/dashboard instead — there is no CLI for those.

---

## Architecture

A custom page is **not** a plain hosted HTML file. It's a bundle served inside an iframe on `yard.sh`, and Yard runs the auth + checkout flows in the parent frame. Your HTML talks to the parent via `window.postMessage`.

- The iframe has `sandbox="allow-scripts allow-forms allow-popups"` — **no `allow-same-origin`**. Scripts run in an opaque origin. You cannot read `yard.sh` cookies, touch the parent DOM, or call same-origin APIs as the signed-in user. You also cannot use your own `localStorage`/`sessionStorage` (the origin is opaque even to your own scripts). Keep state in-memory or delegate it to your own backend via `fetch`.
- Response headers include `Content-Security-Policy: sandbox allow-scripts allow-forms allow-popups` in addition to the iframe attribute.
- The iframe is positioned `fixed inset-0 w-screen h-screen` — there is no Yard chrome around it. Your page owns every pixel.

Every message in both directions carries `v: 1`. The parent ignores any message whose `v` is not `1`; this is reserved for future contract changes.

---

## The mandatory `yard:ready` handshake

This is the single most common way to ship a broken custom page.

The parent has no way to know when your page's `message` listener has actually been attached — `postMessage` is fire-and-forget, so the parent cannot send `yard:product` blindly. Instead:

1. Your page attaches a `message` listener.
2. Your page posts `{ type: 'yard:ready', v: 1 }` to `parent`.
3. The parent replies with `yard:product` (once).

**If `yard:ready` does not arrive within 10 seconds, the parent replaces the iframe with the built-in template.** Visitors will briefly see your HTML and then it flips to Yard's default layout. That is always a missing-handshake bug.

Attach the listener **before** you send ready:

```html
<script>
  window.addEventListener('message', (event) => {
    const msg = event.data;
    if (!msg || typeof msg !== 'object' || msg.v !== 1) return;
    if (msg.type === 'yard:product') {
      renderProduct(msg.product, msg.isAuthenticated, msg.user);
    }
    if (msg.type === 'yard:checkout:result') {
      // { status: 'success' | 'error', message?: string }
    }
  });
  parent.postMessage({ type: 'yard:ready', v: 1 }, '*');
</script>
```

---

## Messages the iframe sends (child → parent)

All four are validated with Zod in the parent; anything that fails validation is dropped silently with a `console.warn`.

### `yard:ready`
```ts
{ type: 'yard:ready', v: 1 }
```
No payload. Must be sent first. Triggers `yard:product` in response.

### `yard:checkout`
```ts
{
  type: 'yard:checkout',
  v: 1,
  tierId?: string,          // UUID; must match a real tier on the product
  quantity?: number,        // integer, 1..999; defaults to 1
  interval?: 'monthly' | 'yearly',  // subscription tiers only
  gift?: boolean,           // defaults to false
}
```
Validation rules enforced by the parent:
- `tierId`, if present, must match `/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i` **and** exist on `product.tiers`. Mismatched IDs produce a `yard:checkout:result` with `status: 'error'`.
- Omit `tierId` to use the product's default tier.
- `quantity` is capped at 999.
- `interval` is only meaningful for a subscription tier; ignored otherwise.

Flows the parent runs, in order:
1. Subscription tier → navigates the top window to `/checkout/{slug}?tier=...&interval=monthly|yearly[&quantity=N]`. Requires login; unauthenticated users are bounced to `/login?redirect=...` first.
2. Tier is free after any state discount (and not a gift) → the parent POSTs `/v1/claim` in place and replies with `yard:checkout:result`. Requires login.
3. Paid one-time or gift → navigates to `/checkout/{slug}[?gift=true][&tier=...][&quantity=N]`.

Only the free-claim branch ever completes without a navigation, so `yard:checkout:result` will only fire in that case.

### `yard:trial`
```ts
{ type: 'yard:trial', v: 1 }
```
No payload. The parent POSTs `/v1/start-trial` and navigates to `/trial-started/{transaction_id}` on success. There is no result message — paint trial UI only when `product.freeTrialEnabled === true`.

### `yard:navigate`
```ts
{ type: 'yard:navigate', v: 1, href: string }  // valid URL
```
The parent parses `href` against the current origin:
- Same-origin → `window.location.href = href` (replaces the current page).
- Cross-origin → `window.open(href, '_blank', 'noopener,noreferrer')`.
- Malformed URLs are dropped.

Use this for "view docs", "open repo", "see other product" links. Plain `<a href>` tags inside the iframe also work for cross-origin targets (target defaults to the iframe, so use `target="_top"` for same-origin or `target="_blank"` for external).

---

## Messages the iframe receives (parent → child)

### `yard:product`
Sent exactly once, immediately after the parent receives your `yard:ready`.

```ts
{
  type: 'yard:product',
  v: 1,
  product: PublicProduct,                       // shape below
  isAuthenticated: boolean,
  user: { username: string, avatar: string | null } | null,
}
```

### `yard:checkout:result`
Sent after a free-claim in-place checkout finishes. Not sent for paid/subscription checkout (the browser has already navigated away).

```ts
{
  type: 'yard:checkout:result',
  v: 1,
  status: 'success' | 'error',
  message?: string,          // only on error
}
```

---

## The `PublicProduct` shape (what your iframe receives)

The parent sends the **already-transformed, camelCase** TS object from `frontend/src/lib/public-product.ts`. These are the field names you use — not the snake_case the API returns.

```ts
interface PublicProduct {
  slug: string;
  title: string;
  description: string;             // plain Markdown source
  descriptionHtml: string;         // pre-rendered HTML — safe to inject into your page
  tagline: string;
  readmeHtml: string;              // README.md from the linked GitHub repo, pre-rendered
  priceCents: number;              // primary/default-tier price
  discountedPriceCents: number | null;
  state: 'draft' | 'preorder' | 'early_access' | 'released' | string;
  stateDiscountPercent: number | null;   // e.g. 20 means 20% off
  category: string;
  images: {
    iconLight: PublicProductImage | null;
    iconDark: PublicProductImage | null;
    gallery: PublicProductImage[];       // sorted by position
  };
  tiers: PricingTier[];
  releases: PublicRelease[];
  releaseCount: number;
  freeTrialEnabled: boolean;
  freeTrialDays: number;
  giftEnabled: boolean;
  licenseKeyEnabled: boolean;
  faq: { question: string; answer: string }[];
  metadata: { key: string; value: string }[];
  githubRepo: string | null;       // "owner/repo" or null
  pageType: 'builtin' | 'custom';
  customHtmlEtag: string | null;
  customDomain: { hostname: string; status: string } | null;
  seller: {
    username: string;
    avatarUrl: string | null;
    sellerReady: boolean;
  };
}

interface PublicProductImage {
  id: string;
  imageType: string;     // 'icon' | 'gallery'
  position: number;
  url: string;           // absolute URL — load directly
  mediaType: string;     // 'image' | 'video'
  videoUrl?: string;     // for 'video' gallery items
  videoProvider?: string;
  videoId?: string;
}

interface PublicRelease {
  tagName: string;
  releaseName: string;
  releaseNotes: string;
  releaseNotesHtml: string;
  publishedAt: string;   // ISO 8601
}

interface PricingTier {
  id: string;                          // UUID — pass this as tierId in yard:checkout
  productId: string;
  name: string;
  priceCents: number;                  // 0 or 300..1_000_000
  description: string;
  sortOrder: number;                   // render tiers in this order
  isDefault: boolean;                  // exactly one tier is the default
  seatType: 'single' | 'fixed_pack' | 'per_seat';
  seatCount: number | null;            // fixed_pack only (2..1000)
  minSeats: number | null;             // per_seat
  maxSeats: number | null;             // per_seat
  features: string[];                  // bullet list for the tier card
  volumeBrackets: {
    minQuantity: number;
    maxQuantity: number | null;        // null = open-ended
    discountPercent: number;
  }[];
  pricingModel: 'one_time' | 'subscription';
  yearlyDiscountPercent: number | null; // subscription only; 1..100
}
```

Rendering guidance:
- Display price by formatting `priceCents` (or `discountedPriceCents` if non-null) as dollars: `$${(cents/100).toFixed(2)}`.
- Sort tiers by `sortOrder` ascending.
- Respect `freeTrialEnabled` and `giftEnabled` — only surface those buttons when true.
- Auto-switch between `iconLight` and `iconDark` using `window.matchMedia('(prefers-color-scheme: dark)')`.
- `descriptionHtml` and `readmeHtml` are server-rendered from Markdown and safe to inject with `innerHTML`.
- For subscription tiers, apply `yearlyDiscountPercent` yourself when showing a monthly/yearly toggle — the server does not pre-compute the yearly price.
- For per-seat tiers with `volumeBrackets`, apply brackets yourself per-quantity (match by `minQuantity <= q <= maxQuantity`).

---

## Wiring checkout buttons

The pattern is: a button posts `yard:checkout` with the right `tierId` and optional extras. Let the parent handle everything after that.

```html
<!-- Default tier, qty 1 -->
<button id="buy">Buy now</button>

<!-- Specific tier, subscription, yearly -->
<button data-tier="11111111-2222-3333-4444-555555555555" data-interval="yearly">
  Pro yearly
</button>

<!-- Gift purchase -->
<button data-tier="..." data-gift="true">Buy as a gift</button>

<script>
  document.addEventListener('click', (e) => {
    const btn = e.target.closest('button[data-tier], #buy');
    if (!btn) return;
    parent.postMessage({
      type: 'yard:checkout',
      v: 1,
      tierId: btn.dataset.tier || undefined,
      quantity: Number(btn.dataset.quantity) || 1,
      interval: btn.dataset.interval || undefined,
      gift: btn.dataset.gift === 'true',
    }, '*');
  });
</script>
```

Because paid/subscription flows navigate the top window away, any "thank you" state must live on a separate page (Yard's post-checkout flow handles this). Only free-claim (price 0 after discount) stays in-frame — listen for `yard:checkout:result` to show a confirmation.

Trial button:

```html
<button id="trial">Start free trial</button>
<script>
  document.getElementById('trial').addEventListener('click', () => {
    parent.postMessage({ type: 'yard:trial', v: 1 }, '*');
  });
</script>
```

Only render this when `product.freeTrialEnabled === true`.

---

## Bundle rules

- **`index.html` is required** to publish.
- Up to **20 files** per bundle.
- Up to **1 MB** per file.
- Up to **5 MB** total per bundle (draft or published).
- At most **one subdirectory level**. `img/logo.png` is OK; `a/b/logo.png` is not.
- Allowed extensions: `.html .css .js .json .svg .png .jpg .jpeg .webp .gif .woff2`. Anything else is rejected on upload.
- Paths: letters/digits/`._-` only, first character alphanumeric.
- Bundle bytes count toward the same storage quota as release artifacts.

Server-side behavior when serving the bundle:

- A `<base href="…/custom-page/render/">` tag is injected into your `index.html` automatically so relative URLs resolve against the bundle — no build step required.
- **Sibling-file inlining**: before the iframe sees `index.html`, the server inlines `<script src="foo.js">` as `<script>…</script>`, `<link rel="stylesheet" href="foo.css">` as `<style>…</style>`, and `<img src="foo.png">` as a base64 data URL. This sidesteps cookie loss on the opaque-origin iframe and avoids a subresource round-trip. Applies only to references into the bundle — external URLs (`http://`, `https://`, `//`, `data:`, `blob:`, `mailto:`, `tel:`, `javascript:`, `#`, `?`) pass through untouched.
- Published bundles cache at the edge: `Cache-Control: public, max-age=300, s-maxage=3600`. Expect up to ~5 minutes for published changes to propagate to a given viewer.

---

## Minimal working `index.html`

A complete starting point that satisfies the handshake, renders the essentials, and wires a Buy button. ~60 lines, no dependencies.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Product</title>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 640px; margin: 4rem auto; padding: 0 1.5rem; }
    h1 { margin: 0 0 0.25rem; }
    .tagline { color: #555; margin: 0 0 2rem; }
    .price { font-size: 2rem; font-weight: 600; }
    button { font: inherit; padding: 0.75rem 1.25rem; border: 0; background: #111; color: #fff; cursor: pointer; }
    button:hover { background: #333; }
  </style>
</head>
<body>
  <img id="icon" alt="" width="64" height="64" style="display:none">
  <h1 id="title">Loading…</h1>
  <p id="tagline" class="tagline"></p>
  <div id="price" class="price"></div>
  <p><button id="buy">Buy now</button> <button id="trial" style="display:none">Start free trial</button></p>
  <div id="description"></div>

  <script>
    // 1. Attach listener BEFORE announcing ready.
    window.addEventListener('message', (event) => {
      const msg = event.data;
      if (!msg || typeof msg !== 'object' || msg.v !== 1) return;

      if (msg.type === 'yard:product') {
        const p = msg.product;
        document.title = p.title;
        document.getElementById('title').textContent = p.title;
        document.getElementById('tagline').textContent = p.tagline || '';
        document.getElementById('description').innerHTML = p.descriptionHtml || '';

        const cents = p.discountedPriceCents ?? p.priceCents;
        document.getElementById('price').textContent = '$' + (cents / 100).toFixed(2);

        const iconUrl = (matchMedia('(prefers-color-scheme: dark)').matches
          ? p.images.iconDark : p.images.iconLight)?.url;
        if (iconUrl) {
          const img = document.getElementById('icon');
          img.src = iconUrl;
          img.style.display = 'block';
        }

        if (p.freeTrialEnabled) document.getElementById('trial').style.display = '';
      }

      if (msg.type === 'yard:checkout:result') {
        if (msg.status === 'success') alert('Added to your library!');
        else alert('Checkout failed: ' + (msg.message || 'unknown error'));
      }
    });

    document.getElementById('buy').addEventListener('click', () => {
      parent.postMessage({ type: 'yard:checkout', v: 1 }, '*');
    });
    document.getElementById('trial').addEventListener('click', () => {
      parent.postMessage({ type: 'yard:trial', v: 1 }, '*');
    });

    // 2. Announce ready — parent replies with yard:product.
    parent.postMessage({ type: 'yard:ready', v: 1 }, '*');
  </script>
</body>
</html>
```

---

## Publishing from an agent

Once `.yard/landing-page/` has the files you want:

```sh
yard page status                    # diff local vs remote draft, no writes
yard page push                      # upload changed files to the draft
yard page push --publish            # upload + go live in one step
yard page push --publish --json     # machine-readable; prints preview_url + live_url
```

Non-Pro accounts see a friendly upgrade error on publish and keep the draft saved. For the full `yard page` command surface see [cli-commands.md](cli-commands.md).

---

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Page renders for ~10s, then Yard's built-in template replaces it | Missing or delayed `parent.postMessage({ type: 'yard:ready', v: 1 }, '*')` | Send `yard:ready` synchronously on script load; listener must be attached first |
| `yard:checkout` appears to do nothing | `tierId` is not a valid UUID, doesn't match `product.tiers[i].id`, or the message is missing `v: 1` | Use a tier ID from the `yard:product` payload; include `v: 1` on every message |
| `yard:checkout:result` never arrives | Expected — paid/subscription checkout navigates the top window away. Only free-claim flows reply in place | Handle the success page on Yard's side, not in-iframe |
| Relative asset URL 404s | File not in the bundle, or uses a second subdirectory level | Keep the bundle flat or one level deep; confirm with `yard page ls` |
| Third-party script won't load | Almost always the third party refusing iframe embedding (X-Frame-Options / frame-ancestors) — not Yard's CSP, which is `sandbox` | Use a script the provider marks iframe-safe, or host the integration on your own domain |
| `document.cookie` / `localStorage.setItem` throws or has no effect | Iframe origin is opaque by design — no `yard.sh` cookies, no own-origin storage | Keep state in-memory or send it to your own backend via `fetch` |
| `yard:navigate` drops a URL | `href` failed `new URL()` parsing | Pass an absolute URL, or a root-relative path — not a bundle-relative one |

---

## Contract version

Current: `v: 1`. Any field addition stays on `v: 1`; any breaking change bumps the version. The parent drops any message whose `v` is not the current one, so an older iframe talking to a newer parent (or vice versa) degrades to a no-op rather than misbehaving.

Source of truth if this doc ever drifts:
- Parent handler + Zod schemas: `frontend/src/components/product/CustomProductPage.tsx`
- Checkout flow: `frontend/src/components/product/checkoutBridge.ts`
- `PublicProduct` / `PricingTier` TS: `frontend/src/lib/public-product.ts`, `frontend/src/lib/pricing.ts`
- Bundle serving (`<base>`, inlining, CSP, cache): `platform/internal/handlers/products/custom_page_files.go`
