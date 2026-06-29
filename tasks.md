# M2M Variant Migration — Task Checklist (Reima)

Working to-do for the pseudo-product → per-type-variant migration (split shipping + reporting). Ordered by **execution sequence**, not by workstream letter. Tick as you go.

**Legend:** ⭐ = part of the ~10-day focused-dev MVP · ⚠️ = critical gotcha / ordering constraint · 🔬 = decision or spike to settle first · `(Nd)` = est. dev-days

**Two non-negotiables:** (1) ship **E1** *before* creating any variants; (2) keep the price hash in **3-way lockstep** (route ↔ theme JS ↔ Function) — never touch the secret.

Full rationale + file/line detail: see [m2m-variant-migration-spec.prompt.md](m2m-variant-migration-spec.prompt.md) §9.

---

## Phase 0 — Decisions & spikes (do first; these unblock everything)

- [ ] 🔬 Confirm assumptions: M2M lead time + delivery-profile name(s); approx. product count; which M2M types go live first (all 11 vs curtains/roller/roman/cushion/sample); option name + value labels; SKU suffix convention; target start date.
- [ ] 🔬 **Blank-`suitability` behaviour:** create-all types vs skip-and-flag for manual review. (Theme shows all when blank — but auto-creating 11 variants for every blank product is likely wrong.)
- [ ] 🔬 **Admin-order scope:** do admin-created orders also need to split-ship + report by variant? If no → drop Workstream D (~8d). If yes → do the spike below.
- [ ] 🔬 **API spike (gates Workstream D):** does a *variant-referenced* draft-order line honour a unit-price override (`originalUnitPriceWithCurrency` / `priceOverride`) on the live API version? If not → admin lines stay custom, split/reporting = storefront-only.
- [ ] 🔬 Confirm ASR rule strategy per new profile (which rates attach to the made-to-order profile vs the standard profile).

---

## Phase 1 — Safety net & inert plumbing (ship first; low risk, reversible)

- [ ] ⭐⚠️ **E1 (1.5d)** Fix `getProductStockInfo` `variants(first:1)` → `variants(first:50)` + deterministic meterage-variant selection (option `Fabric` / SKU `-M2M-Fabric`), in `app/lib/inventory.server.ts`. **MUST be live before A2 creates any variant** — otherwise stock corrupts on every order incl. old carts.
- [ ] ⭐ **E2 (1d)** Make the non-M2M deduction path variant-aware (`webhooks.orders.create.tsx`) — an M2M variant sold without `_m2m_stock` must not deduct metres.
- [ ] **E3 (1d)** Guard/contract-test that `_m2m_stock` stays anchored to the fabric product/variant.
- [ ] ⭐ **C1 (1.5d)** Add `body.m2mVariantId` + hash against it across all 11 `app/routes/api.storefront.*-price.tsx`; extract shared `resolveHashVariantId()` (fallback `m2mVariantId ?? fabric.variantId`). Do **not** touch `apiFabricRef`/`_m2m_stock`. No algo change to `price-hash.server.ts` or the Function. (Inert until the theme sends the new id.)
- [ ] **C2 (0.5d)** Saved-order/quote replay audit (`api.storefront.saved-order*.tsx`): re-price on replay or store `m2mVariantId`.
- [ ] ⭐ **F1 (0.5d)** Hash-parity test: server == Function == JS for an M2M-variant id (extend `extensions/m2m-price-transform/tests/default.test.js`).
- [ ] ⭐ **F2 (0.5d)** Pricing snapshot test: identical config pre/post → identical `total`.
- [ ] **F3 (1.5d)** Inventory regression matrix (old `_m2m_stock` once; new variant-line once; M2M variant w/o stock not deducted; mixed order no double-count).

---

## Phase 2 — Catalog migration & Shopify config (the big ops effort)

- [ ] **A1 (1d)** Define the `M2M` option + SKU convention `<base>-M2M-<Type>`; document in `CLAUDE.md`. The `Fabric` value **reuses the original variant id/SKU/price/inventory** — never delete-and-recreate (⚠️ R5).
- [ ] **Shared module** — extract suitability→types mapping (reads metaobjects `create_curtains_blinds_more` + `create_furniture` for `suitability_key`/`config`, intersected with product `custom.suitability`). Reused by A2, A5, (and admin if D is in scope). Do **not** hardcode the type list.
- [ ] **A2 (4–5d)** Build `scripts/migrate-m2m-variants.ts`: page `product_type:Fabric`; per product create only its suitable-type variants via `productOptionsCreate` + `productVariantsBulkCreate`; tracking **off** / policy **CONTINUE**. Idempotent (key on SKU), resumable/checkpointed, throttle-aware, **dry-run** mode, per-product log + re-runnable failure list.
- [ ] ⚠️ **A2 dry-run** on a handful of fabrics; verify variants/SKUs/inventory-policy before scaling.
- [ ] **A2 full run** at scale (after E1 is live and dry-run verified).
- [ ] **Profiles** — create the delivery-profile shells (e.g. "Made-to-order — 10–12 weeks" + standard/by-the-metre). Manual in Settings *or* scripted via `deliveryProfileCreate` (flat/weight rates easy to script; ASR/carrier rates easier in the UI).
- [ ] **A4 (3d)** Build + run `scripts/migrate-m2m-delivery-profiles.ts`: `deliveryProfileUpdate` `variantsToAssociate`, batched ≤250 gids/call, idempotent on SKU suffix, throttle-retry. (Assignment must be scripted — manual is impossible at scale.)
- [ ] **Reconciliation report** — suitability-vs-variant mismatches, orphaned configurators, SKU-suffix audit.

