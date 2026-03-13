# Shopify Horizon Theme — Architecture Analysis

**Theme:** Horizon v3.4.0
**Author:** Shopify
**Last analyzed:** March 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Directory Map](#2-directory-map)
3. [Block Catalog](#3-block-catalog)
4. [Template Breakdown](#4-template-breakdown)
5. [Styling Approach](#5-styling-approach)
6. [JavaScript Strategy](#6-javascript-strategy)
7. [Configuration System](#7-configuration-system)
8. [Key Patterns](#8-key-patterns)
9. [Architecture Diagram](#9-architecture-diagram)

---

## 1. Executive Summary

Horizon is a **block-first** Shopify theme that inverts the traditional section-heavy architecture. Instead of putting all logic and markup into monolithic section files, Horizon uses **94 blocks** (44 public, 50 internal) as the primary building units, with **39 thin section files** that act as containers. Each block delegates its rendering to one of **93 reusable snippets**, creating a clean separation between schema definition (block), presentation logic (snippet), and page composition (template). This three-layer delegation pattern—Block → Snippet → HTML—is the architectural backbone of the entire theme.

The styling system is built entirely on **CSS custom properties** (design tokens) generated from Liquid templates at render time. There is no build step, no preprocessor, and no CSS framework. A single `base.css` file (4,900+ lines) provides all styles, and dynamic theming flows through a pipeline: merchant settings → Liquid snippets (`theme-styles-variables.liquid`, `color-schemes.liquid`) → CSS custom properties → component styles. Six color schemes are defined with full token sets covering backgrounds, text, buttons, inputs, variants, and borders—each auto-adjusting opacity values based on background brightness for dark/light mode support.

JavaScript follows a **vanilla Web Components** approach using a custom `Component` base class that extends `HTMLElement`. Components use a declarative event system (`on:click="component/method"`) bound via HTML attributes, a `refs` system for child element discovery, and a DOM morphing engine for efficient partial updates via the Shopify Section Rendering API. There are no framework dependencies—no React, Alpine, or Stimulus. Performance is a first-class concern: the theme detects low-power devices, uses `IntersectionObserver` for lazy loading, supports the View Transitions API for page-level animations, and employs `requestIdleCallback` and `scheduler.yield()` for non-blocking initialization.

---

## 2. Directory Map

```
horizon/
├── assets/                          # 130+ files: JS modules, CSS, SVG icons
│   ├── base.css                     # Main stylesheet (4,900+ lines)
│   ├── overflow-list.css            # Standalone overflow list styles
│   ├── template-giftcard.css        # Gift card page styles
│   ├── component.js                 # Component base class (Web Components)
│   ├── events.js                    # ThemeEvents system & custom event classes
│   ├── utilities.js                 # Debounce, throttle, view transitions, helpers
│   ├── morph.js                     # DOM diffing/morphing engine
│   ├── section-renderer.js          # Section Rendering API integration
│   ├── section-hydration.js         # Section lazy hydration
│   ├── view-transitions.js          # View Transitions API support
│   ├── performance.js               # Performance monitoring & low-power detection
│   ├── focus.js                     # Focus management utilities
│   ├── scrolling.js                 # Scroll behavior utilities
│   ├── money-formatting.js          # Currency formatting
│   ├── product-form.js              # Product form & add-to-cart logic
│   ├── variant-picker.js            # Variant selection component
│   ├── media-gallery.js             # Product media gallery
│   ├── cart-drawer.js               # Cart drawer component
│   ├── header.js                    # Header/navigation component
│   ├── header-menu.js               # Navigation menu component
│   ├── header-drawer.js             # Mobile menu drawer
│   ├── predictive-search.js         # Search autocomplete
│   ├── facets.js                    # Collection filtering
│   ├── slideshow.js                 # Slideshow/carousel component
│   ├── dialog.js                    # Modal dialog component
│   ├── fly-to-cart.js               # Add-to-cart animation
│   ├── quick-add.js                 # Quick add modal
│   ├── accordion-custom.js          # Accordion component
│   ├── comparison-slider.js         # Before/after comparison slider
│   ├── marquee.js                   # Scrolling marquee text
│   ├── jumbo-text.js                # Jumbo/oversized text component
│   ├── product-card.js              # Product card interactions
│   ├── product-price.js             # Dynamic price display
│   ├── product-inventory.js         # Inventory status component
│   ├── product-sku.js               # SKU display component
│   ├── product-recommendations.js   # Product recommendations loader
│   ├── recently-viewed-products.js  # Recently viewed tracking
│   ├── sticky-add-to-cart.js        # Sticky add-to-cart bar
│   ├── localization.js              # Language/currency switcher
│   ├── paginated-list.js            # Infinite scroll / pagination
│   ├── icon-*.svg                   # 25+ SVG icon files
│   ├── global.d.ts                  # TypeScript type declarations
│   └── jsconfig.json                # JS project configuration
│
├── blocks/                          # 94 files: theme building blocks
│   ├── [44 public blocks]           # Merchant-facing blocks (no prefix)
│   │   ├── accordion.liquid
│   │   ├── add-to-cart.liquid
│   │   ├── button.liquid
│   │   ├── group.liquid             # Universal container block
│   │   ├── text.liquid
│   │   ├── image.liquid
│   │   ├── video.liquid
│   │   ├── price.liquid
│   │   ├── variant-picker.liquid
│   │   ├── product-card.liquid
│   │   └── ... (34 more)
│   │
│   └── [50 internal blocks]         # Composition blocks (_ prefix)
│       ├── _card.liquid             # Card layout (47 settings)
│       ├── _slide.liquid            # Slideshow slide (47 settings)
│       ├── _product-card.liquid     # Product card composition
│       ├── _product-details.liquid  # Product details group
│       ├── _product-media-gallery.liquid
│       ├── _header-menu.liquid
│       ├── _content.liquid
│       └── ... (43 more)
│
├── config/                          # Theme configuration
│   ├── settings_schema.json         # Merchant-facing settings schema (68KB)
│   └── settings_data.json           # Current settings values + presets
│
├── layout/                          # Page layouts (entry points)
│   ├── theme.liquid                 # Main layout (all pages)
│   └── password.liquid              # Password/coming-soon layout
│
├── locales/                         # 47 files: translations
│   ├── en.default.json              # English (default) — storefront strings
│   ├── en.default.schema.json       # English — settings/admin UI strings
│   ├── [20 language].json           # Storefront translations
│   ├── [16 language].schema.json    # Settings translations
│   └── [10 language-only].json      # Storefront-only (bg, hr, hu, id, etc.)
│
├── sections/                        # 39 files: section containers
│   ├── _blocks.liquid               # Generic block container section
│   ├── section.liquid               # Universal custom section
│   ├── header.liquid                # Site header
│   ├── footer.liquid                # Site footer
│   ├── hero.liquid                  # Hero banner
│   ├── slideshow.liquid             # Image carousel
│   ├── product-information.liquid   # Product page main section
│   ├── main-collection.liquid       # Collection page
│   ├── main-cart.liquid             # Cart page
│   ├── header-group.json            # Header section group
│   ├── footer-group.json            # Footer section group
│   └── ... (28 more)
│
├── snippets/                        # 93 files: reusable Liquid partials
│   ├── theme-styles-variables.liquid # CSS custom properties (688 lines)
│   ├── color-schemes.liquid         # Dynamic color scheme generation
│   ├── stylesheets.liquid           # CSS asset loading
│   ├── scripts.liquid               # JS import map & script loading (299 lines)
│   ├── fonts.liquid                 # Font preloading
│   ├── meta-tags.liquid             # SEO meta tags
│   ├── section.liquid               # Section wrapper template
│   ├── group.liquid                 # Group block renderer
│   ├── product-card.liquid          # Product card template
│   ├── price.liquid                 # Price display template
│   ├── button.liquid                # Button component template
│   ├── image.liquid                 # Image component template
│   └── ... (81 more)
│
├── templates/                       # 13 files: page templates (JSON + Liquid)
│   ├── index.json                   # Homepage
│   ├── product.json                 # Product page
│   ├── collection.json              # Collection page
│   ├── cart.json                    # Cart page
│   ├── blog.json                   # Blog listing
│   ├── article.json                # Blog post
│   ├── page.json                   # Generic page
│   ├── page.contact.json           # Contact page variant
│   ├── search.json                 # Search results
│   ├── 404.json                    # Not found page
│   ├── list-collections.json       # All collections
│   ├── password.json               # Password page
│   └── gift_card.liquid            # Gift card (Liquid, not JSON)
│
└── .cursor/rules/                   # Development guidelines (47 MDC files)
    ├── blocks.mdc                   # Block development standards
    ├── sections.mdc                 # Section development standards
    ├── snippets.mdc                 # Snippet development standards
    ├── schemas.mdc                  # Schema patterns
    ├── css-standards.mdc            # CSS conventions
    ├── javascript-standards.mdc     # JS conventions
    ├── *-accessibility.mdc          # 20+ accessibility pattern guides
    └── examples/                    # Example block/section/snippet templates
```

---

## 3. Block Catalog

### Naming Convention

- **Public blocks** (no prefix): Visible to merchants in the theme editor. These appear in the block picker.
- **Internal blocks** (`_` prefix): Used for composition within sections. Not directly selectable by merchants—they're added programmatically as static blocks or within specific section schemas.

### Public Blocks (44)

| Block | Purpose | Has JS? | Settings | Can Nest? | Key Snippets |
|-------|---------|---------|----------|-----------|--------------|
| `accelerated-checkout` | Dynamic checkout buttons (Shop Pay, etc.) | No | 0 | No | — |
| `accordion` | Expandable content sections | Yes (`accordion-custom.js`) | 4 | Yes | — |
| `add-to-cart` | Add to cart button | Yes | 0 | No | — |
| `button` | Configurable CTA button | No | 5 | No | `button` |
| `buy-buttons` | Add-to-cart + dynamic checkout group | Yes | 0 | Yes | — |
| `collection-card` | Collection preview card | No | 9 | Yes | `collection-card` |
| `collection-title` | Collection title display | No | 14 | No | `text` |
| `comparison-slider` | Before/after image comparison | Yes (`comparison-slider.js`) | ~20 | No | — |
| `contact-form` | Contact form with fields | Yes | ~15 | No | — |
| `contact-form-submit-button` | Form submit button | No | 0 | No | `button` |
| `custom-liquid` | Raw Liquid code block | No | 1 | No | — |
| `email-signup` | Newsletter signup form | Yes | ~10 | No | — |
| `featured-collection` | Product grid from collection | No | ~20 | Yes | — |
| `filters` | Collection/search facets | Yes (`facets.js`) | ~15 | No | — |
| `follow-on-shop` | Follow on Shop button | No | 3 | No | — |
| `footer-copyright` | Copyright text | No | 3 | No | — |
| `footer-policy-list` | Policy links list | No | 3 | No | — |
| `group` | Universal container/layout block | No | ~30 | Yes | `group` |
| `icon` | Icon display | No | 2 | No | `icon` |
| `image` | Image display | No | ~10 | No | `image` |
| `jumbo-text` | Large decorative text | Yes (`jumbo-text.js`) | ~15 | No | `text` |
| `logo` | Logo image | No | 4 | No | `image` |
| `menu` | Navigation menu | Yes (`header-menu.js`) | ~13 | No | `header-drawer`, `mega-menu-list` |
| `page` | Page content container | No | 1 | Yes | — |
| `page-content` | Page body content | No | 0 | Yes | — |
| `payment-icons` | Accepted payment method icons | No | 2 | No | — |
| `popup-link` | Link that opens a popup | No | 3 | No | — |
| `price` | Product price display | No | 3 | No | `price` |
| `product-card` | Product card for grids | No | 4 | Yes | `product-card` |
| `product-custom-property` | Custom metafield display | No | 3 | No | — |
| `product-description` | Product description text | No | 2 | No | — |
| `product-inventory` | Stock level indicator | No | 1 | No | — |
| `product-recommendations` | Related products grid | No | ~15 | Yes | — |
| `product-title` | Product title display | No | 3 | No | — |
| `quantity` | Quantity selector | Yes | 2 | No | — |
| `review` | Product review/rating | No | 4 | No | — |
| `sku` | SKU display | No | 0 | No | — |
| `social-links` | Social media icon links | No | 0 | Yes | — |
| `spacer` | Vertical spacing block | No | 1 | No | — |
| `swatches` | Color/option swatch picker | Yes (`variant-picker.js`) | ~12 | No | — |
| `text` | Rich text content block | No | ~25 | No | `text` |
| `variant-picker` | Product variant selector | Yes (`variant-picker.js`) | ~12 | No | — |
| `video` | Video embed/player | No | ~12 | No | `video` |

### Internal Blocks (50)

| Block | Purpose | Settings | Can Nest? | Key Snippets |
|-------|---------|----------|-----------|--------------|
| `_accordion-row` | Single accordion item | 8 | Yes | `icon-or-image` |
| `_announcement` | Announcement bar item | 10 | No | `typography-style` |
| `_blog-post-card` | Blog post card wrapper | 1 | No | — |
| `_blog-post-content` | Blog post body | 0 | No | — |
| `_blog-post-description` | Blog post excerpt | 21 | No | `spacing-padding`, `typography-style` |
| `_blog-post-featured-image` | Blog post hero image | 17 | No | `size-style`, `spacing-style`, `border-override` |
| `_blog-post-image` | Blog post inline image | 7 | No | `border-override` |
| `_blog-post-info-text` | Blog post metadata | 9 | No | `spacing-style` |
| `_card` | **Card layout container** | **47** | Yes | `background-media`, `overlay`, `layout-panel-style`, `spacing-style` |
| `_carousel-content` | Carousel slide content | 14 | Yes | `slideshow`, `gap-style`, `spacing-style`, `border-override` |
| `_cart-products` | Cart line items | 9 | No | `cart-products` |
| `_cart-summary` | Cart totals & checkout | 10 | No | `cart-summary` |
| `_cart-title` | Cart page title | 14 | No | `spacing-style`, `cart-bubble` |
| `_collection-card` | Collection card in lists | 14 | Yes | `collection-card`, `resource-image` |
| `_collection-card-image` | Collection card image | 8 | No | `resource-image` |
| `_collection-image` | Collection banner image | 8 | No | — |
| `_collection-info` | Collection header info | 8 | Yes | `slideshow-controls` |
| `_collection-link` | Collection link item | 2 | No | — |
| `_content` | Content container with appearance | 3 | Yes | `group` |
| `_content-without-appearance` | Content container (no styling) | 3 | Yes | `group` |
| `_divider` | Horizontal rule | 5 | No | `divider` |
| `_featured-blog-posts-card` | Featured blog card | 16 | No | `resource-image` |
| `_featured-blog-posts-image` | Featured blog image | 8 | No | `resource-image` |
| `_featured-blog-posts-title` | Featured blog title | 36 | No | `text` |
| `_featured-product` | Featured product card | 1 | No | `product-card` |
| `_featured-product-gallery` | Featured product images | 3 | No | `quick-add`, `card-gallery` |
| `_featured-product-information-carousel` | Featured product media carousel | 27 | No | `product-media-gallery-content` |
| `_featured-product-price` | Featured product price | 8 | No | `price` |
| `_footer-social-icons` | Footer social media icons | 0 | Yes | — |
| `_header-logo` | Header logo | 5 | No | `image` |
| `_header-menu` | Header navigation menu | 20 | No | `header-drawer`, `mega-menu-list`, `overflow-list` |
| `_heading` | Heading text | 20 | No | `text` |
| `_hotspot-product` | Product hotspot overlay | 3 | No | `image`, `price`, `quick-add` |
| `_image` | Internal image block | 5 | No | — |
| `_inline-collection-title` | Inline collection title | 8 | No | `typography-style` |
| `_inline-text` | Inline text element | 8 | No | `typography-style` |
| `_layered-slide` | Layered slideshow slide | **45** | Yes | `overlay`, `spacing-style`, `layout-panel-style` |
| `_marquee` | Scrolling marquee | 8 | Yes | — |
| `_media` | Media block with appearance | 17 | No | `media` |
| `_media-without-appearance` | Media block (no styling) | 8 | No | `media` |
| `_product-card` | Product card in grids | 8 | Yes | `product-card` |
| `_product-card-gallery` | Product card image gallery | 9 | No | `quick-add`, `card-gallery` |
| `_product-card-group` | Product card layout group | **42** | Yes | `group` |
| `_product-details` | Product details group | **39** | Yes | `group` |
| `_product-list-button` | Product list CTA button | 8 | No | `button` |
| `_product-list-content` | Product list header | 29 | Yes | `group` |
| `_product-list-text` | Product list title text | **40** | No | `text` |
| `_product-media-gallery` | Product page media gallery | 29 | No | `product-media-gallery-content` |
| `_search-input` | Search input field | 4 | Yes | — |
| `_slide` | Slideshow slide | **47** | Yes | `overlay`, `spacing-style`, `layout-panel-style`, `slideshow-slide` |
| `_social-link` | Single social media link | 3 | No | — |

### Block Schema Pattern

Every block follows this structure:

```liquid
{%- capture children %}
  {% content_for 'blocks' %}    {%- comment -%} Only if block nests children {%- endcomment -%}
{% endcapture %}

{% render 'snippet-name', children: children, settings: block.settings, shopify_attributes: block.shopify_attributes %}

{% schema %}
{
  "name": "t:names.block_name",     // Translated name
  "tag": null,                       // No wrapping HTML tag (block renders its own)
  "blocks": [                        // Child block types allowed (if nesting)
    { "type": "@theme" },            // Any public block
    { "type": "@app" },              // App blocks
    { "type": "_divider" }           // Specific internal blocks
  ],
  "settings": [ ... ],              // Block-specific settings
  "presets": [                       // Only on public blocks
    { "name": "t:names.block_name", "category": "t:categories.category_name" }
  ]
}
{% endschema %}
```

---

## 4. Template Breakdown

All templates (except `gift_card.liquid`) are **JSON templates** that compose sections and blocks declaratively.

### Template → Section → Block Composition

#### `index.json` — Homepage

| Section | Type | Blocks |
|---------|------|--------|
| Hero | `hero` | `text` (heading), `text` (subheading), `button` |
| Product List | `product-list` | `_product-list-content` → `_product-list-text`, `_product-card` → `_product-card-gallery`, `text`, `price`, `swatches` |

#### `product.json` — Product Page

| Section | Type | Blocks |
|---------|------|--------|
| Main | `product-information` | `_product-media-gallery` (static), `_product-details` (static) → `group` (Header: `product-title`, `price`), `divider`, `variant-picker`, `buy-buttons` → `add-to-cart`, `accelerated-checkout`, `product-description` |
| Recommendations | `product-recommendations` | `text` (heading), `_product-card` blocks |

#### `collection.json` — Collection Page

| Section | Type | Blocks |
|---------|------|--------|
| Heading | `section` | `text` (title), `text` (description) |
| Main | `main-collection` | `filters`, `_product-card` → `_product-card-gallery`, `text`, `price`, `swatches` |

#### `cart.json` — Cart Page

| Section | Type | Blocks |
|---------|------|--------|
| Main Cart | `main-cart` | `_cart-title`, `_cart-products`, `_cart-summary` |
| Related Products | `product-list` | `_product-list-content` → `_product-list-text`, `_product-card` blocks |

#### `article.json` — Blog Post

| Section | Type | Blocks |
|---------|------|--------|
| Main | `main-blog-post` | `text` (title), `_blog-post-info-text`, `_blog-post-featured-image`, `_blog-post-content` |

#### `blog.json` — Blog Listing

| Section | Type | Blocks |
|---------|------|--------|
| Main | `main-blog` | `text` (title), `_blog-post-card` → `_heading`, `_blog-post-info-text`, `_blog-post-image` |

#### `search.json` — Search Results

| Section | Type | Blocks |
|---------|------|--------|
| Header | `search-header` | `_heading`, `_search-input` |
| Results | `search-results` | `filters`, `_product-card` blocks |

#### `page.json` — Generic Page

| Section | Type | Blocks |
|---------|------|--------|
| Main | `main-page` | `text` (heading), `page-content` |

#### `page.contact.json` — Contact Page

| Section | Type | Blocks |
|---------|------|--------|
| Content | `main-page` | `text` (heading), `page-content` |
| Form | `section` | `contact-form`, `contact-form-submit-button` |

#### `404.json` — Not Found

| Section | Type | Blocks |
|---------|------|--------|
| Main | `main-404` | `text` (heading), `text` (body), `button` |
| Recommendations | `product-list` | `_product-list-content`, `_product-card` blocks |

#### `list-collections.json` — All Collections

| Section | Type | Blocks |
|---------|------|--------|
| Main | `main-collection-list` | `group` (header) → `text`, `_collection-card` → `text`, `_collection-card-image` |

#### `password.json` — Password/Coming Soon

| Section | Type | Blocks |
|---------|------|--------|
| Main | `password` | `logo`, `text` (title), `text` (description), `email-signup` |

#### `gift_card.liquid` — Gift Card (Liquid Template)

This is the only non-JSON template. It uses `template-giftcard.css` and `qr-code-generator.js` / `qr-code-image.js` for the QR code. It renders the gift card balance, code, and a QR code image.

---

## 5. Styling Approach

### No Build Step

Horizon uses **zero build tools**. All CSS is authored as plain CSS and delivered as-is. There is no Sass, PostCSS, Tailwind, or CSS-in-JS. The only CSS files are:

- `assets/base.css` — Main stylesheet (4,900+ lines)
- `assets/overflow-list.css` — Overflow list component styles
- `assets/template-giftcard.css` — Gift card page styles

Both are loaded via `snippets/stylesheets.liquid`:
```liquid
{{ 'overflow-list.css' | asset_url | preload_tag: as: 'style' }}
{{ 'base.css' | asset_url | stylesheet_tag: preload: true }}
```

### CSS Custom Properties (Design Tokens)

All design tokens are generated as CSS custom properties by `snippets/theme-styles-variables.liquid` (688 lines). This snippet reads merchant settings and outputs `:root` variables.

#### Typography Tokens

```css
/* Font families (4 slots) */
--font-body-family: ...;
--font-subheading-family: ...;
--font-heading-family: ...;
--font-accent-family: ...;

/* Font sizes (fixed scale) */
--font-size-3xs: 0.625rem;    /* 10px */
--font-size-2xs: 0.6875rem;   /* 11px */
--font-size-xs: 0.75rem;      /* 12px */
--font-size-sm: 0.8125rem;    /* 13px */
--font-size-md: 0.875rem;     /* 14px */
--font-size-lg: 1rem;         /* 16px */
--font-size-xl: 1.125rem;     /* 18px */
--font-size-2xl: 1.25rem;     /* 20px */
--font-size-3xl: 1.5rem;      /* 24px */
--font-size-4xl: 2rem;        /* 32px */
--font-size-5xl: 2.5rem;      /* 40px */
--font-size-6xl: 3.5rem;      /* 56px */

/* Heading sizes (dynamic, from settings, using clamp() for fluid sizing) */
--font-size-h1: clamp(...);
--font-size-h2: clamp(...);
/* ... through h6 */

/* Line heights */
--line-height-display: 1.05;
--line-height-heading: 1.15;
--line-height-body: 1.5;

/* Letter spacing */
--letter-spacing-tight: -0.02em;
--letter-spacing-normal: 0;
--letter-spacing-loose: 0.06em;
```

#### Spacing Tokens

```css
/* Margin scale */
--margin-3xs: 0.125rem;   --margin-2xs: 0.25rem;
--margin-xs: 0.5rem;      --margin-sm: 0.75rem;
--margin-md: 1rem;        --margin-lg: 1.5rem;
--margin-xl: 2rem;        --margin-2xl: 2.5rem;
--margin-3xl: 3rem;       --margin-4xl: 3.5rem;
--margin-5xl: 4rem;       --margin-6xl: 5rem;

/* Padding scale */
--padding-3xs: 0.125rem;  --padding-2xs: 0.25rem;
/* ... same pattern through 6xl */

/* Gap scale */
--gap-3xs: 0.125rem;  --gap-2xs: 0.25rem;
/* ... through 3xl: 3rem */
```

#### Z-Index Layers

```css
--layer-section-background: -2;
--layer-lowest: -1;
--layer-base: 0;
--layer-flat: 1;
--layer-raised: 2;
--layer-heightened: 4;
--layer-sticky: 8;
--layer-window-overlay: 10;
--layer-header-menu: 12;
--layer-overlay: 16;
--layer-menu-drawer: 18;
--layer-temporary: 20;
```

#### Layout Dimensions

```css
/* Content widths */
--content-width-narrow: 36rem;
--content-width-normal: 42rem;
--content-width-wide: 46rem;

/* Page widths */
--page-width-narrow: 90rem;
--page-width-normal: 120rem;
--page-width-wide: 150rem;

/* Section heights */
--section-height-small: clamp(15rem, 30svh, 20rem);
--section-height-medium: clamp(20rem, 50svh, 27.5rem);
--section-height-large: clamp(25rem, 80svh, 35rem);
```

#### Animation Tokens

```css
--ease-out-cubic: cubic-bezier(0.33, 1, 0.68, 1);
--ease-out-quad: cubic-bezier(0.5, 1, 0.89, 1);
--ease-in-out-quad: cubic-bezier(0.45, 0, 0.55, 1);

--animation-speed-fast: 0.0625s;
--animation-speed-normal: 0.125s;
--animation-speed-slow: 0.2s;

/* Spring animations (linear() timing functions) */
--spring-d300-b0: linear(...);
--spring-d280-b0: linear(...);
/* ... 5 spring presets */
```

### Color Scheme System

`snippets/color-schemes.liquid` generates 6 color schemes, each with a full token set:

```css
.color-scheme-1 {
  --color-background: #FFFFFF;
  --color-background-rgb: 255, 255, 255;
  --color-foreground: #000000;
  --color-foreground-heading: #000000;
  --color-primary: #000F9F;
  --color-primary-hover: #000000;
  --color-border: #E6E6E6;
  --color-shadow: #000000;

  /* Button tokens per scheme */
  --color-primary-button-background: #000F9F;
  --color-primary-button-text: #FFFFFF;
  --color-primary-button-border: #000F9F;
  --color-primary-button-hover-background: #000F9F;
  --color-primary-button-hover-text: #FFFFFF;
  --color-primary-button-hover-border: #000F9F;

  --color-secondary-button-background: #FFFFFF;
  --color-secondary-button-text: #000000;
  /* ... secondary button tokens */

  /* Input tokens per scheme */
  --color-input-background: #FFFFFF;
  --color-input-text: #000000;
  --color-input-border: #000000;
  --color-input-hover-background: #F5F5F5;

  /* Variant picker tokens per scheme */
  --color-variant-background: #FFFFFF;
  --color-variant-text: #000000;
  --color-variant-border: #E6E6E6;
  /* ... hover, selected states */

  /* Auto-adjusted opacity based on background brightness */
  --opacity-5-15: 0.05;   /* 0.15 for dark backgrounds */
  --opacity-10-25: 0.1;   /* 0.25 for dark backgrounds */
  /* ... dynamic opacity tokens */
}
```

**Dark/light auto-detection:** The color scheme snippet calculates the brightness of the background color. If brightness < 64 (dark background), opacity values are increased to maintain visibility.

### Responsive Design

- **Approach:** Mobile-first with progressive enhancement
- **Breakpoints:**
  - `750px` — Tablet / small desktop
  - `990px` — Desktop
  - `1200px` — Large desktop
  - `1400px` — Wide desktop
- **Fluid typography:** `clamp()` functions for heading sizes that scale smoothly between breakpoints
- **Section heights:** Use `svh` (small viewport height) units for mobile-safe viewport calculations
- **Touch targets:** Minimum 44px for interactive elements

### Component-Scoped Styles

Blocks and snippets can include scoped CSS using `{% stylesheet %}` tags:

```liquid
{% stylesheet %}
  .group-block__link {
    position: absolute;
    inset: 0;
  }
{% endstylesheet %}
```

These styles are extracted and deduplicated by Shopify's rendering engine.

---

## 6. JavaScript Strategy

### No Framework — Vanilla Web Components

Horizon uses **zero JavaScript frameworks**. All interactivity is built on native Web Components with a custom `Component` base class.

### Component Base Class (`assets/component.js`)

```javascript
export class Component extends DeclarativeShadowElement {
  refs = {};           // Auto-populated from ref="name" attributes
  requiredRefs;        // Validation for required child elements

  get roots() {        // Shadow root or self
    return this.shadowRoot ? [this, this.shadowRoot] : [this];
  }

  connectedCallback() {
    super.connectedCallback();
    registerEventListeners();  // Set up declarative events
    this.#updateRefs();         // Populate refs
    // Set up MutationObserver for DOM changes
  }

  updatedCallback() {          // Called after Section Rendering API morph
    this.#updateRefs();
  }

  disconnectedCallback() {     // Cleanup
    this.#mutationObserver.disconnect();
  }
}
```

### Declarative Event System

Events are bound via HTML attributes, not JavaScript:

```html
<!-- Basic event -->
<button on:click="component/handleClick">Click me</button>

<!-- With data -->
<button on:click="component/selectVariant?id=123">Select</button>

<!-- Multiple events -->
<input on:input="component/search" on:keydown="component/handleKey">
```

**Supported events:** `click`, `change`, `select`, `focus`, `blur`, `submit`, `input`, `keydown`, `keyup`, `toggle`
**Expensive events** (only bound when used): `pointerenter`, `pointerleave`

### Refs System

Child elements are discovered via `ref` attributes:

```html
<div ref="container">
  <button ref="items[]">Item 1</button>   <!-- Array ref -->
  <button ref="items[]">Item 2</button>
  <span ref="count">0</span>              <!-- Single ref -->
</div>
```

```javascript
class MyComponent extends Component {
  connectedCallback() {
    super.connectedCallback();
    console.log(this.refs.container);  // <div>
    console.log(this.refs.items);      // [<button>, <button>]
    console.log(this.refs.count);      // <span>
  }
}
```

A `MutationObserver` automatically updates refs when the DOM changes.

### Custom Event System (`assets/events.js`)

```javascript
// Event types (static constants)
ThemeEvents.variantSelected
ThemeEvents.variantUpdate
ThemeEvents.cartUpdate
ThemeEvents.cartError
ThemeEvents.mediaStartedPlaying
ThemeEvents.quantitySelectorUpdate
ThemeEvents.megaMenuHover
ThemeEvents.zoomMediaSelected
ThemeEvents.discountUpdate
ThemeEvents.FilterUpdate

// Custom event classes
new VariantSelectedEvent({ variant, product });
new CartUpdateEvent({ cart, source });
new CartErrorEvent({ error });
new FilterUpdateEvent({ url, searchParams });
// ... etc.
```

### DOM Morphing (`assets/morph.js`)

Instead of full page reloads or innerHTML replacement, Horizon uses a DOM morphing/diffing engine:

```javascript
morph(oldTree, newTree, {
  childrenOnly: true,
  reject(node) {
    // Skip empty text nodes, already-initialized components
  },
  onBeforeUpdate(fromEl, toEl) {
    // Preserve specific attributes during morph
    // (product-grid-view, cart-summary-sticky, etc.)
  },
  onAfterUpdate(element) {
    // Call updatedCallback() on morphed components
  }
});
```

This enables partial updates that preserve component state, scroll position, and focus.

### Section Rendering API (`assets/section-renderer.js`)

```javascript
class SectionRenderer {
  cache = new Map();             // Section HTML cache
  abortControllers = new Map();  // Request cancellation
  pendingPromises = new Map();   // Request deduplication

  async renderSection(sectionId, { cache, mode, url }) {
    // Fetch rendered section HTML from Shopify
    // Morph the DOM instead of replacing innerHTML
    // Support 'hydration' mode (partial) and 'full' mode
  }
}
```

### Key JavaScript Modules

| Module | Purpose | Size |
|--------|---------|------|
| `component.js` | Web Component base class | 349 lines |
| `events.js` | Custom event system | 291 lines |
| `utilities.js` | Helpers (debounce, throttle, view transitions) | 200+ lines |
| `morph.js` | DOM diffing engine | 100+ lines |
| `section-renderer.js` | Section Rendering API | 100+ lines |
| `product-form.js` | Product add-to-cart logic | — |
| `variant-picker.js` | Variant selection & updates | — |
| `media-gallery.js` | Product image gallery | — |
| `cart-drawer.js` | Slide-out cart drawer | — |
| `header.js` | Header scroll behavior | — |
| `header-menu.js` | Navigation & mega menus | — |
| `predictive-search.js` | Search autocomplete | — |
| `facets.js` | Collection filtering | — |
| `slideshow.js` | Carousel/slideshow | — |
| `dialog.js` | Modal dialogs | — |
| `fly-to-cart.js` | Add-to-cart animation | — |
| `quick-add.js` | Quick add modal | — |
| `paginated-list.js` | Infinite scroll / pagination | — |
| `view-transitions.js` | Page transition animations | — |
| `performance.js` | Performance monitoring | — |
| `sticky-add-to-cart.js` | Sticky product bar | — |
| `recently-viewed-products.js` | Recently viewed tracking | — |

### JavaScript Loading Strategy

Defined in `snippets/scripts.liquid` (299 lines):

1. **Import Map** — 28 modules registered with `@theme/` prefix:
   ```html
   <script type="importmap">
   {
     "imports": {
       "@theme/component": "component.js",
       "@theme/events": "events.js",
       "@theme/utilities": "utilities.js",
       "@theme/morph": "morph.js",
       ...
     }
   }
   </script>
   ```

2. **Module Preloads** — Critical modules preloaded:
   - `utilities.js`, `component.js`, `section-renderer.js`, `section-hydration.js`, `morph.js`

3. **Custom Element Registration** — Components registered as custom elements:
   ```html
   <script src="dialog.js" type="module"></script>
   <script src="product-form.js" type="module"></script>
   <!-- ... 18+ more -->
   ```

4. **Conditional Loading** — Some scripts load only on specific pages:
   - `sticky-add-to-cart.js` — Product pages only
   - `localization.js` — When localization is enabled
   - View transitions — When transition settings are enabled

### Performance Optimizations

- **Low-power device detection:** `isLowPowerDevice()` checks CPU cores ≤ 2 or RAM ≤ 2GB; disables animations on constrained devices
- **`requestIdleCallback`:** Non-critical initialization deferred to idle periods
- **`scheduler.yield()`:** Yields to main thread during long tasks
- **`IntersectionObserver`:** Lazy-loads images and defers off-screen component initialization (`section-hydration.js`)
- **View Transitions API:** Smooth page transitions without full reloads; respects `prefers-reduced-motion`
- **Request deduplication:** `SectionRenderer` prevents duplicate fetch requests for the same section
- **Debounce/throttle:** Applied to search input, scroll handlers, resize listeners

---

## 7. Configuration System

### `settings_schema.json` (68KB)

This file defines all merchant-facing theme settings in the Shopify admin. It contains **18 setting groups**:

| Group | Purpose | Key Settings |
|-------|---------|--------------|
| **Theme Info** | Theme metadata | Name (Horizon), version (3.4.0), author (Shopify) |
| **Logo & Favicon** | Brand identity | Logo image, inverse logo, desktop/mobile height, favicon |
| **Colors** | Color schemes | 6 color schemes × 30+ tokens each (background, foreground, headings, primary, hover, border, shadow, buttons, inputs, variants) |
| **Typography** | Font system | 4 font families (body, subheading, heading, accent), H1-H6 sizes, line heights, letter spacing, text transform |
| **Page Layout** | Layout options | Page width (narrow/normal/wide) |
| **Animations** | Motion settings | Page transitions, product card transitions, card hover effects (lift/shadow/zoom), reduced motion |
| **Badges** | Product badges | Sale badge style (blob/rectangle/text), sold-out badge, custom badge, badge colors |
| **Buttons** | Button styling | Border radius (primary/secondary), pill mode, sizes |
| **Cart** | Cart behavior | Cart type (drawer/page/notification), cart note, free shipping threshold |
| **Drawers** | Drawer styling | Drop shadows, border radius, drawer direction |
| **Icons** | Icon styling | Icon style (default/rounded), stroke weight |
| **Input Fields** | Form inputs | Border radius, border style, padding |
| **Popovers & Modals** | Overlay styling | Border radius, drop shadows |
| **Prices** | Price display | Sale price color, compare-at price style |
| **Product Cards** | Card settings | Image ratio, second image on hover, vendor display, badge position |
| **Search** | Search behavior | Predictive search, search scope, min characters |
| **Swatches** | Color swatches | Shape (circle/square), size, border, selected state |
| **Variant Pickers** | Variant UI | Picker style (buttons/dropdown), size options, color swatch mapping |

### Settings → CSS Pipeline

```
Merchant sets color in admin
    ↓
settings_schema.json defines the control
    ↓
settings_data.json stores the value
    ↓
theme-styles-variables.liquid reads {{ settings.* }}
    ↓
Outputs CSS custom property: --color-primary: {{ settings.primary }};
    ↓
color-schemes.liquid generates per-scheme overrides
    ↓
base.css uses: color: var(--color-primary);
    ↓
Blocks/snippets apply via class: color-{{ section.settings.color_scheme }}
```

### Color Scheme Cascade

Color schemes can be set at three levels, with inner levels overriding outer:

1. **Theme level** — Default color scheme for all pages
2. **Section level** — Each section has a `color_scheme` setting
3. **Block level** — Blocks with `inherit_color_scheme: false` can override

```liquid
<!-- Section applies scheme -->
<div class="color-{{ section.settings.color_scheme }}">
  <!-- Block inherits or overrides -->
  <div class="{% if settings.inherit_color_scheme == false %}color-{{ settings.color_scheme }}{% endif %}">
```

---

## 8. Key Patterns

### Pattern 1: Block → Snippet Delegation

Every block is a thin wrapper that captures nested content and delegates rendering to a snippet:

```liquid
{%- capture children %}
  {% content_for 'blocks' %}
{% endcapture %}

{% render 'group',
  children: children,
  settings: block.settings,
  shopify_attributes: block.shopify_attributes
%}
```

**Why:** Snippets are reusable across blocks and sections. The same `group` snippet renders both the `group` block and internal `_content` blocks.

### Pattern 2: Internal Block Composition (`_` prefix)

Internal blocks are not directly selectable by merchants. They're used within section schemas as `static` blocks:

```json
// In product.json template
"media-gallery": {
  "type": "_product-media-gallery",
  "static": true,
  "settings": { ... }
}
```

**Why:** Provides fixed structure while allowing merchant customization of settings. The product page always has a media gallery; merchants configure it but can't remove it.

### Pattern 3: Declarative Web Component Events

```html
<variant-picker-component>
  <select on:change="/handleVariantChange" ref="select">
    <option value="123">Small</option>
    <option value="456">Large</option>
  </select>
</variant-picker-component>
```

```javascript
class VariantPicker extends Component {
  handleVariantChange(event) {
    const variantId = this.refs.select.value;
    // Dispatch VariantSelectedEvent
  }
}
```

**Why:** Keeps event binding declarative in HTML, making it visible in Liquid templates without reading JavaScript.

### Pattern 4: Section Rendering API + DOM Morphing

When a variant is selected, the theme fetches updated HTML from Shopify and morphs the DOM:

```javascript
// 1. User selects a variant
dispatchEvent(new VariantSelectedEvent({ variant }));

// 2. Section renderer fetches new HTML
const html = await sectionRenderer.renderSection(sectionId);

// 3. DOM morph updates changed elements in-place
morph(currentSection, newHtml, MORPH_OPTIONS);

// 4. Components receive updatedCallback()
component.updatedCallback();  // Refs auto-update
```

**Why:** Preserves scroll position, focus state, and component instances. Much smoother than full innerHTML replacement.

### Pattern 5: CSS Custom Property Theming Pipeline

```
settings_schema.json       →  Defines UI controls
settings_data.json         →  Stores values
theme-styles-variables     →  Generates :root { --var: value }
color-schemes.liquid       →  Generates .color-scheme-N { --var: value }
base.css                   →  Uses var(--var)
section/block class        →  Applies .color-scheme-N
```

**Why:** Single source of truth for all design tokens. Changes in the admin instantly propagate through the entire theme.

### Pattern 6: Group Block as Universal Container

The `group` block is the most versatile building block:

```json
{
  "name": "t:names.group",
  "blocks": [{ "type": "@theme" }, { "type": "@app" }, { "type": "_divider" }],
  "settings": [
    // Layout: direction, alignment, gap
    // Size: width (fit/fill/custom), height, mobile width
    // Appearance: color scheme, background media, overlay, border
    // Padding: 4-directional
    // Link: URL, open in new tab
  ]
}
```

**Why:** Merchants can create arbitrary layouts by nesting groups. A group can contain text, images, buttons, other groups—enabling complex compositions without custom code.

### Pattern 7: Fluid Typography with `clamp()`

Heading sizes scale smoothly between mobile and desktop:

```css
--font-size-h1: clamp(
  calc({{ settings.heading_h1_size }}px * var(--spacing-scale-sm)),
  calc({{ settings.heading_h1_size }}px * 1vw / 14.4),
  {{ settings.heading_h1_size }}px
);
```

**Why:** No breakpoint jumps. Text scales proportionally to viewport width within defined min/max bounds.

### Pattern 8: Import Map Module System

```html
<script type="importmap">
{
  "imports": {
    "@theme/component": "{{ 'component.js' | asset_url }}",
    "@theme/events": "{{ 'events.js' | asset_url }}",
    "@theme/utilities": "{{ 'utilities.js' | asset_url }}",
    "@theme/morph": "{{ 'morph.js' | asset_url }}"
  }
}
</script>
```

```javascript
// In any component file:
import { Component } from '@theme/component';
import { ThemeEvents, CartUpdateEvent } from '@theme/events';
```

**Why:** Native ES module imports with clean `@theme/` namespacing. No bundler required. Shopify's CDN handles fingerprinting via `asset_url`.

### Pattern 9: View Transitions API

```javascript
// In view-transitions.js
document.startViewTransition(() => {
  // Navigation callback
});
```

```css
/* CSS transition definitions */
::view-transition-old(root) {
  animation: fadeOut var(--animation-speed-slow) ease;
}
::view-transition-new(root) {
  animation: fadeIn var(--animation-speed-slow) ease,
             slideInTopViewTransition var(--animation-speed-slow) ease;
}
```

**Why:** Native browser API for smooth page-to-page transitions. Gracefully degrades—older browsers get instant navigation.

### Pattern 10: Conditional Visibility in Settings (`visible_if`)

```json
{
  "type": "range",
  "id": "custom_width",
  "label": "t:settings.custom_width",
  "visible_if": "{{ block.settings.width == 'custom' }}"
}
```

**Why:** Reduces visual clutter in the theme editor. Settings only appear when relevant, preventing merchant confusion.

---

## 9. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SHOPIFY ADMIN                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  Theme Editor UI                                                     │   │
│  │  ├── settings_schema.json → Color schemes, typography, buttons...   │   │
│  │  └── settings_data.json   → Persisted merchant values               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │ {{ settings.* }}
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            LAYOUT (theme.liquid)                            │
│                                                                             │
│  <head>                                                                     │
│    ├── snippets/meta-tags          → SEO tags                              │
│    ├── snippets/stylesheets        → base.css (preload)                    │
│    ├── snippets/fonts              → Font preloading (woff2)               │
│    ├── snippets/scripts            → Import map + module preloads          │
│    ├── snippets/theme-styles-variables → :root { CSS custom properties }   │
│    └── snippets/color-schemes      → .color-scheme-N { overrides }         │
│  </head>                                                                    │
│                                                                             │
│  <body>                                                                     │
│    ├── {% sections 'header-group' %}  → Header + announcements             │
│    ├── {{ content_for_layout }}       → Template content (below)           │
│    └── {% sections 'footer-group' %}  → Footer + utilities                 │
│  </body>                                                                    │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TEMPLATES (JSON files)                              │
│                                                                             │
│  product.json ─────────┬──→ Section: product-information                   │
│                        │    ├── Block: _product-media-gallery (static)      │
│                        │    └── Block: _product-details (static)            │
│                        │        ├── Block: group (Header)                   │
│                        │        │   ├── Block: product-title                │
│                        │        │   └── Block: price                        │
│                        │        ├── Block: variant-picker                   │
│                        │        ├── Block: buy-buttons                      │
│                        │        │   ├── Block: add-to-cart                  │
│                        │        │   └── Block: accelerated-checkout         │
│                        │        └── Block: product-description              │
│                        │                                                    │
│                        └──→ Section: product-recommendations               │
│                             └── Block: _product-card (×4)                  │
│                                                                             │
│  collection.json ──────┬──→ Section: section (heading)                     │
│                        └──→ Section: main-collection                       │
│                             ├── Block: filters                             │
│                             └── Block: _product-card (×N)                  │
│                                                                             │
│  index.json ───────────┬──→ Section: hero                                  │
│                        └──→ Section: product-list                          │
│                                                                             │
│  (... 10 more templates)                                                    │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     BLOCK → SNIPPET → HTML RENDERING                        │
│                                                                             │
│  Block (.liquid)              Snippet (.liquid)              Output         │
│  ┌──────────────┐            ┌──────────────────┐          ┌──────────┐    │
│  │ capture      │            │ Receives:        │          │ <div     │    │
│  │   children   │──render──▶ │  - children      │────────▶ │  class=  │    │
│  │ endcapture   │            │  - settings      │          │  "group" │    │
│  │              │            │  - shopify_attrs  │          │  style=  │    │
│  │ {% schema %} │            │                   │          │  "..."   │    │
│  │ { settings,  │            │ Calls helpers:    │          │ >        │    │
│  │   blocks,    │            │  - spacing-style  │          │   ...    │    │
│  │   presets }  │            │  - border-override│          │ </div>   │    │
│  └──────────────┘            │  - layout-panel   │          └──────────┘    │
│                              └──────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         JAVASCRIPT ARCHITECTURE                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      Import Map (@theme/*)                          │    │
│  │  ┌──────────┐  ┌────────┐  ┌───────────┐  ┌────────┐  ┌────────┐  │    │
│  │  │component │  │ events │  │ utilities │  │ morph  │  │section-│  │    │
│  │  │  .js     │  │  .js   │  │   .js     │  │  .js   │  │renderer│  │    │
│  │  └────┬─────┘  └───┬────┘  └─────┬─────┘  └───┬────┘  └───┬────┘  │    │
│  │       │             │             │             │            │       │    │
│  │       ▼             ▼             ▼             ▼            ▼       │    │
│  │  Component    ThemeEvents    debounce()     morph()    renderSection │    │
│  │  base class   event types    throttle()    DOM diff    fetch + cache │    │
│  │  refs system  dispatch       viewTransit   preserve    deduplication │    │
│  │  on:events    bubbling       idle/yield    state       abort control │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Page-Specific Components                          │    │
│  │                                                                      │    │
│  │  product-form.js      header.js          cart-drawer.js              │    │
│  │  variant-picker.js    header-menu.js     predictive-search.js        │    │
│  │  media-gallery.js     slideshow.js       facets.js                   │    │
│  │  sticky-add-to-cart   dialog.js          paginated-list.js           │    │
│  │  fly-to-cart.js       quick-add.js       localization.js             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Event Flow:                                                                │
│    User Action → on:event attr → Component method → Custom Event →         │
│    Section Renderer → Fetch HTML → DOM Morph → updatedCallback()           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           CSS TOKEN PIPELINE                                │
│                                                                             │
│  settings_schema.json                                                       │
│       │                                                                     │
│       ▼                                                                     │
│  Merchant picks colors/fonts/sizes in admin                                 │
│       │                                                                     │
│       ▼                                                                     │
│  theme-styles-variables.liquid                                              │
│       │  Reads: {{ settings.type_body_font }}                              │
│       │  Outputs: :root { --font-body-family: "Inter", sans-serif; }       │
│       │                                                                     │
│       ▼                                                                     │
│  color-schemes.liquid                                                       │
│       │  Loops through 6 schemes                                            │
│       │  Auto-detects dark/light backgrounds                                │
│       │  Outputs: .color-scheme-1 { --color-background: #FFF; ... }        │
│       │                                                                     │
│       ▼                                                                     │
│  base.css                                                                   │
│       │  Uses: body { color: var(--color-foreground); }                     │
│       │  Uses: .btn { background: var(--color-primary-button-background); } │
│       │                                                                     │
│       ▼                                                                     │
│  Section/Block applies class="color-{{ scheme }}"                           │
│       │  Cascading override: Theme → Section → Block                        │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────┐                                │
│  │ Rendered HTML with correct colors/fonts  │                                │
│  └─────────────────────────────────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture Decisions & Tradeoffs

### Why blocks instead of sections?

Sections are used as containers, but **blocks are the primary building units**. This is because:
- Blocks can **nest inside each other** (via `content_for 'blocks'`), enabling tree-like compositions
- Blocks are **reusable across sections** — the same `text` or `group` block works everywhere
- Blocks support **`@theme` wildcard** — any public block can be dropped into any `@theme`-accepting container
- Internal blocks (`_` prefix) provide **fixed structure** while still being configurable

**Tradeoff:** More files (94 blocks + 93 snippets vs. fewer monolithic sections), but each file is focused and testable.

### Why no CSS preprocessor or framework?

- **CSS custom properties** provide the same dynamic theming that Sass variables would
- No build step means **simpler development** and **instant Shopify deployment**
- A single `base.css` file avoids HTTP request overhead of per-component stylesheets
- `{% stylesheet %}` tags in blocks/snippets provide component-scoped styles without a CSS-in-JS runtime

**Tradeoff:** The `base.css` file is large (4,900+ lines), but it's served gzipped from Shopify's CDN.

### Why vanilla Web Components?

- **Zero runtime overhead** — no framework bundle to ship
- **Native browser APIs** — custom elements, shadow DOM, mutation observers
- The `Component` base class adds just what's needed: refs, declarative events, lifecycle hooks
- **Section Rendering API compatibility** — DOM morphing works naturally with custom elements

**Tradeoff:** More boilerplate than React/Alpine for simple interactions, but the `Component` base class minimizes this.

### How does this support theme variations?

- **Color schemes** — 6 complete token sets enable wildly different color themes
- **Typography** — 4 font slots with independent sizing per heading level
- **Layout** — Page width (narrow/normal/wide), section heights, group block compositions
- **Component styling** — Border radius, button styles, card hover effects, badge shapes
- All controlled through `settings_schema.json` — no code changes needed for visual variations

### What makes this production-ready?

1. **35 language translations** with separated storefront/schema files
2. **Accessibility standards** — 20+ `.cursor/rules/*-accessibility.mdc` guides, focus management, skip-to-content, ARIA patterns
3. **Performance** — Low-power device detection, lazy loading, request deduplication, `requestIdleCallback`
4. **Progressive enhancement** — View Transitions API degrades to instant navigation, `popover` polyfill for older browsers
5. **Theme editor integration** — `visible_if` conditional settings, visual preview mode, design mode classes
6. **Structured data** — Product JSON-LD via `product | structured_data`
7. **CDN-optimized** — No build step, native ES modules with import maps, single CSS file
