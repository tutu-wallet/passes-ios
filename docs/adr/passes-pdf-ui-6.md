# passes-pdf-ui-6: PDF pager support needs no kernel change

A horizontal pager with a next/previous-page peek needs three things from the kernel. Android had to add them (configurable cache capacity and adjacent-page prefetch, because evicting a still-painted `Bitmap` crashes). The iOS kernel already satisfies all three:

1. **Cache capacity is consumer-configurable.** `PDFThumbnailCache(maxSize:)` is public (default `defaultPageWindow = 5`); a peek pager passes a larger window.
2. **Eviction cannot invalidate a displayed page.** `RenderedPageCache` eviction releases the kernel's reference only (`onEvict` is a no-op for `PageImage`), and `PageImageHandle` holds its own strong `CGImage`-backed `Image`. Android's recycle-while-painted crash class is structurally absent under ARC.
3. **Adjacent-page prefetch needs no dedicated API.** Peeked pages are real visible views, each rendering through its own `PDFThumbnailViewModel` into the shared lock-guarded per-document cache. Warming beyond the peek is possible with the same public surface by driving a spare `PDFThumbnailViewModel` without mounting a view. No speculative prefetch API is added; a measured loading flash would reopen that.

Android source: `passes-android-main/passes-pdf-ui`.
