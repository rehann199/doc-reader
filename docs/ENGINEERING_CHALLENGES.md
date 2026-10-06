# Engineering Challenges

Seven problems that stand out in the production code. Outcomes describe observable behaviour in the code; no metrics are claimed.

---

## 1. Imported files vanish after an iOS reinstall

**Problem**
Library rows recorded absolute file paths under the app's documents directory. On iOS that directory sits in a container whose UUID changes when the app is reinstalled, so every previously imported file looked missing: sharing did nothing, exporting failed and the viewer could not open them.

**Engineering approach**
A small path utility rebases a stored path onto the current sandbox by keeping the portion after the container root and re-anchoring it to wherever the documents directory is now. The current directory is captured once at startup as a plain string so the rebase stays synchronous and usable anywhere. Android paths are stable, so it is a no-op there. The helper has its own unit tests.

**Trade-offs**
Storing relative paths from the start would be cleaner, but would have required migrating existing rows. Rebasing at read time repaired existing installs without a data migration.

**Outcome**
Imported documents resolve again after the container moves.

---

## 2. Search highlights that match the text

**Problem**
A PDF search returns hit rectangles anchored to the text baseline. Drawn as-is they sit under the glyphs, and a case-insensitive search needs to know which casings actually occur in the document.

**Engineering approach**
- Whole-document text is extracted lazily on first search and reused, only to discover which casings of the query exist.
- Per-page text is loaded only for pages that actually produce a hit.
- Baseline-anchored rectangles are expanded using typographic ascent and descent ratios so the box covers the glyphs without a lot of slack.
- Dedicated tests cover highlight alignment, word-span matching and the search service.

**Trade-offs**
The ratios are tuned for common Latin typefaces; unusual fonts may not align perfectly. Lazy extraction makes the first search slower than later ones but avoids paying for documents that are never searched.

**Outcome**
Highlights sit on the matched words, and large documents are not fully processed up front.

---

## 3. A print call that can kill the process

**Problem**
Android's print spooler throws an uncaught exception on a malformed PDF in a place the app cannot catch, which crashes the whole process.

**Engineering approach**
The file is validated before the platform channel is invoked: it must open as a PDF and begin with the PDF header. Synchronous native failures are caught in the bridge, and the Dart side raises a dedicated exception for "not a valid PDF" so the UI can show a message.

**Trade-offs**
Validation cannot catch every malformed file, as the code comments acknowledge, so it is a defence in depth rather than a guarantee.

**Outcome**
The common failure case becomes an error message instead of a crash.

---

## 4. Opening files handed over by other apps

**Problem**
Files arrive as `content://` URIs from email clients and file managers. The last path segment is often an opaque identifier, so the real filename and extension, which decide which viewer to use, are not recoverable from the URI alone. Files can also arrive at cold start or while the app is already running.

**Engineering approach**
- Incoming `VIEW` and `SEND` intents are handled outside the router so a content URI never has to pass through route parsing.
- The native side queries the storage framework for the display name, and copies the content to a cache path the viewers can open.
- A launch-time intent is consumed once; later intents arrive through a callback registered by the app.
- Routing by extension then reuses the same open-file logic as the library.

**Trade-offs**
Copying to cache costs space and time for very large files but gives every viewer a plain file path.

**Outcome**
Shared PDFs, Word and Excel files open directly with the right viewer and their real names.

---

## 5. Scan processing without freezing the screen

**Problem**
Perspective correction, enhancement filters and EXIF orientation fixes are expensive on full-resolution camera images, and users flip between filters while reviewing.

**Engineering approach**
- Native document scanners (ML Kit on Android, VisionKit on iOS) handle capture, edge detection and auto-crop.
- Page processing runs in an isolate through `compute`.
- Enhanced previews and exports are cached separately and never overwrite the original image, so changing filters is reversible.
- EXIF orientation is corrected before PDF building and OCR so pages are not rotated.
- Dart-side edge detection, corner refinement and perspective correction are available for processing page images.

**Trade-offs**
Passing images to an isolate copies data, and the cache uses disk, in exchange for a responsive review screen.

**Outcome**
The review screen stays interactive, and "undo" to the original filter costs nothing.

---

## 6. Exporting from the iOS sandbox

**Problem**
On iOS, a directory chosen with a folder picker comes back as a security-scoped URL. Copying into it with ordinary file APIs fails because the scope is never claimed.

**Engineering approach**
Export goes through the system document picker's export mode, which makes iOS perform the copy with permissions it already holds. A picker that was cancelled is distinguished from one that failed, because an earlier version swallowed all errors and a failed import looked like a cancellation.

**Trade-offs**
The user picks a destination each time instead of a remembered folder.

**Outcome**
Exports work inside iOS's permission model, and failures are surfaced.

---

## 7. Biometric prompts that fight each other

**Problem**
iOS answers a biometric request made while another is pending with a system-cancel result. An automatic prompt on the lock screen plus a manual button tap could strand the unlock flow.

**Engineering approach**
The biometric service tracks whether a prompt is pending and refuses to start another. The lock screen can ask whether one is in progress before firing a redundant request. PINs are stored in secure storage and verified there.

**Trade-offs**
A pending-flag has to be reset reliably on every exit path.

**Outcome**
The lock screen can use both automatic and manual unlock without cancelling itself.
