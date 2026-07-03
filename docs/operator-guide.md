# Operator Guide

## First Deployment
1. Register store, SKUs, suppliers and operators.
2. Import opening inventory and barcode catalog.
3. Run read-only EAN-13 validation against the catalog.
4. Configure reorder thresholds and escalation.
5. Publish a dry-run reconciliation and audit export.

## Day in the Life: A Restock at a Corner Grocery

`cloud-itonami-4711` is Community Retail Operations — this walkthrough
follows one real occupation task on the floor: restocking a shelf after a
supplier delivery, run through the intake → propose → approve → execute →
audit loop that `:retail-governor` enforces on every SKU movement.

1. **Intake.** A delivery of canned tomatoes arrives from the supplier. The
   clerk logs it through the `:forms` intake surface: SKU, EAN-13 barcode,
   quantity, supplier invoice price. This creates the reorder/restock record
   `:retail-governor` will later gate.
2. **Propose.** The clerk picks up a case of the delivered stock and walks it
   toward the shelf that's running low — the shelf is now a proposed restock
   target, but nothing has been priced or committed yet.
3. **Approve (the governor gate).** Before touching the shelf, the clerk
   stops at the stockroom counter and pulls the stock record for this
   round's pricing sign-off. `:retail-governor` runs its `:dmn` checks —
   EAN-13 validation on the barcode, unit price against the catalog's price
   band, reorder quantity against the threshold-and-lead-time rule — and
   either signs off or rejects (e.g. the invoice price doesn't match the
   catalog, or the barcode fails checksum). Only a *signed-off* pull is good
   for restocking one shelf; it is never a standing permit for the whole
   shift.
4. **Execute.** With sign-off in hand, the clerk restocks the shelf: the
   cans go up priced and sellable, inventory count updates, the shelf is
   cleared.
5. **Audit.** The sign-off and the restock event are both written to the
   `:audit-ledger` immutable log before the SKU is considered committed —
   this is the record a supplier, customer, or regulator could later
   inspect to confirm the price on the shelf was signed off, not guessed.

**What happens if the clerk skips step 3** (restocks straight from the
delivery without pulling the stockroom sign-off, e.g. to save time during a
rush): the shelf goes up *unpriced*. `:retail-governor` logs this as a
violation rather than letting the sale happen silently — an unpriced shelf
cannot be legally sold from, and a store repeatedly skipping sign-off fails
certification (see below).

**Rush-hour pricing.** When a high-demand item runs out during a lunch rush
(bread, milk — the `food-security` case this blueprint exists for), that
restock is worth acting on fast, but the *same* stockroom sign-off applies —
time pressure is not a waiver. There is no fast lane around
`:retail-governor`.

**Feel the loop hands-on.** `network-isekai`'s playable prototype "ITONAMI:
Retail Shift" (`public/games/itonami/retail-shift`) turns this exact loop
into a short depot round: touch the "stockroom" for the `:retail-governor`
pricing sign-off, then clear "shelf" jobs around it; an occasional
"rush-hour-shelf" is worth 3x but needs the same sign-off; skipping the
sign-off and restocking anyway leaves a shelf unpriced and costs a life.
Clearing all 8 shelves closes the shift clean. It's a fast way for a new
operator (or a developer onboarding onto this blueprint) to feel why the
approve-before-execute gate exists before running it against a real store.

## Minimum Production Controls
- EAN-13 validation before every sale line
- inventory delta integrity for every receipt
- daily reconciliation gate before close
- audit export for every void and reorder
- backup manual sale and cash-up process

## Certification
Certified operators must prove POS integrity, evidence-backed reconciliation
and human review for void/reorder-affecting actions.
