# Product Overview

All screenshots on this page are real captures from the Android app, with system status bars removed. The app is published as **DocReader Pro**; the public listing is on [Google Play](https://play.google.com/store/apps/details?id=com.docreader.pro&hl=en).

## What it is

A document reader, scanner and PDF toolbox for phones. People use it to open PDF, Word and Excel files, scan paper documents into PDFs, run OCR on scans, and do common PDF jobs such as merging, splitting, compressing and converting, without leaving the app.

It is a Flutter app for Android and iOS. Documents live in a local SQLite library; Firebase provides optional accounts, settings sync, messaging and analytics.

## First run

A short onboarding explains the main capabilities, a permission screen explains notifications and file access before asking, and sign-in is optional: Google, Apple, or continue as a guest.

<table>
  <tr>
    <td align="center"><img src="../assets/screenshots/docreader-permissions.webp" width="240" alt="Permissions"/><br/><sub>Permissions explained up front</sub></td>
    <td align="center"><img src="../assets/screenshots/docreader-sign-in.webp" width="240" alt="Sign-in"/><br/><sub>Sign in or continue as guest</sub></td>
  </tr>
</table>

## Navigation

Four tabs: **Home**, **Tools**, **Library**, **Settings**.

- **Home** greets the user, offers search, Import and Scan, a "Continue reading" shelf, and favourites.
- **Tools** lists the document tools; most are Pro-gated.
- **Library** is the full file list with a Doc Reader entry that browses documents already on the device.
- **Settings** covers theme, language, sound effects, haptics, keep-screen-on, app lock, biometric unlock, and account.

<table>
  <tr>
    <td align="center"><img src="../assets/screenshots/docreader-home.webp" width="240" alt="Home"/><br/><sub>Home</sub></td>
    <td align="center"><img src="../assets/screenshots/docreader-add-document.webp" width="240" alt="Add document"/><br/><sub>Add Document: scan or import</sub></td>
  </tr>
</table>

## Reading

**PDF.** Three reading layouts (continuous vertical, single page, horizontal swipe), fit-to-width or fit-to-page, pinch zoom, a thumbnails panel and a table-of-contents panel. Full-text search highlights matches on the page. Users can add bookmarks and the app remembers where they stopped; a document read to the end is dropped from the "recent" shelf. Password-protected PDFs prompt for a password. Documents can be printed or shared.

**Word, Excel, text and images.** `.docx` opens in an in-app viewer; `.xls` and `.xlsx` open in a grid viewer with a sheet selector and toolbar; text and image files have their own viewers. PowerPoint files are not rendered in-app: a hand-built renderer did not reach acceptable fidelity and was removed instead of shipping something half-correct.

**Opening from elsewhere.** The app registers for PDF, Word and Excel open and share actions, so a file from email or a file manager opens directly.

## Library

Import from the system picker, with a progress dialog, a summary and a duplicate-resolution sheet. Organise with folders, favourites and pinning. Filter by All, PDF, Word, Images or Scanned and sort by name, date, size or last opened. Multi-select for bulk actions. Cards show a "left off at X%" indicator.

## Scanning and tools

- **Scan to PDF** (free) uses the platform's native document scanner for edge detection, auto-crop and perspective correction, then a review step with crop adjustment and filters.
- **Image to PDF**, **Merge PDF**, **Split PDF**, **Compress PDF** (three quality levels), **PDF to JPG**, and **Text Scan** with on-device OCR.
- Results are saved back to the library; split results are saved as ZIP archives. Export uses the system document picker, and images can be saved to the gallery.

## Access and monetisation

Scan to PDF is free; the other tools are Pro. A tool can be unlocked for one hour by watching a rewarded ad, or permanently with a one-time Pro purchase. Ads (banner, native, interstitial, app-open, rewarded) are switched on and off per placement from remote config.

<p align="center"><img src="../assets/screenshots/docreader-premium-paywall.webp" width="260" alt="Pro paywall"/></p>

## Security and privacy features

App lock with a PIN stored in secure storage and optional biometric unlock. Notification and file-access permissions are explained before the system prompts appear.

## Localisation

Nine languages: English, Arabic, German, Spanish, French, Hindi, Portuguese, Urdu and Chinese.

Continue with the [technical case study](TECHNICAL_CASE_STUDY.md).
