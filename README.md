# Nihilent_VideoTransferSystem

## Overview
This system is designed to fulfill the Confluentia 2026 Technical Challenge requirements. It features a fault-tolerant, chunk-based video transfer protocol capable of surviving severe network fluctuations (20%-100%), temporary disconnections, and connection switching. 

Once a 30-second video is reliably reassembled and validated byte-for-byte against the client's original payload, an asynchronous background pipeline analyzes the video at 30 FPS to detect and localize frame anomalies (drops, duplicates, or decoding errors). 

## 🏗️ System Architecture

The application is divided into three distinct operational layers:

### 1. Client Layer (Frontend)
Responsible for user interaction, file ingestion, chunking, and rendering the final proof-of-work.
*   **File Slicing:** Uses the browser's `File.slice()` API to divide the 30-second video into manageable chunks (e.g., 1MB each).
*   **Hashing:** Calculates a SHA-256 hash of the entire file before transmission to uniquely identify the upload session and ensure final integrity.
*   **Resumption Logic:** Automatically queries the backend upon connection restoration (`GET /upload/status`) to determine missing chunks, resuming only what failed.
*   **Playback & Reporting:** Asynchronously polls the backend. Displays the video immediately upon successful reassembly (status: `Processing`), and updates the UI with a detailed frame integrity report once processing concludes.

### 2. API & Application Layer (Backend)
Handles state management, file reassembly, and coordinates background tasks.
*   **Chunk Receiver:** Exposes endpoints to receive chunks in any order and stores them in a temporary `.part` directory.
*   **State Tracker (SQLite):** Tracks the progress of each `fileHash`. Records which chunk indices have been safely written to disk.
*   **Validation:** Upon receiving the final chunk, reassembles the file, computes the server-side SHA-256 hash, and compares it to the client's hash.
*   **Task Delegation:** If the hash is validated, moves the video to permanent storage and triggers the asynchronous analysis pipeline.

### 3. Processing Layer (Background Worker)
Executes heavy media processing without blocking the backend API or frontend client.
*   **FFmpeg Decoding:** Leverages `ffmpeg-python` and `ffprobe` to decode the video at exactly 30 FPS.
*   **Metadata Analysis:** Inspects Presentation Timestamps (PTS) and Decoding Timestamps (DTS) to differentiate between network-induced file corruption and source-level frame drops.
*   **Report Generation:** Saves the findings (missing, skipped, or duplicated frames with their exact timestamps) into the SQLite database.

---

## 🛠️ Tech Stack

*   **Frontend:** React 19, TypeScript, Vite (Fast development, robust typing for File APIs)
*   **Backend:** Python 3.10+, FastAPI (High-performance async endpoints)
*   **Database / State:** SQLite (Zero-configuration, highly portable for challenge submission)
*   **Video Processing:** FFmpeg (via `ffmpeg-python`), `ffprobe` for frame metadata
*   **Background Tasks:** FastAPI `BackgroundTasks` (or `asyncio` tasks) for zero-dependency async execution.

---

## 🗄️ Database Schema (SQLite)

The system utilizes a lightweight relational schema:

1.  **`uploads` Table:**
    *   `hash_id` (Primary Key, String): SHA-256 of the file.
    *   `filename` (String): Original file name.
    *   `total_chunks` (Integer): Total number of chunks expected.
    *   `status` (String): `uploading`, `validating`, `processing`, `completed`, `failed`.
    *   `final_path` (String): Local path to the reassembled video.

2.  **`chunks` Table:**
    *   `id` (Primary Key)
    *   `upload_id` (Foreign Key -> `uploads.hash_id`)
    *   `chunk_index` (Integer): The index of the received chunk.

3.  **`reports` Table:**
    *   `id` (Primary Key)
    *   `upload_id` (Foreign Key -> `uploads.hash_id`)
    *   `is_valid` (Boolean)
    *   `anomalies_json` (Text): JSON string detailing timestamp anomalies.

---

## 🚀 Upload Workflow (Fault-Tolerant)

1.  **Init:** Client selects video -> calculates SHA-256 -> calls `POST /init` with hash and `total_chunks`.
2.  **Upload:** Client loops through chunks, sending them via `POST /upload/{hash_id}/{chunk_index}`.
3.  **Interruption:** Network drops. Client catches the exception.
4.  **Recovery:** Connection returns. Client calls `GET /status/{hash_id}`. Backend queries SQLite and returns an array of received `chunk_index` values.
5.  **Resume:** Client filters out the received chunks and uploads only the missing data.
6.  **Complete:** Backend receives the last chunk -> Reassembles -> Validates Hash -> Updates status to `processing` -> Triggers async FFmpeg task.

## 🏃‍♂️ Run Instructions (Draft)

*(To be finalized once code is implemented)*

```bash
# 1. Setup Backend
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python main.py # Runs FastAPI on localhost:8000

# 2. Setup Frontend
cd frontend
npm install
npm run dev # Runs Vite on localhost:5173
```