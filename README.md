# Products — NonprofitCRM plugin

The Products domain vertical for
[NonprofitCRM](https://github.com/abeuscher/npc-beta): the public product
checkout and waitlist endpoints (`ProductCheckoutController` +
`ProductWaitlistController` with the throttled `products.checkout` /
`products.waitlist` POST routes), the ProductDisplay and ProductCarousel
widgets, the Product admin resource (three Filament pages plus the price
and waitlist relation managers), both product observers, the product
policy, and the `RecordProductPurchase` checkout-fulfillment listener.
Extracted to its own repository in session 398 (Plugin Architecture arc,
Stage D — the ninth extracted plugin).

**Package:** `nonprofitcrm/products`
**Implements plugin contract:** `0.22.0` (`docs/plugin-contract.md` in the core repo)

## How this package is consumed

The core repo's `composer.json` carries a `vcs` repository entry pointing at
this repo and a bound version requirement. Composer resolves **git tags** as
versions — this package deliberately declares no `version` field in its
`composer.json` (a retained field that disagrees with the tag is a composer
install error; under a VCS repository the tag is canonical).

Activation stays with the core repo: `config/plugins.php` (generated from
`distribution.json`) lists `Plugins\Products\ProductsServiceProvider`, and
the per-install `PLUGINS_DISABLED=products` flag vanishes the plugin at
runtime. Nothing about activation is keyed to where this code lives.

## Schema

This package owns its schema (contract surface 5): `database/migrations/`
creates the four product tables in FK order (`products` → `product_prices`
→ `purchases` → `waitlist_entries`). Install order is core schema dump →
enabled plugins' migrations → seeders; disabling the plugin never drops its
tables or data (disabled ≠ uninstalled). All four PKs are uuid — **no
sequences travel** (the arc's second no-sequences redraw). The two RESTRICT
FKs on `purchases` (`product_id`, `product_price_id`) are behavior — a
product with purchases cannot be deleted; the plugin-internal
`product_prices.product_id` and `waitlist_entries.product_id` FKs are
CASCADE; the two `contact_id` FKs into core `contacts` are `ON DELETE SET
NULL` (a contact force-delete orphans the row rather than blocking). The
four product models themselves stay in core (plan § 6.7), so core code that
reads these tables gates on route presence
(`Route::has('products.checkout')`).

## Composition notes

This plugin **publishes no capability** — nothing consumes products
presence. The presence signal is route presence. The checkout controller
*consumes* the `payments` capability (contract surface 13): payments-absent,
checkout answers its not-configured error and the waitlist path runs
untouched. Purchase fulfillment arrives through core's `CheckoutSettled`
event, dispatched by the Payments plugin's webhook (`crm-plugin--payments`
v0.4.0 — the arc's third cross-repo inversion); the listener self-filters on
the `product_price_id` metadata key before any read, and products-absent the
dispatch has no listener (200, no purchase row, data kept).

ProductCarousel declares vendor-relative front-end assets
(`vendor/nonprofitcrm/products/Widgets/ProductCarousel/{styles.scss,script.js}`
plus the `swiper` lib) — built into the public widget bundle by the core
build. ProductDisplay declares none. Widget thumbnails ship in this repo
(the core thumbnail generator cannot reach an extracted plugin — regenerate
here when visuals change).

## Tests

The `tests/` directory ships with this package and is the phpunit
`Products` testsuite in the core repo, which runs it from
`vendor/nonprofitcrm/products/tests` — the central build runs the
distribution's plugin suites (plugin contract, normative rule 5). No CI
here yet (per-plugin CI is a scheduled later arc position).

## Releases

- **v0.1.0** — extracted from the core repo (session 398, arc D8). Git
  history predating the extraction lives in the core repo.
