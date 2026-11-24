# ADR image-decode-1: In-process bounded image decode-and-retain

Status: accepted (human security approval of the priced in-process deviation).

Mirrors Android `passes-image` / `passes-image-decode` (ADR 0005 addendum I1-I5), reshaped for a platform with no isolated child process. Sibling of `barcode-decode-1` Deviation 2, which took in-process bounded decode for barcode reads; this ADR extends that posture to decoding and retaining arbitrary user images as wallet documents.

## The lost containment claim

Android decodes every image byte inside a permission-stripped `isolatedProcess` service: a codec exploit lands in a process with no permissions or app data, and a watchdog kills it at the 5 s budget. iOS decodes in the app process. A sufficiently capable ImageIO exploit therefore lands in the app itself: app compromise, on-device data theft and, once a payment SDK ships, SDK-bounded fraud. No cap below changes where it lands, only how hard it is to deliver.

Residual risk accepted, priced for a device assumed payment-capable: a one-shot, PAC-defeating ImageIO zero-day delivered via a user-imported shared file. Mitigations: each decode is passive (no scripting environment, no attacker-paced retries) and header-capped before any allocation. Import runs one retained-lane decode (a composite adds one read-lane decode). Display then re-decodes the original through the same bounded `BoundedImageDecoder` lane each time an image document's detail surface appears (`DocumentImageViewModel` drops the pixels on disappear and re-decodes on return), so every user open is a further decode of the attacker's bytes under the same caps. Only PDFs keep originals away from display (`pdf-render-once-1`).

## Three lanes

1. **Camera:** first-party sensor pixels; structurally immune.
2. **Gallery:** PHPicker in `.compatible` mode; Photos transcodes HEIC and anything else to JPEG out of process before the app sees a byte; structurally immune.
3. **Share / file:** the only lane where attacker bytes reach ImageIO, and a surface already shipped via the barcode read path. This module bounds it.

## Allowlist: JPEG and PNG only on the retained lane

`ImageDecodeConfig.defaultAllowedContentTypes` admits JPEG and PNG only, narrower than both Android's five-container decoder allowlist and iOS's `BarcodeDecodeConfig` roster (five containers, for the read lane).

- WebP is dropped: the worst still-image CVE history of the roster (CVE-2023-4863 class), and not a format iPhone photos arrive in.
- HEIF/HEIC is not admitted natively: the one lane that would produce HEIC (gallery) transcodes out of process; a shared HEIC does not import.
- JPEG and PNG are the oldest, most-fuzzed ImageIO parsers; the public exploit record concentrates in exotic codecs, and a single passive decode offers no scripting environment.

The storage `DocumentFormat` value space keeps `webp` for Android schema parity; the importer sniff enforces it unreachable. The insert-side carrier `DocumentInsert.ImageFormat` (png/jpeg/webp) cannot carry a `pdf` label, so the PDF-only raster-lane guard's trust in the format column holds by construction on the write path.

## Caps (mirror of Android `ImageDecodeConfig`, enforced pre-allocation)

25 MB compressed bytes (read bounded at `maxBytes + 1`, never buffering the bomb); 12 000 px per side and 50 MP area from the header, before any bitmap allocates; 4 MP output ceiling (a 2048x2048 request sits exactly at it); output aspect-fitted and never upscaled; 5 s wall-clock budget. The numbers match `BarcodeDecodeConfig`; changing any is a test-breaking edit on both.

## The timeout bounds the wait, not the work

Android's watchdog kills the sandbox process at the budget. In-process iOS cannot kill anything: `withImageDecodeTimeout` releases the caller with `.decoderUnavailable` and the orphaned decode runs to completion on its lane, its result discarded. This is the largest security-property delta of the port. The header caps keep the work bounded, and the dedicated lane bank (own serial queues, never the cooperative pool, refuse-not-queue when full) keeps a wedged decode from starving anything else. It is separate from `PassesBarcode`'s banks so the image-document lane and the camera lane never share capacity.

`ImageDecodeRejectedKind.decoderUnavailable` is the timeout bucket, keeps taxonomy parity with Android, and holds the slot for a future out-of-process decoder behind the same `BoundedImageDecoder` seam.

