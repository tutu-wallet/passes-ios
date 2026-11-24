# passes-ui-6: List-surface posture and the trust-presentation boundary

The consumer wallet list is a stack of large card faces rendering kernel-produced content inline. This ADR is the iOS anchor for the list-surface rows of Android's `SCANNABLE_CARD_THREAT_MODEL.md` (canonical in the Android kernel repo; iOS sources cite it by name) and records where the trust-presentation boundary sits. Colour rules are in `passes-ui-7`.

## List-surface posture

- **Kernel content renders only through guarded entry points.** A PDF card's page-0 thumbnail comes from the stored-raster path (`passes-pdf-ui-6`, `pdf-render-once-1`); a scannable card's code comes from the encoder-only `CompactCodeView` path (no decode lane; its contract is in that view's doc comment). Consumer card faces must route all kernel-content rendering through these paths: decoding issuer bytes or user images outside the kernel's bounded entry points (`BoundedImage` for pass images, the caller-supplied `BoundedImageDecoder` for image documents, stored first-party rasters for PDFs) is a contract violation, not a styling choice. The current consumer renders no previews on list cards, but the rule binds any host that does.
- **No list card of any class carries a signature or verified affordance.** The trust ladder (signature status, trust captions, provenance) lives only on detail surfaces, so the verified-band confusion class is structurally absent at list scale: no verified visual exists for a user-created card to imitate. Consumers are expected to pin this in their own list tests, mirroring Android's `WalletListTest` invariant.

## Trust-presentation boundary

A consumer detail that composes kernel primitives (`BarcodeView`) inside its own ticket card instead of mounting `PassFront` takes over two signals:

- **Kernel-guaranteed (non-suppressible) only where the kernel view is mounted:** `ExpiredOverlay` inside `PassFront`, and the provenance caption inside `ScannableCardScreen` / `ScannableCardTile`.
- **Consumer-owned when composing primitives directly:** a trust caption stating signature status verbatim from the kernel's `SignatureStatus` classification (never a stronger claim than the classification supports; a self-signed pass is signed), and an expired treatment derived from the pass's own dates. The kernel does not offer a suppressible variant of its badged views; a consumer that wants kernel-guaranteed presentation mounts the kernel view.
