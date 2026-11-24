# passes-pdf-ui-4: Full-screen zoom is a `scaleEffect` over the stored raster; no sub-rect re-render

Status: accepted.

Android's `FullScreenDocumentView` fires a `renderer.render(SubRect)` call on pinch settle and swaps in the sharper bitmap within the 4 MP per-bitmap cap. iOS does not: the display layer holds no renderer (see `pdf-render-once-1`), and pinch-zoom scales the stored 4 MP raster bilinearly through SwiftUI `scaleEffect`.

The gesture surface (pinch, pan, double-tap) lives in the shared `Internal/ZoomableContent` primitive, which both the PDF pager pages and the image/composite arm compose. It clamps pan to ±((scale-1) × slot/2) and clips unscaled to the slot frame.

Trust posture is unaffected: sub-rect rendering is a sharpness optimisation, not a trust control.

Android source: `passes-android-main/passes-document-ui/src/main/kotlin/is/walt/passes/document/ui/internal/ZoomableImage.kt`, `.../document/ui/FullScreenDocumentView.kt`.
