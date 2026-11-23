# passes-ui-2: EAN-13 / UPC-A / Code39 render through hand-rolled 1D encoders; QR ships UTF-8 without ECI

Apple ships no CoreImage generator for EAN-13, UPC-A or Code39, so `PassesCore.OneDimensionalBarcodeEncoder` encodes them into single-row `BarcodeMatrix` values from the GS1 / AIM module patterns, with no new dependency. Shape matches what Android's ZXing renders: Code39 wide:narrow 2:1, UPC-A as the leading-zero EAN-13 symbol, quiet zones included. `BarcodeRenderer` rasterizes the row with nearest-neighbor upscale so modules stay crisp, and `CompactCodeView`'s 1D arm routes through the same encoder.

## One encode entry point

`PassesCore.BarcodeEncoder` is the single encode entry for every format. The 1D trio goes through `OneDimensionalBarcodeEncoder`; QR, Code128, PDF417 and Aztec go through the CoreImage generators (pinned error correction and the Latin-1 charset posture are in `barcode-decode-1`). `BarcodeRenderer.cgImage(payload:format:)`, `CompactCodeView.renderImage` and the trial-encode gate on the storage layer's scannable-card write paths all call it, so save-time approval and draw-time render judge encodability with the same code and cannot diverge.

The encoder re-validates structure (charset, exact length, check digit via `ScannableFormatConstraints`) and returns `nil` on failure. It refuses an empty payload for every format, so an empty QR renders the failure tile rather than the empty-but-scannable symbol `CIQRCodeGenerator` emits; a persisted row is never empty, so this is reachable only with a caller-constructed in-memory card.

Failure visuals: the 1D trio degrades to a 1x1 grey placeholder on detail surfaces; QR / Code128 and the compact path show their failure tiles. Rows on disk that clear the validator but fail the encode (the CJK-density case) are not backfilled: they show the failure tile and stay editable to a valid payload.

Correctness is pinned by structural table tests plus a Vision decode round-trip per symbology (`OneDimensionalBarcodeEncoderTests`, `OneDRoundTripTests`).

## QR ships UTF-8 without an ECI header

Android pins ZXing's QR character set to UTF-8 with an ECI header because ZXing otherwise transliterated non-Latin-1 payloads. `CIQRCodeGenerator` always encodes raw UTF-8 bytes and exposes no ECI control, and hand-rolling a QR encoder only to emit the header was rejected. The de-facto mobile convention, UTF-8 without ECI, is accepted.

- `QrCharsetRoundTripTests` round-trips Android's original defect payload ("café - naïve - 東京"), CJK-only and supplementary-plane payloads verbatim through `BarcodeEncoder` and Vision.
- `qrByteModeCeilingIsExactAtTheBoundary` pins the absence of an ECI header: its 2331-byte success arm fails if the generator ever spends capacity on one (Vision decodes both forms alike, so the round-trip suite cannot detect it). The v40-M byte-mode ceiling is therefore 2331 on iOS where Android's header lowers it to 2330.
- Residual accepted risk: a strictly spec-conformant reader defaults ECI-less symbols to Latin-1 and shows mojibake for non-ASCII payloads. ASCII is unambiguous everywhere.

Android source: `passes-android-main/passes-ui/src/main/kotlin/is/walt/passes/ui/ScannableCardView.kt`.
