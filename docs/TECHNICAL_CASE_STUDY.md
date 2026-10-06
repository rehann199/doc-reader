# Technical Case Study: Doc Reader

An engineering write-up of a Flutter document reader, scanner and PDF toolbox, published as DocReader Pro. It is based on read-only inspection of the production codebase. No production source is reproduced; descriptions are deliberately high level.

## 1. Product context

Doc Reader is a mobile document app that combines three jobs that are often separate apps: a viewer for common office formats, a scanner with OCR, and a set of PDF utilities. It has a free tier with ads and a Pro tier unlocked by a one-time purchase or temporarily by a rewarded ad.

Most of the engineering effort goes into file handling: getting documents into the app from many sources, keeping paths valid, rendering several formats, and doing CPU-heavy work without freezing the UI.

## 2. Requirements that shaped the design

| Area | Requirement |
|---|---|
| Formats | PDF, Word, Excel, text and images, opened from the library, the device, or another app |
| Files | Work on Android scoped storage and iOS sandboxing without surprising the user |
| Scanning | Native capture quality, editable crop and filters, on-device OCR |
| Tools | Merge, split, compress, convert, all producing files that land back in the library |
| Library | Fast to browse, remembers where you stopped, supports organisation |
| Privacy | Documents stay on-device; optional lock; no account required |
| Monetisation | Free scanning, Pro tools, remotely controlled ads |

## 3. Architecture

Screens → `provider` notifiers → services → SQLite, secure storage, native bridges and Firebase. See [ARCHITECTURE.md](ARCHITECTURE.md).

- **Routing.** `go_router` with a tab shell. Incoming file intents bypass the router entirely.
- **State.** Notifiers for files, settings, auth, lock, subscription, scanner, locale, notifications and usage statistics.
- **Services.** One per concern: import, export, thumbnails, search, outline, OCR, each PDF tool, lock, billing, ads, sync, updates. Screens do not call plugins directly for these.
- **Startup.** A native splash, a splash video, and a startup trace that logs timings. Firebase is initialised defensively so a failure does not stop the app from opening.

## 4. Rendering and viewing

- **PDF.** `pdfrx` (PDFium) renders pages and thumbnails. A single viewer type handles continuous and paged layouts by changing the page layout function, so there is one controller to manage. Fit mode is applied on load and re-applied on rotation or page change unless the user has zoomed manually, in which case their zoom is left alone.
- **Selection.** The viewer's built-in selection menu did not deliver taps reliably, so it is suppressed and replaced with a custom popup that lives in the screen's own hit-testing area.
- **Password-protected PDFs.** The app prompts, and keeps a decrypted copy in the cache for the session, deleted when the viewer closes.
- **Word.** `.docx` renders through a vendored Office-viewer package.
- **Excel.** `.xls`/`.xlsx` parse into a workbook model and render in a grid with a sheet selector and toolbar.
- **PowerPoint.** Deliberately unsupported. A hand-written renderer could not reach acceptable fidelity (fonts, fills, layout inheritance) and was removed instead of shipped half-correct; such files show a "preview not available" state.

## 5. Library and storage

- **SQLite** (`sqflite`, schema version 9, with migrations) holds users (including a local guest), files, nested folders, favourites, bookmarks, reading progress, recent pages, search history, settings, notifications, subscription state and usage statistics.
- **Reading progress** is loaded in bulk and keyed by file so the library list can show "left off at X%" on every card without one query per file. Documents finished past a threshold drop out of "recent".
- **Import** goes through the system picker with a progress dialog, a summary, and a duplicate-resolution step. A cancelled pick is distinguished from a failed one.
- **Device documents** are listed natively through MediaStore plus a shallow walk of common public folders, with thumbnails, so users can open files without importing them.
- **Paths** are rebased onto the current sandbox at read time to survive iOS container changes.
- **Export** uses the system document picker so iOS performs the copy; images can be saved to the gallery.

## 6. Scanning, OCR and PDF tools

- **Capture.** ML Kit document scanner on Android, VisionKit on iOS through a Swift bridge, with modes for document, business card and ID/photo.
- **Review.** Crop editor, page strip, and filters: original, auto, colour, grayscale, black and white.
- **Processing.** Perspective correction and filters in an isolate; previews cached; EXIF orientation corrected first.
- **OCR.** ML Kit text recognition configured for Latin-script languages, with detection of PDFs that already contain extractable text so OCR is not run needlessly.
- **Tools.** Image to PDF, merge, split (results packaged as ZIP), compress at three quality levels via image recompression and a full rewrite, PDF to JPG.

## 7. Native integration

| Platform | Capability |
|---|---|
| Android | Document listing via MediaStore with thumbnails; print through the platform print service; `VIEW`/`SEND` intent handling; display-name lookup for content URIs |
| iOS | VisionKit scanner; print; share |
| Both | Google and Apple sign-in, local notifications and push, in-app update (Play update API on Android, store version check on iOS), in-app review prompt |

Storage permissions are the legacy read/write permissions capped by SDK level, with media permissions explicitly removed from the manifest; modern access goes through MediaStore queries and the system pickers.

## 8. Accounts, sync and security

- Google and Apple sign-in, or guest. Guest and signed-in users share the same local schema.
- Profile and settings merge between Firestore and local SQLite on sign-in and restore, a live listener applies remote changes, and local edits made offline are flushed when connectivity returns.
- App lock stores a PIN in secure storage; biometric unlock uses the platform API with a guard against overlapping prompts.

## 9. Monetisation

- **Pro** is a one-time purchase handled by a wrapper over `in_app_purchase` that exposes live store prices and supports restore. A debug flag can bypass the store for QA.
- **Per-tool unlocks.** Scan to PDF is free; other tools can be unlocked for an hour by watching a rewarded ad. Unlock expiries are persisted per tool.
- **Ads.** Google Mobile Ads with banner, native, interstitial, app-open and rewarded formats. Each placement has an integer mode in a remote document: zero is off and anything else is on. The default is off until a read succeeds. A shared cooldown and online check gate interstitials, and tool actions run behind a "loading ad" popup that switches to "processing" if the tool outlasts the ad.

## 10. Performance

- Lazy thumbnail rendering with in-memory caching.
- Lazy, per-page text extraction for search.
- Isolate-based scan processing and preview caching.
- Shimmer placeholders while native and banner ads load.
- A startup trace for timing, Firebase Performance for field data.

## 11. Testing

Automated tests cover sandbox path rebasing, ad placement mode parsing and ad configuration, PDF search highlight alignment, word-span matching and the search service, plus a default widget test. Viewers, import/export, scanning and billing are not covered by automated tests.

## 12. Trade-offs and maintainability

- **Strengths:** services separated from screens, dedicated tests for the fiddliest algorithms (search alignment, path repair), comments that explain why decisions were made, and willingness to remove features that were not good enough (PowerPoint).
- **Weaknesses:** several screens are very large (the PDF viewer, library and home screens each run to hundreds or over a thousand lines); some schema columns and settings are leftovers from an earlier product scope; most viewer, scan and billing paths rely on manual testing.

## 13. Possible next steps

- Break the largest screens into smaller widgets and view-models.
- Add golden or integration tests for the viewer and scan review flows.
- Store paths relative to the documents directory in a future schema migration.
- Remove unused columns and settings.
- Add a fidelity-first PowerPoint renderer if it can be done properly.
