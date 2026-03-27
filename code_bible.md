# AudioNoti Project Code Bible

## 1. Project Overview & Architecture

**AudioNoti** is a suite of web-based tools for audio annotation, transcription, PDF text extraction, and text-to-speech synthesis. It operates as a set of standalone HTML files that function as Single Page Applications (SPAs).

### Technology Stack
*   **Core Framework**: React 18 (UMD build).
*   **Compiler**: Babel Standalone (in-browser JSX compilation).
*   **Styling**: Tailwind CSS (CDN).
*   **Key Libraries**:
    *   `wavesurfer.js` (Audio visualization & regions).
    *   `pdf.js` (PDF parsing and text extraction).
    *   `jspdf` (PDF generation).
    *   `jszip` (File archiving).

### Data Persistence & Communication
The applications maintain state persistence and communicate via browser APIs without a backend server.

1.  **LocalStorage**: Used for long-term persistence of settings, history, and session data.
    *   `pdf_gen_history`: JSON array of generated TTS history.
    *   `notes_{filename}` / `session_{filename}`: JSON data for audio notes and transcripts.
    *   `studio_editor_*`: Current state of the PDF Editor.
    *   `notetaker_categories`: Custom category definitions.
2.  **SessionStorage**: Used for passing large data payloads (like text content) between tabs during initialization (e.g., "Load into Generator").
3.  **BroadcastChannel**: Enables real-time communication between open tabs.
    *   `studio_audio_bridge`: Transfers audio blobs (e.g., from PDF Converter to Notetaker).
    *   `studio_text_bridge`: Transfers text content (e.g., from Dashboard/Editor to Converter).

---

## 2. File Analysis

### A. `dashboard.html` (Studio Projects)
**Purpose**: Central hub for managing projects, viewing history, and launching tools.

#### Key State
*   `pdfHistory`: Array of metadata from `pdf_converter.html` history.
*   `noteSessions`: Array of saved audio annotation sessions found in LocalStorage.
*   `editorProject`: Metadata about the current active file in `pdf_editor.html`.

#### Key Functionalities
1.  **History Loading**: Scans LocalStorage keys to aggregate distinct project types.
2.  **Cross-App Launching**:
    *   `handleLoadPdfHistory`: Loads text from history into `pdf_converter.html` via `sessionStorage` and `BroadcastChannel`.
    *   `handleLoadNoteSession`: Converts annotation objects into a readable text format and sends to the converter.
3.  **Export**: `exportSession` allows downloading note sessions as CSV or JSON directly from the dashboard.
4.  **Pagination & Filtering**: Client-side logic for searching and sorting project lists.

---

### B. `notetaker.html` (Audio NoteTaker)
**Purpose**: Audio playback, waveform visualization, annotation, and transcript synchronization.

#### Core Component: `AudioNotetaker`
*   **Audio Engine**: Wraps `wavesurfer.js` in a `useRef`. Handles playback, zooming, volume, and playback rate.
*   **Region Plugin**: Used to visualize notes as colored regions on the waveform.
*   **Minimap Plugin**: Provides a navigation map for the audio file.

#### Core Component: `TranscriptViewer`
*   **Rendering**: Renders text with flexible matching logic.
*   **Synchronization**:
    *   `currentStart`: Highlights the active sentence based on `audioprocess` time.
    *   `currentTime`: Used for "Karaoke" style word-level highlighting (interpolated if word timestamps are missing).
    *   `forceScroll`: Programmatic scrolling to keep the active text in view.
    *   **Minimap**: A visual side-bar representing sentence distribution, current playback position, and search matches.
*   **Interaction**:
    *   **Click**: Sync audio to text time.
    *   **Option + Click**: Re-sync (Manual Sync) text time to current audio time.
    *   **Shift + Click**: Open modal to manually edit sentence timestamps.