**Lane sizing.** The bank is 4 lanes with no overflow tier (the barcode banks use 8+32): import is single-flight UX, and a refusal resolves the caller immediately. The I/O-free preflight (output bound, the `.data` arm's byte cap) runs outside the lanes so a saturated bank cannot mask those rejections. The `.fileURL` read runs inside the lane so a slow source (file provider, network volume) is bounded by the wait; under a saturated bank its over-cap rejection surfaces as `decoderUnavailable` rather than `oversizedAtImport`. Each lane can hold a decode allocation up to the ~200 MB the 50 MP cap admits, so the worst-case simultaneously live allocation is 4 x ~200 MB, about 800 MB, and an orphaned decode holds its share until it finishes. More lanes or an overflow tier would multiply that ceiling.

**Gate ordering is iOS-ahead of Android.** The shared primitive judges the container before the header-properties read, so ImageIO's metadata parser (its own CVE surface) never runs over a container the allowlist rejects. Both consumers get this ordering.

**The 25 MB bounded read is per arm.** The `.fileURL` arm reads at most `maxBytes + 1`. The `.data` arm's bytes are already resident, so its bound is the caller's: the importer must apply its own bounded read before constructing `.data`.

**The decode is forced eager.** `CGImageSourceCreateImageAtIndex` returns a lazy image by default, which would run codec work at first draw, outside any lane or deadline. The shared primitive passes `kCGImageSourceShouldCacheImmediately` so the pixel decode happens inside `decodeBounded`, and both consumers run it under their own bounded wait (the barcode facade runs its whole ImageIO-then-Vision pipeline inside `withDecodeTimeout`).

## The composite importer

`PassesDocument` (mirror of Android `passes-document`) orchestrates the sniff-and-branch import and the composite confirm seam.

**The C2 delta.** On Android, barcode extraction from an imported image runs inside the isolated barcode worker and only `{payload, format}` crosses the binder. iOS extracts in-process through the same bounded barcode read lane already shipped and priced in `barcode-decode-1` Deviation 2 (caps, lanes, five-container read allowlist). The seam shape is preserved: the importer's internal `BarcodeExtraction` never threads a raw decode result past it; only the distilled pair and the payload-free `BarcodeExtractionOutcome` cross.

**Two-decode accounting.** A composite import runs two bounded decodes of the same once-read bytes: the retained-lane decode (display raster and thumbnail, JPEG/PNG only) and the barcode read-lane decode (five-container). Android also decodes twice, once per sandbox, so the delta is only in-process versus sandboxed. Extraction runs only when the consumer supplies `confirmBarcode`, and the decoded payload crosses to the app pre-persist only through that hook (and, post-confirm, `DocumentPersist.barcodedImage`). `webp` is recognized at the sniff and rejected before any codec contact.

Two iOS-ahead hardenings on the source read:

- An over-cap source is rejected at the importer from the truncated prefix's sniff. Android delegates oversize to its sandbox caps, which would persist truncated bytes as the original if a backend cap were raised above the read ceiling.
- The `.fileURL` read runs off the cooperative pool under a bounded wait (`withSourceReadDeadline`). Reader threads are capped (`SourceReadSlots`, 4) with refuse-past-cap, because a wedged provider read parks its thread for the process lifetime; the cap is a security bound like the decode banks'. A FIFO or device node is rejected at open (`O_NONBLOCK` + `S_ISREG`) and never holds a slot. Non-`file` URL schemes are refused (Android's `content://` allowlist analogue); acquiring security-scoped access to a picker URL is the caller's job.

## Structure mapping (Android module to iOS target)

- `passes-image-decode` to `PassesImageDecode`: mechanism-only header-gated decode, generic over the caller's rejection type, no dependencies, below the `PassesBarcode` / `PassesImage` peers without an edge between them. Android folds `IOException` / `IllegalArgumentException` / `RuntimeException` and optionally `OutOfMemoryError`; ImageIO reports failure as nil and Swift has no catchable allocation failure, so the iOS fold surface is exactly the nil-returns.
- `passes-image` to `PassesImage`: policy (config, allowlist, taxonomy, facade). The binder, service, client and wire plumbing has no iOS analogue.
- `passes-document` to `PassesDocument`: sniff-and-branch import and the composite confirm seam; the single meeting point of the PDF, image and barcode peers, which gain no edge to each other.
- `ImageSource`'s no-byte-array rule is not mirrored: Android forbids a `ByteArray` arm because bytes in the caller's heap would defeat its sandbox; in-process there is none to defeat, and the importer reads its source once into bytes before sniffing, so `ImageDecodeSource.data` is the primary arm. The closed two-arm discipline (no path-string arm) is kept.

## Sealed `Document`

`Document` / `DocumentId` are protocols with a closed arm set pinned by `documentArms` + `DocumentSealedSetTests` (Swift has no sealed protocols). `ImageDocument` carries `widthPx` / `heightPx` of the bounded raster, never a re-decoded canvas, and no container format (a persistence detail). `BarcodedImageDocument` is the composite arm, as on Android.
