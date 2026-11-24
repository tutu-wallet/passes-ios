# passes-iOS

Walt Passes for iOS: PDF pass parser, encrypted storage, secure rendering.

This is the iOS counterpart of
[`passes-android`](https://github.com/tutu-wallet/passes-android) and is
consumed by [`walt-app/ios`](https://github.com/walt-app/ios) as a Swift
Package dependency.

The two kernels mirror each other's *contracts*: the same result shapes, the
same trust-claim surface, the same faithful-payload posture. They are allowed to
diverge where the platforms' process models genuinely differ, and where they do,
the ADR for that area records why. `DecodeFailureReason.decodeTimedOut` shows the
difference: Android kills its isolated decode process at the budget, while iOS
cannot kill Vision mid-decode, so its timeout bounds only the caller's wait (see
[`docs/adr/barcode-decode-1.md`](docs/adr/barcode-decode-1.md)).

## Modules

Pure-logic targets are split from their SwiftUI counterparts (`*Core` vs. the
UI target) so the logic stays testable without a UI host.

| Target | Purpose |
|---|---|
| `PassesCore` | Domain types, the `PassParser` trust-claim surface, and pkpass signature verification (swift-certificates) |
| `PassesBarcode` | Bounded barcode decode (Vision) over still images and camera frames |
| `PassesImageDecode` | Shared header-gated bounded decode mechanism (no public product) |
| `PassesImage` | In-process bounded image decode-and-retain (`image-decode-1`) |
| `PassesDocument` | Sniff-and-branch document import + composite confirm seam (`image-decode-1`) |
| `PassesPDFCore` | Pure PDF parsing/validation and the sealed `Document` model family |
| `PassesPDF` | PDF import and bounded rendering |
| `PassesPDFUI` | SwiftUI document views |
| `PassesStorage` | GRDB-backed, device-only encrypted persistence (iOS Data Protection) |
| `PassesUICore` | UI identity primitives and pass-display logic |
| `PassesUI` | SwiftUI pass / scannable-card views |

## Security

See [`SECURITY.md`](SECURITY.md) for the trust-claim surface every
implementation must uphold and how to report a vulnerability.

## Build & test

```bash
swift build
swift test
```

iOS-specific code paths build via `xcodebuild` against an iOS simulator.

## License

Apache 2.0. See [`LICENSE`](LICENSE).