---

## Phase 3 — Storefront cutover

- [ ] ⭐ **B1 (4d)** Replace the demo with production variant resolution across all 11 `snippets/*-configurator.liquid` + `product-form-fabric.liquid` + `src/js/app/components/curtain-configurator.js`; write resolved M2M variant id to both `input.product-variant-id[name="id"]` and `dataset.fabricVariantId`; disable a configurator if its M2M variant is missing.
- [ ] ⭐ **B2** Runner special-case: re-anchor `data-runner-variant-id` pricing/hash path too.
- [ ] ⭐ **B3 (1d)** Send `m2mVariantId` to pricing routes but keep per-metre price sourced from the **fabric** variant metafields (so `total` is unchanged → hash stays valid).
- [ ] Rebuild theme bundle (`pnpm run vite:build:app`); verify hash agreement on **staging**.
- [ ] ⭐ **F4 (2d)** E2E on dev store: add-to-cart → checkout **splits into 2 delivery groups** with distinct delivery promises; ASR + native rates correct; draft create→complete works. (Use the existing demo picker to drive the 11 types first.)
- [ ] ⚠️ Remove the demo picker (`snippets/configurator-variant-picker-demo.liquid` + the 11 render lines — `git grep configurator-variant-picker-demo`) and **flip production**.

> **MVP cutover (~10 focused dev-days) = all ⭐ tasks above** (E1, E2, C1, B1, B2, B3, F1, F2, F4) — delivers split shipping + reporting + hash integrity + no stock corruption. Phase 2 catalog/profile work is a required *enabler* (mostly ops/parallel), and Phases 4–5 trail the cutover.

---

## Phase 4 — Admin-order parity *(only if Phase 0 decision = yes; gated on the API spike)*

- [ ] **D1 (0.5d)** Add optional `m2mVariantId` (+ sku/productId) to `M2MItemDetails` (`order-session.ts`), fallback to `fabric.variantId`.
- [ ] **D2 (2.5d)** `resolveM2MVariant()` + thread through all 11 `create*M2MItemFromForm` builders; populate from ProductPicker/metafield fetch.
- [ ] **D3 (2.5d)** Anchor admin draft lines to the M2M variant in `buildDraftOrderLineItems` (`order.server.ts`) — per the spike outcome; add `_m2m_hash` to admin lines; fallback to custom line when unresolved (feature-flag).
- [ ] **D4 (1d)** Edit-order round-trip preserves `m2mVariantId` (`app.edit-order*.tsx`, `order.server.ts`).
- [ ] **D5 (0.5d)** Re-anchor admin saved-quote hash (`app.create-order.tsx`).
- [ ] **D6 (1d)** Back-compat normalizer tolerates missing `m2mVariantId` (`order-session.ts`).

---

## Phase 5 — Going-forward automation

- [ ] **A5 (3d)** `products/create` + `products/update` (suitability-changed) webhooks (`app/routes/webhooks.products.create.tsx` + `webhooks.products.update.tsx`) re-applying the suitability rule to add missing variants + assign profile. **Add-only** (removal → flag for manual review, never auto-delete). Reuse the shared module.
- [ ] Reconciliation report after A5 is live.

---

## Phase 6 — Deploy & close-out

- [ ] **F5 (1d)** Synchronised deploy + rollback runbook: rebuild Function `dist/function.js` + theme `assets/app.js`; confirm secret parity; document deploy ordering (C1 → E1–E3 → A2/A4 → B → flip → D/A5) and rollback (revert theme bundle to fabric-variant `name="id"`; routes still accept old payloads via fallback; Function unchanged).
- [ ] Acceptance criteria verified: fabric + M2M variant in one cart → 2 delivery groups w/ distinct promises; ShopifyQL returns correct per-type units sold; pricing unchanged; no stock drift.
- [ ] Update `CLAUDE.md` (new variant/SKU model, configurator behaviour) + project memory.

---

### Quick progress view
- Phase 0 (decisions/spikes): 0/5
- Phase 1 (safety net): 0/8
- Phase 2 (catalog/config): 0/8
- Phase 3 (storefront cutover): 0/7
- Phase 4 (admin parity, optional): 0/6
- Phase 5 (automation): 0/2
- Phase 6 (deploy/close-out): 0/3
