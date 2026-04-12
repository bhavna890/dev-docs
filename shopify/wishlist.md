# Wishlist Feature — Shopify Trade Theme

> **Theme:** Trade (Shopify)
> **Storage:** `localStorage` (client-side, no app required)
> **Status:** Production-ready, extensible

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [File Structure & Code Placement](#file-structure--code-placement)
4. [Step-by-Step Integration Guide](#step-by-step-integration-guide)
5. [Core Functionality](#core-functionality)
   - [Product Page Button](#product-button-code) (`custom liquid`)
   - [Header Icon & Badge](#header-icon-code) (`header.liquid`)
   - [Wishlist Page](#wishlist-page-mainpageliquid) (`main-page.liquid`)
   - [Product Card Button](#product-card-button-card-productliquid) (`card-product.liquid`)
6. [UI Behavior & Visual States](#ui-behavior--visual-states)
7. [Header Integration](#header-integration)
8. [Critical Logic Points](#critical-logic-points)
9. [Data Model — Extended](#data-model--extended)
10. [Future Extensibility](#future-extensibility)
11. [Best Practices](#best-practices)
12. [Testing Checklist](#testing-checklist)

---

## Overview

This feature adds a **persistent, client-side Wishlist** to a Shopify store running the **Trade theme**. It allows customers to save products they are interested in across sessions using the browser's `localStorage`.

### What it does

- Displays an **"Add to Wishlist"** button on each product page
- Shows a **heart icon** on every product card (collection/category pages) for quick wishlist access
- Toggles the button state between **active (wishlisted)** and **inactive (not wishlisted)**
- Persists wishlist data in `localStorage` so it survives page reloads and navigation
- Renders a full **Wishlist page** at `/pages/wishlist` with product cards, remove buttons, and "Move to Cart"
- Shows a **real-time count badge** on the header wishlist icon
- Links the header icon to a dedicated `/pages/wishlist` page

### Where it lives

| Component | File | Added via |
|---|---|---|
| Wishlist button (full) | Custom Liquid block | Shopify Theme Editor |
| Header icon + badge | `sections/header.liquid` | Code editor |
| Wishlist display page | `sections/main-page.liquid` | Code editor |
| Product card heart icon | `snippets/card-product.liquid` | Code editor |

---

## Architecture

The feature is split into three loosely coupled layers across four files:

```
┌──────────────────────────────────────────────────────────────────────┐
│                             UI Layer                                 │
│                                                                      │
│  [custom liquid]        [card-product.liquid]   [header.liquid]      │
│  Product page button  │ Card heart icon        │ Icon + count badge  │
│                                                                      │
│                    [main-page.liquid]                                │
│                    Wishlist page grid + cards                        │
└───────────────────────────────┬──────────────────────────────────────┘
                                │ reads / writes
┌───────────────────────────────▼──────────────────────────────────────┐
│                           Logic Layer                                │
│  getWL() · saveWL() · setActive() · toggleCardWishlist()            │
│  removeFromWishlist() · moveToCart() · updateHeaderBadge()          │
└───────────────────────────────┬──────────────────────────────────────┘
                                │ persists to
┌───────────────────────────────▼──────────────────────────────────────┐
│                          Storage Layer                               │
│            localStorage key: "shopify_wishlist"                     │
│            Value: JSON array of product objects                     │
└──────────────────────────────────────────────────────────────────────┘
```

### Communication Between Components

- Every add/remove fires both a native `storage` window event and a custom `wishlist-updated` DOM event.
- The header script and any other listener on the page responds to both events to stay in sync.
- The wishlist page (`main-page.liquid`) reads `localStorage` once on `DOMContentLoaded` and renders the full list.

### Data Model

Items stored from the **product page button** (Custom Liquid):

```json
{
  "id": "123456789",
  "title": "Product Name",
  "image": "//cdn.shopify.com/s/files/.../image_300x.jpg",
  "url": "/products/product-handle"
}
```

Items stored from the **product card button** (`card-product.liquid`) include additional fields:

```json
{
  "id": "123456789",
  "variantId": "987654321",
  "title": "Product Name",
  "image": "//cdn.shopify.com/s/files/.../image_300x.jpg",
  "url": "/products/product-handle",
  "price": "₹1,200.00"
}
```

> **Note:** The card button stores `variantId`, `price`, and optionally `sku` and `moq` — these are used by the wishlist page to display richer product info and enable "Move to Cart". Items saved from the product page button will not have these fields unless the button snippet is updated to match.

---

## File Structure & Code Placement

```
theme/
├── sections/
│   ├── header.liquid              ← Wishlist icon HTML, CSS, and JS (Step 2)
│   └── main-page.liquid           ← Wishlist page rendering logic (Step 3)
├── snippets/
│   └── card-product.liquid        ← Heart icon on product cards (Step 4)
└── [product page — Theme Editor]
    └── Custom Liquid block        ← Full wishlist button (Step 1)
```

### Placement Notes

| File | Where exactly to insert |
|---|---|
| `header.liquid` | Inside the existing header icon group, near the cart icon |
| `main-page.liquid` | At the top, wrapping the existing page content in the `{% if page.handle == 'wishlist' %}` conditional |
| `card-product.liquid` | Inside the card image wrapper `<div>`, after the product image tag |
| Custom Liquid block | Added via **Shopify Theme Editor** on the Product page template |

---

## Step-by-Step Integration Guide

### Prerequisites

- Access to **Shopify Admin** and the **Theme Editor**
- The **Trade theme** installed and active
- Basic familiarity with Liquid templating

---

### Step 1 — Add the Wishlist Button to the Product Page

1. Go to **Online Store → Themes → Customize**
2. Navigate to any **Product** page
3. In the left sidebar, find the product section and click **"Add block"**
4. Select **"Custom Liquid"**
5. Paste the full button snippet (see [Product Button Code](#product-button-code) below)
6. Save and preview

---

### Step 2 — Add the Wishlist Icon to the Header

1. In Shopify Admin, go to **Online Store → Themes → Actions → Edit Code**
2. Open `sections/header.liquid`
3. Locate the block of header icons (search for `header__icon` or the cart icon)
4. Paste the **header icon HTML** immediately before or after the cart icon
5. Paste the `<style>` and `<script>` blocks within the same file, before `</section>`
6. Save the file

---

### Step 3 — Set Up the Wishlist Page (`main-page.liquid`)

1. In **Edit Code**, open `sections/main-page.liquid`
2. At the very top (before the existing content), add the full wishlist conditional block (see [Wishlist Page Code](#wishlist-page-mainpageliquid) below)
3. The block uses `{% if page.handle == 'wishlist' %}` — it only activates for the wishlist page and falls through to the default page layout for all other pages
4. Go to **Online Store → Pages → Add page**, set the handle to `wishlist`
5. The page at `/pages/wishlist` will now render the wishlist UI

---

### Step 4 — Add the Heart Icon to Product Cards (`card-product.liquid`)

1. In **Edit Code**, open `snippets/card-product.liquid`
2. Locate the image wrapper `<div>` for the product card (search for `card__media` or the `<img>` tag)
3. Paste the wishlist button markup and `<style>` block inside the image wrapper
4. Paste the `<script>` block immediately after — the `if (typeof toggleCardWishlist === 'undefined')` guard ensures the function is only defined once even if the snippet renders multiple times
5. Save the file

---

## Core Functionality

### Product Button Code

```liquid
<div id="wl-wrap-{{ product.id }}" style="margin:10px 0;">
  <button
    type="button"
    id="wl-btn-{{ product.id }}"
    style="width:100%;padding:14px;border:2px solid #C8952E;background:#fff;color:#C8952E;font-size:15px;font-weight:600;cursor:pointer;display:flex;align-items:center;justify-content:center;gap:10px;border-radius:6px;transition:background 0.2s,border-color 0.2s,color 0.2s;"
    onmouseover="if(!this.dataset.active){this.style.borderColor='#E8A838';this.style.color='#E8A838';document.getElementById('wl-icon-{{ product.id }}').setAttribute('stroke','#E8A838');}"
    onmouseout="if(!this.dataset.active){this.style.borderColor='#C8952E';this.style.color='#C8952E';document.getElementById('wl-icon-{{ product.id }}').setAttribute('stroke','#C8952E');}"
  >
    <svg id="wl-icon-{{ product.id }}" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#C8952E" stroke-width="2">
      <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"/>
    </svg>
    <span id="wl-label-{{ product.id }}">Add to Wishlist</span>
  </button>
</div>

<script>
(function(){
  var pid = '{{ product.id }}';
  var btn  = document.getElementById('wl-btn-' + pid);
  var icon = document.getElementById('wl-icon-' + pid);
  var lbl  = document.getElementById('wl-label-' + pid);

  function getWL(){ return JSON.parse(localStorage.getItem('shopify_wishlist')||'[]'); }
  function saveWL(l){ localStorage.setItem('shopify_wishlist', JSON.stringify(l)); }

  function setActive(on){
    btn.dataset.active = on ? '1' : '';
    if(on){
      btn.style.background    = '#0B1D3A';
      btn.style.borderColor   = '#0B1D3A';
      btn.style.color         = '#ffffff';
      icon.setAttribute('fill',   '#ffffff');
      icon.setAttribute('stroke', '#ffffff');
      lbl.textContent = 'Wishlisted';
    } else {
      btn.style.background    = '#ffffff';
      btn.style.borderColor   = '#C8952E';
      btn.style.color         = '#C8952E';
      icon.setAttribute('fill',   'none');
      icon.setAttribute('stroke', '#C8952E');
      lbl.textContent = 'Add to Wishlist';
    }
  }

  var wl = getWL();
  setActive(wl.some(function(i){ return i.id === pid; }));

  btn.addEventListener('click', function(){
    var list = getWL();
    var idx  = list.findIndex(function(i){ return i.id === pid; });
    if(idx === -1){
      list.push({
        id:    pid,
        title: {{ product.title | json }},
        image: '{{ product.featured_image | image_url: width: 300 }}',
        url:   '{{ product.url }}'
      });
      setActive(true);
    } else {
      list.splice(idx, 1);
      setActive(false);
    }
    saveWL(list);
    window.dispatchEvent(new Event('storage'));
    document.dispatchEvent(new CustomEvent('wishlist-updated'));
  });
})();
</script>
```

---

### Header Icon Code

Place inside `sections/header.liquid`, within the existing icon group:

```liquid
{% comment %} Wishlist icon — header {% endcomment %}
<a href="/pages/wishlist" class="header__icon header-wishlist-icon" aria-label="Wishlist">
  <svg id="header-wishlist-svg" xmlns="http://www.w3.org/2000/svg" width="20" height="20"
    viewBox="0 0 24 24" fill="none" stroke="currentColor"
    stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
    <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"/>
  </svg>
  <span
    id="wishlist-count-badge"
    style="display:none;position:absolute;bottom:2px;right:-7px;background:#fff;color:#0a0a0a;font-size:10px;width:16px;height:16px;border-radius:50%;align-items:center;justify-content:center;line-height:16px;text-align:center;font-weight:600;">
  </span>
</a>

<style>
  .header-wishlist-icon {
    position: relative;
    display: inline-flex;
    align-items: center;
    padding-left: 20px;
    justify-content: center;
    width: 40px;
    height: 40px;
    color: currentColor;
    text-decoration: none;
  }
  .header-wishlist-icon:hover svg {
    stroke: #fff;
    transition: stroke 0.2s;
  }
  .header-wishlist-icon.has-items svg {
    fill: none;
    stroke: #fff;
  }
</style>

<script>
(function() {
  function updateHeaderWishlist() {
    try {
      var list  = JSON.parse(localStorage.getItem('shopify_wishlist') || '[]');
      var badge = document.getElementById('wishlist-count-badge');
      var svg   = document.getElementById('header-wishlist-svg');
      var link  = svg ? svg.closest('a') : null;
      if (!badge) return;

      if (list.length > 0) {
        badge.textContent   = list.length;
        badge.style.display = 'flex';
        if (link) link.classList.add('has-items');
      } else {
        badge.style.display = 'none';
        if (link) link.classList.remove('has-items');
      }
    } catch(e) {}
  }

  document.addEventListener('DOMContentLoaded', updateHeaderWishlist);
  window.addEventListener('storage', updateHeaderWishlist);
  document.addEventListener('wishlist-updated', updateHeaderWishlist);
})();
</script>
```

---

## Wishlist Page (`main-page.liquid`)

This section is conditionally injected into `sections/main-page.liquid`. It only activates when the current page handle is `wishlist`, leaving all other pages completely unaffected.

### Conditional Guard

```liquid
{%- if page.title == 'Wishlist' or page.handle == 'wishlist' or request.path contains 'wishlist' -%}
  {# Wishlist page UI #}
{%- else -%}
  {# Default page content — unchanged #}
{%- endif -%}
```

The triple condition (`title`, `handle`, `request.path`) ensures the wishlist UI renders correctly regardless of how the page was configured in the Shopify admin.

### Brand Design Tokens

All colours and fonts are defined as CSS custom properties on `:root`, making theme-wide changes a single-line edit:

```css
:root {
  --goel-navy:     #0B1D3A;   /* Primary dark — buttons, titles */
  --goel-gold:     #C8952E;   /* Primary accent — prices, CTAs */
  --goel-amber:    #E8A838;   /* Hover state for gold elements */
  --goel-slate:    #3D4F5F;   /* Secondary text */
  --goel-charcoal: #1E2A38;   /* Card titles */
  --goel-bg:       #F2F4F6;   /* Page background */
  --goel-card-bg:  #FAFBFC;   /* Card background */
  --goel-red:      #D64545;   /* Remove / error actions */
  --goel-green:    #2D8A4E;   /* Success states */
  --font-heading:  'Montserrat', sans-serif;
  --font-body:     'Open Sans', sans-serif;
}
```

### Full `main-page.liquid` Wishlist Block

```liquid
{{ 'section-main-page.css' | asset_url | stylesheet_tag }}

{%- if page.title == 'Wishlist' or page.handle == 'wishlist' or request.path contains 'wishlist' -%}

<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@500;600;700;800&family=Open+Sans:wght@400;600&display=swap" rel="stylesheet">

<style>
  /* ─── GOEL A TO Z — Brand Variables ─── */
  :root {
    --goel-navy:       #0B1D3A;
    --goel-gold:       #C8952E;
    --goel-amber:      #E8A838;
    --goel-slate:      #3D4F5F;
    --goel-charcoal:   #1E2A38;
    --goel-bg:         #F2F4F6;
    --goel-card-bg:    #FAFBFC;
    --goel-cream:      #FFF8ED;
    --goel-teal:       #2A9D8F;
    --goel-red:        #D64545;
    --goel-green:      #2D8A4E;
    --font-heading:    'Montserrat', var(--font-heading--family, sans-serif);
    --font-body:       'Open Sans', var(--font-body--family, sans-serif);
  }

  /* ─── PAGE WRAPPER ─── */
  .wl-wrap { padding: 48px 20px 80px; background: var(--goel-bg); min-height: 60vh; }

  /* ─── PAGE HEADER ─── */
  .wl-header-row { display: flex; align-items: flex-end; justify-content: space-between; flex-wrap: wrap; gap: 16px; margin-bottom: 32px; }
  .wl-title { font-family: var(--font-heading); font-size: 28px; font-weight: 700; color: var(--goel-navy); margin: 0; letter-spacing: 0.5px; }
  .wl-subtitle { font-family: var(--font-body); font-size: 14px; color: var(--goel-slate); margin: 4px 0 0; }
  .wl-header-actions { display: flex; gap: 12px; align-items: center; }

  .btn-primary { background: var(--goel-gold); color: #fff; font-family: var(--font-heading); font-size: 13px; font-weight: 600; letter-spacing: 1px; text-transform: uppercase; border: none; border-radius: 6px; padding: 10px 24px; cursor: pointer; transition: background 0.2s ease, transform 0.1s ease; text-decoration: none; display: inline-block; text-align: center; min-width: 160px; }
  .btn-primary:hover { background: var(--goel-amber); color: #fff; }
  .btn-primary:active { transform: scale(0.98); }

  .btn-outline { background: transparent; color: var(--goel-navy); font-family: var(--font-heading); font-size: 13px; font-weight: 600; letter-spacing: 1px; text-transform: uppercase; border: 1px solid var(--goel-navy); border-radius: 6px; padding: 10px 20px; cursor: pointer; transition: background 0.2s ease; text-decoration: none; display: inline-block; text-align: center; }
  .btn-outline:hover { background: var(--goel-navy); color: #fff; }

  /* ─── EMPTY STATE ─── */
  .wl-empty { background: var(--goel-card-bg); border: 1px solid #E4E8EC; border-radius: 8px; padding: 64px 24px; text-align: center; }
  .wl-empty-icon { width: 56px; height: 56px; margin: 0 auto 20px; opacity: 0.2; color: var(--goel-navy); }
  .wl-empty-title { font-family: var(--font-heading); font-size: 18px; font-weight: 700; color: var(--goel-navy); margin: 0 0 8px; }
  .wl-empty-msg { font-family: var(--font-body); font-size: 14px; color: var(--goel-slate); margin: 0 0 24px; }
  .wl-empty-msg a { color: var(--goel-gold); font-weight: 600; text-decoration: none; }
  .wl-empty-msg a:hover { text-decoration: underline; }

  /* ─── PRODUCT GRID ─── */
  .wl-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 24px; }

  /* ─── PRODUCT CARD ─── */
  .wl-card { background: var(--goel-card-bg); border: 1px solid #E4E8EC; border-radius: 8px; overflow: hidden; position: relative; transition: box-shadow 0.25s ease-in-out, transform 0.25s ease-in-out; }
  .wl-card:hover { box-shadow: 0 8px 24px rgba(11, 29, 58, 0.12); transform: translateY(-4px); }
  .wl-card-img-wrap { position: relative; overflow: hidden; background: #fff; aspect-ratio: 1; }
  .wl-card a { text-decoration: none; color: inherit; display: block; }
  .wl-card-img-wrap img { width: 100%; height: 100%; object-fit: cover; display: block; transition: transform 0.25s ease-in-out; }
  .wl-card:hover .wl-card-img-wrap img { transform: scale(1.05); }

  /* ─── REMOVE BUTTON ─── */
  .wl-card-remove { position: absolute; top: 10px; right: 10px; width: 32px; height: 32px; border-radius: 50%; background: #fff; border: 1px solid #E4E8EC; display: flex; align-items: center; justify-content: center; cursor: pointer; transition: border-color 0.2s, background 0.2s; z-index: 2; padding: 0; }
  .wl-card-remove:hover { border-color: var(--goel-red); background: #FFF0F0; }
  .wl-card-remove svg { width: 15px; height: 15px; stroke: var(--goel-slate); fill: none; stroke-width: 2.5; transition: stroke 0.2s; }
  .wl-card-remove:hover svg { stroke: var(--goel-red); }

  /* ─── CARD BODY ─── */
  .wl-card-body { padding: 16px 16px 20px; }
  .wl-card-title { font-family: var(--font-heading); font-weight: 600; font-size: 14px; color: var(--goel-charcoal); margin: 0 0 4px; display: -webkit-box; -webkit-line-clamp: 3; -webkit-box-orient: vertical; overflow: hidden; min-height: 40px; line-height: 1.4; }
  .wl-card-sku { font-family: var(--font-body); font-size: 12px; color: var(--goel-slate); margin: 0 0 10px; }
  .wl-card-price { font-family: var(--font-heading); font-size: 20px; font-weight: 700; color: var(--goel-gold); margin: 0 0 4px; }
  .wl-card-price span { font-size: 12px; font-weight: 400; color: var(--goel-slate); }
  .wl-card-moq { font-family: var(--font-body); font-size: 12px; font-weight: 600; color: var(--goel-navy); margin: 0 0 14px; }

  /* ─── CARD ACTIONS ─── */
  .wl-card-actions { display: flex; flex-direction: column; gap: 8px; }
  .wl-card-view { background: transparent; color: var(--goel-navy); border: 1px solid var(--goel-navy); border-radius: 6px; padding: 9px 6px; cursor: pointer; width: 100%; font-family: var(--font-heading); font-size: 12px; font-weight: 600; letter-spacing: 1px; text-transform: uppercase; text-align: center; text-decoration: none; display: block; box-sizing: border-box; transition: background 0.2s, color 0.2s; }
  .wl-card-view:hover { background: var(--goel-navy); color: #fff; }
  .wl-card-move { background: var(--goel-gold); color: #fff; border: none; border-radius: 6px; padding: 10px 6px; cursor: pointer; width: 100%; font-family: var(--font-heading); font-size: 12px; font-weight: 600; letter-spacing: 1px; text-transform: uppercase; box-sizing: border-box; transition: background 0.2s ease, transform 0.1s ease; }
  .wl-card-move:hover { background: var(--goel-amber); }
  .wl-card-move:active { transform: scale(0.98); }
  .wl-card-move:disabled { background: #D0D5DB; color: #8A95A3; cursor: not-allowed; opacity: 0.6; }

  /* ─── TOAST ─── */
  .wl-toast { position: fixed; bottom: 24px; left: 50%; transform: translateX(-50%) translateY(80px); background: var(--goel-navy); color: #fff; font-family: var(--font-body); font-size: 14px; padding: 12px 24px; border-radius: 6px; z-index: 9999; transition: transform 0.3s ease; white-space: nowrap; pointer-events: none; }
  .wl-toast.show { transform: translateX(-50%) translateY(0); }

  /* ─── RESPONSIVE ─── */
  @media screen and (max-width: 500px) {
    .wl-wrap { padding: 32px 20px 64px; }
    .wl-title { font-size: 22px; }
    .wl-grid { grid-template-columns: repeat(2, 1fr); gap: 12px; }
    .wl-card-body { padding: 12px 12px 16px; }
    .wl-card-price { font-size: 17px; }
    .wl-header-row { flex-direction: column; align-items: flex-start; }
    .wl-header-actions { width: 100%; }
    .wl-header-actions .btn-primary { width: 100%; }
  }
  @media screen and (min-width: 600px) and (max-width: 1023px) {
    .wl-grid { grid-template-columns: repeat(3, 1fr); gap: 16px; }
  }
  @media screen and (min-width: 1024px) {
    .wl-grid { grid-template-columns: repeat(4, 1fr); }
  }
</style>

<div class="wl-wrap page-width">
  <div class="wl-header-row">
    <div>
      <h1 class="wl-title">My Wishlist</h1>
      <p class="wl-subtitle" id="wl-count-label">Loading saved items…</p>
    </div>
    <div class="wl-header-actions" id="wl-bulk-actions" style="display:none;"></div>
  </div>

  <div id="wl-empty" style="display:none;" class="wl-empty">
    <svg class="wl-empty-icon" viewBox="0 0 56 56" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
      <path d="M48.65 11.07a13 13 0 0 0-18.38 0L28 13.37l-2.27-2.3a13 13 0 0 0-18.38 18.38l2.27 2.27L28 50.3l18.38-18.58 2.27-2.27a13 13 0 0 0 0-18.38z"/>
    </svg>
    <p class="wl-empty-title">Your wishlist is empty</p>
    <p class="wl-empty-msg">
      Save products you want to order later by clicking the heart icon on any product.<br>
      <a href="/collections/all">Browse all products &rarr;</a>
    </p>
    <a href="/collections/all" class="btn-primary" style="display:inline-block;">Shop Now</a>
  </div>

  <div id="wl-grid" class="wl-grid"></div>
</div>

<div class="wl-toast" id="wl-toast"></div>

<script>
document.addEventListener('DOMContentLoaded', function () {
  var wl      = JSON.parse(localStorage.getItem('shopify_wishlist') || '[]');
  var grid    = document.getElementById('wl-grid');
  var empty   = document.getElementById('wl-empty');
  var label   = document.getElementById('wl-count-label');
  var bulkBar = document.getElementById('wl-bulk-actions');

  if (!grid) return;

  function pluralise(n) {
    return n === 1 ? '1 item saved' : n + ' items saved';
  }

  function refreshHeader() {
    var current = JSON.parse(localStorage.getItem('shopify_wishlist') || '[]');
    if (label) label.textContent = pluralise(current.length);
    if (bulkBar) bulkBar.style.display = current.length > 0 ? 'flex' : 'none';
    if (current.length === 0 && empty) empty.style.display = 'block';
  }

  function showToast(msg) {
    var t = document.getElementById('wl-toast');
    if (!t) return;
    t.textContent = msg;
    t.classList.add('show');
    setTimeout(function () { t.classList.remove('show'); }, 3000);
  }

  function removeFromWishlist(id, cardEl) {
    var current = JSON.parse(localStorage.getItem('shopify_wishlist') || '[]');
    var updated = current.filter(function (i) { return String(i.id) !== String(id); });
    localStorage.setItem('shopify_wishlist', JSON.stringify(updated));
    if (cardEl) cardEl.remove();
    refreshHeader();
    showToast('Item removed from wishlist');
  }

  if (wl.length === 0) {
    if (empty) empty.style.display = 'block';
    if (label) label.textContent = '0 items saved';
    return;
  }

  label.textContent = pluralise(wl.length);
  bulkBar.style.display = 'flex';

  wl.forEach(function (item) {
    var card = document.createElement('div');
    card.className = 'wl-card';
    card.dataset.id = String(item.id);

    var itemUrl       = item.url       || '#';
    var itemImage     = item.image     || '';
    var itemTitle     = item.title     || 'Product';
    var itemId        = item.id;
    var itemVariantId = item.variantId || item.variant_id || itemId;
    var itemPrice     = item.price     ? '&#8377;' + item.price + '<span>/unit</span>' : '';
    var itemMOQ       = item.moq       ? 'MOQ: ' + item.moq + ' units' : '';
    var itemSku       = item.sku       ? 'SKU: ' + item.sku : '';

    var imgHtml = itemImage
      ? '<img src="' + itemImage + '" alt="' + itemTitle.replace(/"/g, '&quot;') + '" loading="lazy">'
      : '<div style="width:100%;height:100%;background:#F2F4F6;display:flex;align-items:center;justify-content:center;font-size:40px;opacity:0.25;">&#128230;</div>';

    card.innerHTML =
      '<div class="wl-card-img-wrap">' +
        '<a href="' + itemUrl + '">' + imgHtml + '</a>' +
        '<button class="wl-card-remove" title="Remove from wishlist" onclick="handleRemove(\'' + itemId + '\', this)">' +
          '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">' +
            '<line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/>' +
          '</svg>' +
        '</button>' +
      '</div>' +
      '<div class="wl-card-body">' +
        '<p class="wl-card-title">' + itemTitle + '</p>' +
        (itemSku   ? '<p class="wl-card-sku">'   + itemSku   + '</p>' : '') +
        (itemPrice ? '<p class="wl-card-price">'  + itemPrice + '</p>' : '') +
        (itemMOQ   ? '<p class="wl-card-moq">'   + itemMOQ   + '</p>' : '') +
        '<div class="wl-card-actions">' +
          '<a href="' + itemUrl + '" class="wl-card-view">View Product</a>' +
          '<button class="wl-card-move" onclick="moveToCart(\'' + itemId + '\', \'' + itemVariantId + '\', this)">Move to Cart</button>' +
        '</div>' +
      '</div>';

    grid.appendChild(card);
  });

  window.handleRemove = function (id, btn) {
    var card = btn.closest('.wl-card');
    removeFromWishlist(id, card);
  };

  window.moveToCart = function (id, variantId, btn) {
    if (!variantId || variantId === 'undefined') variantId = id;
    if (btn) { btn.disabled = true; btn.textContent = 'Adding…'; }

    fetch('/cart/add.js', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ id: Number(variantId), quantity: 1 })
    })
    .then(function (res) { return res.json(); })
    .then(function (data) {
      if (data.id) {
        var current = JSON.parse(localStorage.getItem('shopify_wishlist') || '[]');
        var updated = current.filter(function (i) { return String(i.id) !== String(id); });
        localStorage.setItem('shopify_wishlist', JSON.stringify(updated));
        showToast('Added to cart! Redirecting…');
        setTimeout(function () { window.location.href = '/cart'; }, 800);
      } else {
        showToast('Could not add to cart: ' + (data.description || 'Please try again'));
        if (btn) { btn.disabled = false; btn.textContent = 'Move to Cart'; }
      }
    })
    .catch(function (err) {
      showToast('Error: ' + (err.message || 'Please try again'));
      if (btn) { btn.disabled = false; btn.textContent = 'Move to Cart'; }
    });
  };
});
</script>

{%- else -%}

{%- style -%}
  .section-{{ section.id }}-padding {
    padding-top: {{ section.settings.padding_top | times: 0.75 | round: 0 }}px;
    padding-bottom: {{ section.settings.padding_bottom | times: 0.75 | round: 0 }}px;
  }
  @media screen and (min-width: 750px) {
    .section-{{ section.id }}-padding {
      padding-top: {{ section.settings.padding_top }}px;
      padding-bottom: {{ section.settings.padding_bottom }}px;
    }
  }
{%- endstyle -%}

<div class="page-width page-width--narrow section-{{ section.id }}-padding">
  <h1 class="main-page-title page-title h0{% if settings.animations_reveal_on_scroll %} scroll-trigger animate--fade-in{% endif %}">
    {{ page.title | escape }}
  </h1>
  <div class="rte{% if settings.animations_reveal_on_scroll %} scroll-trigger animate--slide-in{% endif %}">
    {{ page.content }}
  </div>
</div>

{% content_for 'blocks' %}

{%- endif -%}

{% schema %}
{
  "name": "t:sections.main-page.name",
  "tag": "section",
  "class": "section",
  "settings": [
    {
      "type": "header",
      "content": "t:sections.all.padding.section_padding_heading"
    },
    {
      "type": "range",
      "id": "padding_top",
      "min": 0,
      "max": 100,
      "step": 4,
      "unit": "px",
      "label": "t:sections.all.padding.padding_top",
      "default": 36
    },
    {
      "type": "range",
      "id": "padding_bottom",
      "min": 0,
      "max": 100,
      "step": 4,
      "unit": "px",
      "label": "t:sections.all.padding.padding_bottom",
      "default": 36
    }
  ]
}
{% endschema %}
```

### Key Functions — Wishlist Page

| Function | Purpose |
|---|---|
| `pluralise(n)` | Returns `"1 item saved"` or `"N items saved"` for the subtitle |
| `refreshHeader()` | Re-reads `localStorage` and updates the count label and bulk action bar visibility |
| `showToast(msg)` | Displays a fixed bottom toast notification for 3 seconds |
| `removeFromWishlist(id, cardEl)` | Filters the item out of `localStorage`, removes the card from the DOM, calls `refreshHeader()` |
| `moveToCart(id, variantId, btn)` | POSTs to Shopify's `/cart/add.js`, removes item from wishlist on success, redirects to `/cart` |

---

## Product Card Button (`card-product.liquid`)

This snippet adds a **floating heart icon** to every product card image. It is positioned absolutely over the top-right corner of the card image using CSS, and is separate from the full product page button.

### How It Differs from the Product Page Button

| Aspect | Product Page Button | Card Heart Icon |
|---|---|---|
| Location | Full-width button below ATC | Small icon overlay on card image |
| State indicator | Background color + label text | SVG `fill` via `.is-wished` CSS class |
| Data stored | `id`, `title`, `image`, `url` | `id`, `variantId`, `title`, `image`, `url`, `price` |
| Function | `btn.addEventListener('click')` | `toggleCardWishlist(btn)` global function |

### Full `card-product.liquid` Snippet

```liquid
<style>
  .card__wishlist-btn {
    position: absolute;
    top: 20px;
    right: 20px;
    z-index: 10;
    width: 24px;
    height: 24px;
    background: rgba(255, 255, 255, 0.85);
    border-radius: 50%;
    border: 1px solid rgba(0, 0, 0, 0.08);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    outline: none;
    transition: transform 0.2s ease;
  }
  .card__wishlist-btn:hover { transform: scale(1.2); }
  .card__wishlist-btn svg {
    width: 18px;
    height: 18px;
    stroke: rgb(0, 0, 0);
    fill: none;
    stroke-width: 1.8;
    transition: fill 0.2s ease;
    pointer-events: none;
    filter: drop-shadow(0 1px 2px rgba(0,0,0,0.18));
  }
  .card__wishlist-btn:hover svg,
  .card__wishlist-btn.is-wished svg {
    fill: #E8A838;
  }
</style>

{%- comment -%} Wishlist toggle button — positioned over the product card image {%- endcomment -%}
<button
  class="card__wishlist-btn"
  type="button"
  aria-label="Add to wishlist"
  data-product-id="{{ card_product.id }}"
  data-variant-id="{{ card_product.selected_or_first_available_variant.id }}"
  data-product-title="{{ card_product.title | escape }}"
  data-product-url="{{ card_product.url }}"
  data-product-image="{{ card_product.featured_media | image_url: width: 300 }}"
  data-product-price="{{ card_product.price | money }}"
  onclick="toggleCardWishlist(this)"
>
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" aria-hidden="true">
    <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"/>
  </svg>
</button>

<script>
if (typeof toggleCardWishlist === 'undefined') {
  function toggleCardWishlist(btn) {
    var pid       = btn.getAttribute('data-product-id');
    var title     = btn.getAttribute('data-product-title');
    var image     = btn.getAttribute('data-product-image');
    var url       = btn.getAttribute('data-product-url');
    var price     = btn.getAttribute('data-product-price');
    var variantId = btn.getAttribute('data-variant-id');
    var list      = JSON.parse(localStorage.getItem('shopify_wishlist') || '[]');
    var idx       = list.findIndex(function(i) { return i.id === pid; });

    if (idx === -1) {
      list.push({ id: pid, variantId: variantId, title: title, image: image, url: url, price: price });
      btn.classList.add('is-wished');
    } else {
      list.splice(idx, 1);
      btn.classList.remove('is-wished');
    }

    localStorage.setItem('shopify_wishlist', JSON.stringify(list));
    window.dispatchEvent(new Event('storage'));
    document.dispatchEvent(new CustomEvent('wishlist-updated'));
  }

  // Initialise all card buttons on page load
  document.addEventListener('DOMContentLoaded', function() {
    var list = JSON.parse(localStorage.getItem('shopify_wishlist') || '[]');
    var ids  = list.map(function(i) { return i.id; });
    document.querySelectorAll('.card__wishlist-btn[data-product-id]').forEach(function(btn) {
      if (ids.indexOf(btn.getAttribute('data-product-id')) > -1) {
        btn.classList.add('is-wished');
      }
    });
  });
}
</script>
```

### ⚠️ `typeof` Guard — Critical for Card Snippets

```javascript
if (typeof toggleCardWishlist === 'undefined') {
  function toggleCardWishlist(btn) { ... }
}
```

`card-product.liquid` is rendered once per product card. On a collection page with 24 products, this snippet executes 24 times. Without the `typeof` guard, the function would be redefined 24 times and the `DOMContentLoaded` listener would fire 24 times — causing redundant operations and potential state conflicts. The guard ensures the function and event listener are only registered **once**, regardless of how many cards are on the page.

### Data Attributes Pattern

All product data is passed via `data-*` attributes on the button element, read at click time:

```html
data-product-id="{{ card_product.id }}"
data-variant-id="{{ card_product.selected_or_first_available_variant.id }}"
data-product-title="{{ card_product.title | escape }}"
data-product-url="{{ card_product.url }}"
data-product-image="{{ card_product.featured_media | image_url: width: 300 }}"
data-product-price="{{ card_product.price | money }}"
```

This avoids inline JavaScript and keeps the Liquid/JS boundary clean. The `| escape` filter on `title` prevents XSS from special characters in product names.



---

## UI Behavior & Visual States

### Product Page Button

| State | Background | Border | Text / Icon Color | Label |
|---|---|---|---|---|
| Default (not wishlisted) | `#ffffff` | `#C8952E` (Rich Gold) | `#C8952E` | Add to Wishlist |
| Hover (not wishlisted) | `#ffffff` | `#E8A838` (lighter gold) | `#E8A838` | Add to Wishlist |
| Active (wishlisted) | `#0B1D3A` (Deep Navy) | `#0B1D3A` | `#ffffff` | Wishlisted |

> **Hover logic:** The `onmouseover` / `onmouseout` inline handlers check `this.dataset.active` before applying hover styles. This prevents the hover style from overriding the active (wishlisted) state.

### Product Card Heart Icon

| State | Visual |
|---|---|
| Default (not wishlisted) | Hollow heart, black stroke, frosted white background |
| Hover | Scales up (`transform: scale(1.2)`), SVG fill becomes amber (`#E8A838`) |
| Active (wishlisted) — `.is-wished` | Heart filled amber (`#E8A838`) persistently |

State is controlled entirely by the CSS class `.is-wished` — no inline styles are used on the card button.

### Wishlist Page Cards

| Element | Behaviour |
|---|---|
| Card hover | Lifts with shadow (`translateY(-4px)`) and image zooms (`scale(1.05)`) |
| Remove button hover | Border turns red, background becomes `#FFF0F0`, icon stroke turns red |
| "Move to Cart" | Disables with "Adding…" while fetch is in flight; re-enables on error |
| Toast | Slides up from bottom, auto-dismisses after 3 seconds |
| Empty state | Shown when `localStorage` is empty or all items removed |

### Header Badge

| Wishlist empty | Badge hidden, icon uses `currentColor` |
|---|---|
| Wishlist has items | Badge visible with count, icon gains `.has-items` class (stroke: `#fff`) |

---

## Header Integration

The header icon and all other wishlist entry points communicate entirely through:

1. **`localStorage`** — the shared data store (key: `shopify_wishlist`)
2. **`wishlist-updated`** custom DOM event — fired after every add/remove on the same page
3. **`storage`** native window event — fires in other open tabs when `localStorage` changes
4. **`DOMContentLoaded`** — runs `updateHeaderWishlist()` on initial page load

This means the badge updates **instantly** on the same page after an add/remove, and **automatically** if the customer has multiple tabs open.

### How `updateHeaderWishlist` works

The complete script locates elements by **ID** rather than by class selector, making it more resilient to DOM structure changes:

```javascript
var badge = document.getElementById('wishlist-count-badge');  // the count span
var svg   = document.getElementById('header-wishlist-svg');   // the heart SVG
var link  = svg ? svg.closest('a') : null;                    // the <a> wrapper
```

The `link` reference is derived from the SVG using `.closest('a')` rather than querying `.header-wishlist-icon` by class. This means the `.has-items` class is toggled on the `<a>` tag directly, which is the correct element for CSS state targeting.

The entire function is wrapped in `try { } catch(e) {}` — if `localStorage` is blocked (e.g., in some private browsing modes or restrictive cookie settings), the function silently fails without throwing a visible console error.

### Event listener summary

| Event | Scope | Purpose |
|---|---|---|
| `DOMContentLoaded` | `document` | Initialises badge on page load |
| `wishlist-updated` | `document` | Reacts to add/remove on the **same** page |
| `storage` | `window` | Reacts to add/remove from **other tabs** |

> **Note:** The `storage` event in the complete script does **not** filter by `e.key`. It calls `updateHeaderWishlist()` on any `localStorage` change. This is safe for this use case but can be tightened to `if (e.key === 'shopify_wishlist')` if other `localStorage` keys are written frequently on the same page.

---

## Critical Logic Points

### ⚠️ `dataset.active` guards hover styling

```javascript
onmouseover="if(!this.dataset.active){ /* apply hover styles */ }"
```

Without this guard, hovering over an already-wishlisted button would revert its appearance to the default gold style temporarily. Always check `dataset.active` before overriding styles in hover handlers.

---

### ⚠️ Product ID is the unique key

```javascript
var idx = list.findIndex(function(i){ return i.id === pid; });
```

The product `id` (Shopify's numeric ID, injected via `{{ product.id }}`) is used as the unique identifier. Do **not** use `handle` or `title` as these can be ambiguous or change over time.

---

### ⚠️ `localStorage` is synchronous

`getWL()` always reads the current state from `localStorage` at the time of the click. This is intentional — it prevents stale state from the closure. Avoid caching the list in a variable outside the click handler.

---

### ⚠️ SVG fill vs stroke for active state

The heart icon uses both `fill` and `stroke` attributes. When active:

```javascript
icon.setAttribute('fill',   '#ffffff');  // fills the heart solid
icon.setAttribute('stroke', '#ffffff');  // keeps the outline white
```

When inactive:

```javascript
icon.setAttribute('fill',   'none');     // hollow heart
icon.setAttribute('stroke', '#C8952E'); // gold outline only
```

Both attributes must be set explicitly, as SVG does not cascade JS-set attributes like CSS properties.

---

### ⚠️ IIFE scope isolation

Each button script is wrapped in an **Immediately Invoked Function Expression (IIFE)**:

```javascript
(function(){ /* ... */ })();
```

This prevents variable collisions when multiple product cards appear on the same page (e.g., collection pages or recently viewed sections).

---

### ⚠️ `moveToCart` — `variantId` fallback

```javascript
var itemVariantId = item.variantId || item.variant_id || itemId;
```

Items saved from the product page button (Custom Liquid) do not store a `variantId`. The wishlist page falls back to using `itemId` (the product ID) as the variant ID. This works for simple products but **will fail for products with multiple variants** — only the card button stores the correct `variantId`. Ensure the product page button is updated to also store `variantId` if your store uses multi-variant products.

---

### ⚠️ String vs Number comparison in `removeFromWishlist`

```javascript
current.filter(function (i) { return String(i.id) !== String(id); });
```

IDs are always cast to `String` before comparison. Shopify product IDs are numeric, but they may be stored or passed as either type depending on the entry point. Coercing both sides to string prevents silent mismatches where `"123" !== 123`.

---

### ⚠️ `typeof` guard on card snippet

```javascript
if (typeof toggleCardWishlist === 'undefined') { ... }
```

See the [Product Card Button section](#product-card-button-card-productliquid) for a full explanation. This is the most critical guard in the entire feature — without it, collection pages silently register dozens of duplicate event listeners.



The current architecture is intentionally lightweight and can be extended in several ways:

### Short-term additions

| Feature | Approach |
|---|---|
| **Wishlist page rendering** | Read `localStorage` on `/pages/wishlist` and dynamically render product cards via JS |
| **Item count in nav** | Already supported — the header badge shows the count |
| **Share wishlist** | Encode the wishlist array as a URL query string |
| **Wishlist limit** | Add a guard in the click handler: `if (list.length >= MAX) return;` |

### Medium-term additions

| Feature | Approach |
|---|---|
| **Backend sync** | Replace `localStorage` reads/writes with `fetch()` calls to a Shopify metafield or custom app endpoint — keep `setActive()` and UI logic intact |
| **Customer-specific wishlists** | On login, merge `localStorage` wishlist with server-stored wishlist using customer ID |
| **Wishlist count in theme settings** | Expose count as a Liquid global via a Shopify app or script tag |

### Long-term

- The IIFE pattern scales to collection pages and quick-view modals with no modification
- Replacing `localStorage` with a server API only requires changing `getWL()` and `saveWL()` — the entire UI layer remains unchanged

---

## Best Practices

### Performance

- **Avoid reading `localStorage` on every render** — the current pattern reads only on page load and on click, which is efficient
- **Debounce the `storage` event listener** if the wishlist page renders many items reactively

### Maintainability

- **Centralise color tokens** — consider moving `#C8952E`, `#0B1D3A`, and `#E8A838` to CSS custom properties on `:root` so they can be updated in one place
- **Avoid inline `style` attributes** for state — prefer toggling CSS classes (e.g., `.wl-btn--active`) so styles are managed in CSS, not JavaScript

```css
/* Preferred approach for future refactoring */
.wl-btn--active {
  background: #0B1D3A;
  border-color: #0B1D3A;
  color: #ffffff;
}
```

- **The `{{ product.id | json }}` pattern is safe** — Shopify IDs are always integers, so XSS is not a concern here, but use `| json` for all other Liquid-injected values (e.g., `title`)

### Accessibility

- The button has a visible text label (`<span>`) alongside the icon — this is correct
- Consider adding `aria-pressed="true/false"` to the button, toggled by `setActive()`, for screen reader compatibility:

```javascript
btn.setAttribute('aria-pressed', on ? 'true' : 'false');
```

---

## Testing Checklist

### Product Page Button

- [ ] Button renders with gold border and white background on page load
- [ ] Clicking "Add to Wishlist" changes button to navy/filled state, label to "Wishlisted"
- [ ] Clicking again removes from wishlist and reverts to default state
- [ ] Hovering over the default button shows lighter gold (`#E8A838`) highlight
- [ ] Hovering over the **wishlisted** button does **not** change its appearance
- [ ] Refreshing the page preserves the wishlisted state

### Product Card Heart Icon

- [ ] Heart icon appears in the top-right corner of each product card image
- [ ] Heart is hollow (unfilled) for unwishlisted products on page load
- [ ] Heart is filled amber for already-wishlisted products on page load
- [ ] Clicking the heart adds the item and fills it amber immediately
- [ ] Clicking the heart again removes the item and returns it to hollow
- [ ] On a collection page with multiple cards, each card's state is independent
- [ ] Adding from a card stores `variantId` and `price` in `localStorage`

### Wishlist Page (`/pages/wishlist`)

- [ ] Page renders product cards from `localStorage` on load
- [ ] Subtitle shows correct item count (e.g., "3 items saved")
- [ ] Empty state shows when wishlist is empty
- [ ] Remove (×) button removes the card from the DOM and updates the count
- [ ] "View Product" link navigates to the correct product page
- [ ] "Move to Cart" adds the item to the Shopify cart and redirects to `/cart`
- [ ] "Move to Cart" removes the item from `localStorage` after a successful add
- [ ] "Move to Cart" re-enables and shows an error toast if the cart add fails
- [ ] Toast notification appears and auto-dismisses after 3 seconds
- [ ] Cards without images show the fallback placeholder emoji

### Header Badge

- [ ] Badge is hidden when wishlist is empty
- [ ] Badge shows the correct count after adding a product
- [ ] Badge updates immediately after add/remove on the same page
- [ ] Badge updates without page reload when wishlist is modified in another tab
- [ ] Clicking the header icon navigates to `/pages/wishlist`

### Data Integrity

- [ ] `localStorage` key `shopify_wishlist` contains a valid JSON array after adding items
- [ ] Items from the card button have `variantId` and `price` fields
- [ ] Removing all items results in an empty array `[]`, not `null`
- [ ] No JavaScript console errors during normal add/remove/cart flows

### Cross-browser & Responsive

- [ ] Feature works in Chrome, Firefox, Safari, and Edge
- [ ] Feature works on mobile (iOS Safari, Android Chrome)
- [ ] Wishlist page grid shows 2 columns on mobile, 3 on tablet, 4 on desktop
- [ ] Feature degrades gracefully if `localStorage` is blocked (private browsing)

---

*Documentation maintained by the development team. Last updated for Trade theme compatibility.*

