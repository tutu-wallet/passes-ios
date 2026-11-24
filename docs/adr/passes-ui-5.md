# passes-ui-5: `UiTelemetryGuard.onImportRejected` and `PassImportRejectionSheet`

Status: superseded. Both now ship: `ParseFailureKind` is ported into `PassesCore`
(`Sources/PassesCore/ParseFailureKind.swift`), and the rejection sheet and its
`onImportRejected(kind:)` telemetry live in `Sources/PassesUI/PassImportRejection.swift`.

The original decision deferred both until `ParseFailureKind` was ported, since the sheet's per-arm copy
and the telemetry hook are keyed on it.

Android source: `passes-android-main/passes-ui/src/main/kotlin/is/walt/passes/ui/PassImportRejection.kt`.
