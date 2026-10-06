# Architecture

All diagrams are original and conceptual. They describe responsibilities, not production class or file names.

## 1. Layers

```mermaid
flowchart TD
    UI[Screens and widgets<br/>go_router, bottom-tab shell]
    ST[State<br/>Provider notifiers: files, settings, auth,<br/>lock, subscription, scanner, usage]
    SV[Services<br/>files, import/export, viewers, scan, OCR, PDF tools,<br/>lock, billing, ads, sync, analytics]
    DB[(SQLite<br/>library + settings)]
    SEC[(Secure storage)]
    NT[Native bridges<br/>Kotlin + Swift]
    FB[Firebase]
    UI --> ST --> SV
    SV --> DB
    SV --> SEC
    SV --> NT
    SV --> FB
```

- **State** is held in `provider` notifiers. Screens read from them and call services; services own the I/O.
- **Services** are the only code that touches SQLite, the file system, ML Kit, billing, ads or Firebase.
- **Native bridges** exist where Flutter plugins do not cover the need: listing shared-storage documents, printing, handling incoming file intents and, on iOS, the VisionKit scanner.

## 2. Opening a file

```mermaid
flowchart LR
    A[Library tap] --> R
    B[Device Documents tap] --> R
    C[Shared / VIEW intent] --> N[Resolve URI to a cache path]
    N --> R{Route by extension}
    R -->|pdf| P[PDF viewer]
    R -->|docx| W[Word viewer]
    R -->|xls, xlsx| X[Excel viewer]
    R -->|txt| T[Text viewer]
    R -->|png, jpg...| I[Image viewer]
    R -->|other| U[Preview not available]
```

Incoming Android `VIEW` and `SEND` intents are consumed outside the router. Content URIs are resolved to a real filename through the storage framework's display name, because the path segment of a document URI is often opaque. A cold-start intent is read once at launch; later intents arrive through a callback.

## 3. Scan to PDF

```mermaid
sequenceDiagram
    participant U as User
    participant S as Scanner screen
    participant N as Native scanner
    participant R as Review screen
    participant P as Page processor (isolate)
    participant L as Library
    U->>S: Choose mode (document, card, ID)
    S->>N: ML Kit (Android) / VisionKit (iOS)
    N-->>S: Cropped page images
    S->>R: Pages
    U->>R: Adjust crop, pick filter
    R->>P: Perspective + filter (off UI thread)
    P-->>R: Processed pages (cached previews)
    U->>R: Save
    R->>L: Build PDF, store, record metadata
```

## 4. Library storage

```mermaid
erDiagram
    USERS ||--o{ PDF_FILES : owns
    PDF_FILES }o--o| FOLDERS : "in"
    PDF_FILES ||--o| READING_PROGRESS : tracks
    PDF_FILES ||--o{ PDF_BOOKMARKS : has
    PDF_FILES ||--o{ RECENT_PAGES : remembers
    USERS ||--|| USER_SETTINGS : has
    USERS ||--o{ SEARCH_HISTORY : has
```

Tables beyond these store notifications, usage and import/export statistics, subscription state and credits. The schema is versioned (currently version 9) with migrations. Guest users get local rows the same way signed-in users do.

## 5. Ads and unlocks

```mermaid
flowchart TD
    ACT[User taps a tool] --> PRO{Pro or ads off?}
    PRO -->|yes| RUN[Run tool]
    PRO -->|no| UNL{Rewarded unlock still valid?}
    UNL -->|yes| RUN
    UNL -->|no| GATE[Interstitial gate: placement on, cooldown, online]
    GATE -->|ad shown or skipped| RUN
    RUN --> RES[Result saved to library]
```

Placement modes live in a remote document; a placement is enabled only by an explicit non-zero value, and everything is off until a read succeeds. The tool action starts as soon as the ad plays or is skipped; a "processing" popup stays up if the work outlasts the ad.

## 6. Sync

Settings and profile data merge between Firestore and local SQLite on sign-in and restore, with a live listener for changes from another device, and locally saved changes are flushed when connectivity returns.
