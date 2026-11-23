# passes-pdf-ui-5: `Data`-backed pixel buffers replace Android `SharedMemory` / `ParcelFileDescriptor`

Android consumes pages through an IPC binder returning a `SharedMemory` handle, then copies pixels into a `Bitmap` and closes the region. iOS has no isolated-process renderer, so pages reach the display layer as stored first-party rasters through `DocumentPageSource` and are decoded in `Sources/PassesPDFUI/Internal/PageRendering.swift` (`decodeStoredRaster`). Buffers are plain `Data` values with ordinary Swift value semantics, and `pdfData` is passed by value rather than as a `ParcelFileDescriptor`.

The `consumerRenderFailureFor` dispatch table keeps Android's marker error types (`OutOfMemoryError`, `DimensionMismatchError`, `SharedMemoryUnavailableError`), so the failure-classification contract stays exhaustively pinned by tests.

Android source: `passes-android-main/passes-pdf-ui/src/main/kotlin/is/walt/passes/pdf/ui/internal/PageRendering.kt`.
