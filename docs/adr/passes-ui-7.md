# passes-ui-7: Colour carries no trust meaning; the wallet-row concession is withdrawn

iOS anchor for Android's colour-system revisions to `SCANNABLE_CARD_THREAT_MODEL.md` (canonical in the Android kernel repo; iOS sources cite it by name). `passes-ui-6` anchors the list-surface and trust-boundary rows.

## Colour carries no trust meaning, for any class

The consumer rules, verbatim from the Android row: colour means class, nothing else; the pass file's `backgroundColor` is read, never rendered; card stock is identical in light and dark; no previews on list cards; expired washes out instead of tinting; any item is user-reassignable; no card presents as verified, and signature is stated in words, on detail.

So colour is decoupled from the issuer (every pkpass renders its class tint regardless of `backgroundColor`), applies to every artifact class (documents and scannables carry a tint too), and is user-reassignable per item from a seven-colour palette.

**The verified-band confusion class (C1/C2) stays closed:**

1. No surface presents as verified. No list card of any class renders a signature affordance, and no card face carries one at detail scale, so there is no verified visual for a user-created card to imitate.
2. Signature status is stated in words, on the detail surface only. Text, not chrome, so a colour choice cannot imitate it.
3. Colour is uniform per class by default and user-reassignable. A Bronze PDF or a Teal pkpass is a user preference, not a spoof, and nothing may be inferred from it. Colour any user can reassign cannot encode provenance even accidentally.

**Kernel surface.** `ScannableCardScreen(faceTint:)` and `DocumentView(faceTint:)` let the consumer tint the card face without reimplementing these surfaces. Both are presentation-only:

- The tint reaches the card face only; the panel behind a code stays literally white in both themes and the rasterised page renders identically tinted or not.
- Neither parameter can suppress the barcode, the payload readback or the trust caption (the C2 note): `faceTint` is not a second route to the C2 bypass the `HostedTypeRow` concession audits. Pinned by `ScannableCardFaceTintTests`.
- Ink on a tinted face derives from the tint's luminance at >= 4.5:1, so the in-words provenance signal cannot be tuned away by a hostile or careless tint.
- The kernel stores no colour. No `ScannableCard` or `Document` model or storage column carries one; which colour an item wears is consumer state.
- Both arms decide "is this a tint" through the shared `faceIsTinted` predicate (`PassesUICore.FaceTint`, pinned by `FaceTintTests`), so a fully transparent tint falls back to the untinted default on either arm.

**Out of bounds without amending this record:** deriving a colour from signature status, verification outcome or issuer identity anywhere in the kernel; re-adding a colour field to a kernel artifact model or storage table; a `faceTint` reaching the code panel or the page render.

## Wallet-row concession: withdrawn

Android's C1/C2 wallet-row concession permitted `ScannableCardRowTile`, a flat label-led row for hosts interleaving scannable cards with passes and PDFs in one list, under three conditions: no signature affordance on the row, no leading strip styled as a verified-pass band, and a detail surface keeping the non-suppressible caption. The tile is deleted from `PassesUI` (with its `leadingSlot` hook) because the surface it permitted no longer exists, and a trust-reasoned surface nothing ships invites adoption on the strength of an argument for a list shape that is gone.

The withdrawal is not a finding that the row shape was unsafe, and not a statement about colour. Reintroducing a homogeneous row register, kernel-side or by a consumer, amends this record: it must re-argue conditions 1 and 3 from the current premises. `ScannableCardTile` and its four-distinguisher contract remain the kernel's surface for hosts presenting scannable cards in their own lane.

## List-face code render concession: dormant

The consumer no longer renders a code on any list face, so the `CompactCodeView` list-face concession is not exercised. It is not withdrawn: a host may render an extracted code at list scale again. Condition 1 (mechanism-only render, pinned by the `CompactCodeView` construction lock) and condition 3 (detail-surface provenance unchanged) govern for such a host; condition 2's "neutral card surface" is replaced by the class tint.

## Document lane (Android ADR 0005 D1.C / D1.L mirrors)

- **D1.C:** documents take a class tint like every other artifact class; the class distinction is carried structurally (type glyph, eyebrow, words-on-detail trust posture), not by the absence of colour.
- **D1.L:** consumers may render documents inline in a unified wallet list, provided documents never masquerade as passes in the model, the row carries no verified affordance, and detail-surface provenance is unchanged.