#### Key Features
*   **Transcript Import/Export**: Supports loading TXT, JSON, SRT, VTT, and CSV. Exports to SRT/VTT.
*   **Transcript Editing**: Full-text edit mode allows rewriting the transcript while attempting to preserve timestamps for unchanged sentences.
*   **Note Management**: CRUD operations for time-coded notes. Notes support categories (colors) and categorization via dropdowns.
*   **Auto-Sync**: Heuristic to distribute plain text linearly across audio duration if no timestamps exist.
*   **Visualizer**: Canvas-based frequency visualizer using `WebAudio API`.

---

### C. `pdf_converter.html` (PDF to Speech)
**Purpose**: Extracts text from PDFs and converts it to speech using the ElevenLabs API.

#### Core Component: `PdfToSpeech`
*   **PDF Parsing**: Uses `pdf.js` to render PDF pages and extract text strings.
*   **Text Handling**: Supports a read-only view and an editable `textarea` mode.

#### Audio Generation Logic
*   **Streaming**: Opens a WebSocket connection to ElevenLabs (`wss://api.elevenlabs.io...`).
*   **Buffering**: Uses `MediaSource` API to append audio chunks as they arrive for low-latency playback.
*   **Alignment**: Captures alignment data (character start times) from the WebSocket to generate `wordBoundaries` for highlighting.

#### Key Features
*   **History**: Caches generated audio blobs (converted to ObjectURLs) in memory and metadata in LocalStorage.
*   **Bulk Export**: Uses `JSZip` to bundle all history items (audio MP3s + CSV metadata) into a single ZIP file.
*   **Integration**:
    *   `handleSendToNoteTaker`: Sends the generated audio blob + text + alignment boundaries to `notetaker.html` via `BroadcastChannel`.

---

### D. `pdf_editor.html` (PDF Editor)
**Purpose**: A text manipulation tool specifically designed to clean up raw text extracted from PDFs before processing.

#### Core Component: `PdfEditor`
*   **Text Engine**: A `textarea` overlaid with a specialized `div` to support syntax highlighting (for Find/Replace).

#### Key Features
1.  **History Stack**: Implements a custom Undo/Redo stack (`history` array state).
2.  **Clean Text Heuristics**:
    *   Joins hyphenated words broken across lines.
    *   Removes single newlines (reflowing paragraphs).
    *   Consolidates whitespace.
3.  **Markdown Helpers**: Toolbar buttons to insert `**bold**`, `*italic*`, and headers.
4.  **Find & Replace**: Regex-based find and replace with case sensitivity toggle and visual highlighting.
5.  **Exports**:
    *   **PDF**: Uses `jspdf` to render text back to a clean PDF.
    *   **JSON/TXT/CSV**: Standard downloadable formats.
6.  **Hot-Reload**: "Send to AudioGen" triggers a broadcast message to instantly update the text in an open `pdf_converter.html` tab.

---

## 3. Data Structures

### Note Object (`notetaker.html`)
```json
{
  "id": "number (timestamp)",
  "timestamp": "number (seconds)",
  "text": "string",
  "color": "hex string"
}
```

### Sentence Boundary Object (`notetaker.html` / `pdf_converter.html`)
```json
{
  "text": "string",
  "start": "number (seconds)",
  "end": "number (seconds)",
  "words": [
    { "word": "string", "start": 0, "end": 0 }
  ]
}
```

### History Item (`pdf_converter.html`)
```json
{
  "id": "number",
  "text": "string",
  "url": "blob:url (runtime only)",
  "voiceName": "string",
  "timestamp": "string (formatted time)"
}
```

## 4. Cross-Communication Protocol

### Audio Bridge (`studio_audio_bridge`)
**Event Type**: `LOAD_AUDIO_WITH_TRANSCRIPT`
**Payload**:
```javascript
{
  type: 'LOAD_AUDIO_WITH_TRANSCRIPT',
  blob: Blob,           // The audio file
  name: String,         // Filename
  transcript: String,   // Full text
  boundaries: Array     // Sentence/Word alignment data
}
```

### Text Bridge (`studio_text_bridge`)
**Event Type**: `LOAD_TEXT`
**Payload**:
```javascript
{
  type: 'LOAD_TEXT',
  text: String,
  fileName: String
}
```