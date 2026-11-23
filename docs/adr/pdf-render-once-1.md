# pdf-render-once-1: Import-time page rasterisation; display never re-parses originals

Status: accepted. iOS-only deviation from Android.

## Context

The iOS kernel parses untrusted PDFs in-process with PDFKit/CoreGraphics (`data-passes-1` records the deviation from Android's isolated `:pdfRenderer`). PDF is the compound, compute-bearing format of the roster (JBIG2/JPX filters, embedded fonts; the FORCEDENTRY class), and the iOS `ProcessKiller` behind the render watchdog is a no-op. Re-parsing the original untrusted bytes on every detail open made display the worst-contained untrusted-parse surface, so display must never touch the originals.

## Decision

**Render once, retain rasters.** At import, `DefaultPDFImporter` rasterises every page (at most `maxPages` = 10) via `PDFRendererBinder.renderFitted`: aspect-correct dimensions fitted within the 4 MP budget (`pageRasterMaxPixels` = `PDFKitRenderer.maxPixels`), PNG-encoded, then handed to `persist`. `renderAllFitted` opens the `PDFDocument` once for the whole pass, so import performs a bounded handful of parses (probe, the page-zero thumbnail, one batch pass), never one per page and never any on display. Any page failing to render or encode rejects the whole import, since a partially rasterised document would reintroduce re-parsing. Storage writes the set in the same transaction as the document row (`document_page_rasters`, cascade delete), re-checking page count and per-raster pixel bound as `pageRastersInvalidAtStorage`.

**Display consumes stored rasters only.** `DocumentView` / `FullScreenDocumentView` take a `DocumentPageSource` (PassesPDFCore) instead of `pdfData` plus a renderer; `PDFThumbnailViewModel` decodes the stored first-party PNG. Enforced twice: `PassesPDFUI` has no `PassesPDF` dependency (the renderer seam does not resolve at compile time), and `RenderOnceGuardTests` source-scans the module for `import PDFKit`, `PDFDocument(data`, `PDFRendererBinder` and `CGPDFDocument`.

**One sanctioned fallback.** `RerenderOnMissPageSource` (PassesPDF) serves a store miss (a pre-v6 document or a lost blob) with exactly one bounded re-render of the originals per miss, persists the backfill through the per-page `insertDocumentPageRaster`, and never retries in a loop. Concurrent misses coalesce per page and serialize across pages, so at most one parse of the originals runs at a time; render failures report `onConsumerRenderFailed`. In steady state the untrusted bytes meet a PDF parser only at import, and a legacy document converges page by page as viewed.

The importer's page-zero 600x800 thumbnail leg is unchanged (`document_thumbnails`).

## Consequences

- Hostile-parse events per document are import-time only, plus at most one lazy backfill per page for pre-v6 rows.
- Stored rasters carry the full 4 MP budget and are aspect-correct; full-screen zoom is a `scaleEffect` over them (`passes-pdf-ui-4`).
- Storage grows by the raster set. Each raster is capped at `DocumentBounds.maxRasterBytes` (20 MiB; a PNG well above the 16 MiB raw-RGBA size is pathological) and the set at `DocumentBounds.maxTotalRasterBytes` (4x the 25 MiB source cap) on both write paths.
- Import peak memory is the accumulated encoded PNGs plus one ~16 MiB render buffer: `renderAllFitted` encodes and releases each page's raw buffer before the next renders. Fine for the 10-page cap, not a shape to grow past it.
- Display decodes off the main actor to what it can show: the inline pager caps the longer side at `DocumentView.inlineMaxPixelSize` (1200 px, about 20 MB across `defaultPageWindow`); full screen caps at `fullScreenMaxPixelSize` (2048 px), since stored dimensions are caller-declared and never verified against the blob.
- `renderFitted` / `renderAllFitted` ship with fail-closed protocol-extension defaults, so the `PDFRendererBinder` additions do not break out-of-package conformers.
- The fit derives from `bounds(for: .mediaBox)`, which ignores `/Rotate` while `page.draw` applies it, so any squash on rotated pages is baked into storage. The mediaBox is attacker-controlled: `fittedDimensions` clamps every value into `[1, maxPixels]` before integer conversion and derives width from the integer budget (no correction loop), so degenerate aspects cost O(1) and cannot trap.
- Android re-renders per open inside its isolated sandbox and stores no rasters, so the iOS schema chain runs one version ahead from v6 (iOS v7/v8 mirror Android v6/v7; see `Schema.swift`).
