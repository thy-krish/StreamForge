# StreamForge 🎬

A full-stack **Video Streaming Application** built with a **Spring Boot backend** and **React (Vite) frontend**. StreamForge allows users to upload, store, and stream videos efficiently. It features real-time HLS video delivery, chunked streaming, FFmpeg-based video processing, and a smooth responsive frontend experience.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [How It Works](#how-it-works)
- [FFmpeg Integration](#ffmpeg-integration)
- [API Endpoints](#api-endpoints)
- [Classes Overview](#classes-overview)
- [Database Schema](#database-schema)
- [Configuration](#configuration)
- [Setup & Installation](#setup--installation)
- [Future Enhancements](#future-enhancements)
- [Contributing](#contributing)

---

## Overview

StreamForge is a video-on-demand (VOD) streaming platform that mimics core functionality seen in platforms like YouTube. Users upload videos through the React frontend, which are stored and automatically transcoded into HLS format (`.m3u8` + `.ts` segments) using FFmpeg on the backend. Videos are then streamed back in the browser with smooth adaptive playback via HLS.js and Video.js.

---

## Features

- **Video Upload** — Upload MP4 (or other formats) with title and description via an intuitive form UI
- **Automatic HLS Transcoding** — FFmpeg converts videos into HLS format on upload (`.m3u8` manifest + `.ts` segments)
- **Chunked Streaming** — HTTP Range request support for efficient 1MB byte-range video delivery
- **HLS Playback** — Browser-side HLS playback via HLS.js with a full Video.js player UI
- **Course Management** — Videos can be grouped into courses for educational content organization
- **Video Listing** — Retrieve all uploaded videos via REST API
- **Upload Progress Bar** — Real-time upload progress tracking shown to the user
- **Cross-Origin Support** — CORS enabled on all API endpoints

---

## Tech Stack

### Backend
| Technology | Purpose |
|---|---|
| Java 21 | Core language |
| Spring Boot 3.3.3 | REST API framework |
| Spring Data JPA | ORM / database interaction |
| MySQL | Relational database |
| Lombok | Boilerplate reduction |
| FFmpeg | Video transcoding to HLS |
| Maven | Build tool |

### Frontend
| Technology | Purpose |
|---|---|
| React 18 | UI framework |
| Vite | Build tool / dev server |
| Tailwind CSS | Utility-first styling |
| Flowbite React | UI component library |
| Video.js | HTML5 video player |
| HLS.js | HLS playback in browsers |
| Axios | HTTP client |
| React Hot Toast | Notifications |

---

## Architecture

```
┌──────────────────────────────────────────────┐
│               React Frontend                 │
│  VideoUpload.jsx      VideoPlayer.jsx        │
│  (Axios POST)         (HLS.js + Video.js)    │
└────────────────────┬─────────────────────────┘
                     │ HTTP (localhost:8080)
┌────────────────────▼─────────────────────────┐
│          Spring Boot REST API                │
│  POST /api/v1/videos    (upload)             │
│  GET  /api/v1/videos    (list all)           │
│  GET  /stream/{id}      (full stream)        │
│  GET  /stream/range/{id}(chunked stream)     │
│  GET  /{id}/master.m3u8 (HLS manifest)       │
│  GET  /{id}/{seg}.ts    (HLS segment)        │
└──────────┬──────────────────┬────────────────┘
           │                  │
┌──────────▼──────┐  ┌────────▼───────────────┐
│   MySQL DB      │  │  File System           │
│  (video meta)   │  │  videos/   → raw MP4   │
│                 │  │  videos_hsl/ → HLS      │
└─────────────────┘  └────────────────────────┘
```

---

## Project Structure

```
StreamForge/
│
├── src/                          # Spring Boot backend
│   └── main/java/com/stream/app/spring_stream_backend/
│       ├── controllers/
│       │   └── VideoController.java       # REST endpoints
│       ├── entities/
│       │   ├── Video.java                 # Video JPA entity
│       │   └── Course.java                # Course entity (future use)
│       ├── repositories/
│       │   └── VideoRepository.java       # JPA repository
│       ├── services/
│       │   ├── VideoService.java          # Service interface
│       │   └── impl/VideoServiceImpl.java # Business logic + FFmpeg
│       ├── payload/
│       │   └── CustomeMessage.java        # API response wrapper
│       ├── AppConstants.java              # Chunk size constant (1MB)
│       └── SpringStreamBackendApplication.java
│
├── src/main/resources/
│   └── application.properties             # DB, file paths, upload limits
│
├── stream-front-end/                      # React frontend
│   └── src/
│       ├── components/
│       │   ├── VideoPlayer.jsx            # HLS player component
│       │   └── VideoUpload.jsx            # Upload form component
│       ├── App.jsx                        # Root component
│       └── main.jsx
│
├── videos/                                # Uploaded raw video files
├── videos_hsl/                            # FFmpeg HLS output
│   └── {videoId}/
│       ├── master.m3u8
│       └── segment_000.ts, segment_001.ts ...
│
├── pom.xml                                # Maven dependencies
└── package.json                           # Frontend dependencies
```

---

## How It Works

### Upload Flow

1. User fills in title, description, and selects a video file in the React UI.
2. Frontend sends a `multipart/form-data` POST request to `/api/v1/videos`.
3. Backend saves the raw file to the `videos/` folder on disk.
4. Video metadata (title, description, file path, content type) is saved to MySQL.
5. `processVideo()` is triggered — FFmpeg transcodes the video into HLS format.
6. HLS output (`.m3u8` + `.ts` segments) is stored under `videos_hsl/{videoId}/`.

### Playback Flow

1. User enters a `videoId` in the player UI and clicks Play.
2. Frontend requests `GET /api/v1/videos/{videoId}/master.m3u8`.
3. HLS.js loads the manifest and sequentially fetches `.ts` segments.
4. Video.js renders the player with full playback controls.

---

## FFmpeg Integration

On every upload, FFmpeg is invoked to convert the raw video into HLS (HTTP Live Streaming) format, enabling smooth adaptive playback across devices and network speeds.

### FFmpeg Command

```bash
ffmpeg -i "<input_video>" \
  -c:v libx264 \
  -c:a aac \
  -strict -2 \
  -f hls \
  -hls_time 10 \
  -hls_list_size 0 \
  -hls_segment_filename "<output_dir>/segment_%3d.ts" \
  "<output_dir>/master.m3u8"
```

### Parameter Breakdown

| Flag | Description |
|---|---|
| `-i` | Input video file path |
| `-c:v libx264` | H.264 video codec — high compatibility, efficient compression |
| `-c:a aac` | AAC audio codec — standard for streaming |
| `-strict -2` | Allows experimental AAC codec usage |
| `-f hls` | Output format set to HLS |
| `-hls_time 10` | Each segment is 10 seconds long |
| `-hls_list_size 0` | All segments retained in the playlist indefinitely |
| `-hls_segment_filename` | Segment naming pattern: `segment_001.ts`, `segment_002.ts`, ... |
| `master.m3u8` | HLS playlist file referenced by the video player |

---

## API Endpoints

### `POST /api/v1/videos`
Upload a new video.

**Form Data:**
| Field | Type | Description |
|---|---|---|
| `file` | MultipartFile | The video file |
| `title` | String | Video title |
| `description` | String | Video description |

**Response:** `200 OK` with saved `Video` object, or `500` on failure.

---

### `GET /api/v1/videos`
Returns a list of all uploaded videos.

---

### `GET /api/v1/videos/stream/{videoId}`
Streams the full raw video file.

---

### `GET /api/v1/videos/stream/range/{videoId}`
Streams video in 1MB chunks using HTTP Range headers.

**Request Header:** `Range: bytes=0-`  
**Response:** `206 Partial Content` with chunk data.

---

### `GET /api/v1/videos/{videoId}/master.m3u8`
Serves the HLS master playlist for a given video.

---

### `GET /api/v1/videos/{videoId}/{segment}.ts`
Serves an individual HLS `.ts` segment file.

---

## Classes Overview

| Class | Description |
|---|---|
| `VideoController` | REST endpoints for upload, streaming, HLS manifest & segments |
| `Video` | JPA entity — stores video metadata (title, description, path, type) |
| `Course` | JPA entity — groups videos into courses (scaffolded for future use) |
| `VideoRepository` | Spring Data JPA interface for DB operations |
| `VideoService` | Service interface defining the contract for video operations |
| `VideoServiceImpl` | Business logic: file storage, DB save, FFmpeg transcoding |
| `CustomeMessage` | Response payload wrapper for API error/success messages |
| `AppConstants` | Holds global constants (e.g. `CHUNK_SIZE = 1MB`) |

---

## Database Schema

**Table: `yt_videos`**

| Column | Type | Description |
|---|---|---|
| `video_id` | VARCHAR (PK) | UUID generated on upload |
| `title` | VARCHAR | Video title |
| `description` | TEXT | Video description |
| `content_type` | VARCHAR | MIME type (e.g. `video/mp4`) |
| `file_path` | VARCHAR | Absolute path to raw video on disk |

---

## Configuration

All configuration lives in `src/main/resources/application.properties`:

```properties
# MySQL
spring.datasource.url=jdbc:mysql://localhost:3306/video_streaming
spring.datasource.username=your_username
spring.datasource.password=your_password

# JPA
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# Upload size limits
spring.servlet.multipart.max-file-size=10000MB
spring.servlet.multipart.max-request-size=10000MB

# Storage paths
files.video=videos/
file.video.hsl=videos_hsl/
```

> ⚠️ **Never commit real credentials.** Add `application.properties` to `.gitignore` or use environment variables before pushing to GitHub.

---

## Setup & Installation

### Prerequisites

- Java 21+
- Maven
- MySQL
- Node.js 18+
- FFmpeg installed and available in system PATH

---

### Backend Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/your-username/StreamForge.git
   cd StreamForge
   ```

2. Create the MySQL database:
   ```sql
   CREATE DATABASE video_streaming;
   ```

3. Update `application.properties` with your DB credentials.

4. Run the backend:
   ```bash
   ./mvnw spring-boot:run
   ```
   Server starts at `http://localhost:8080`.

---

### Frontend Setup

1. Navigate to the frontend folder:
   ```bash
   cd stream-front-end
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Start the dev server:
   ```bash
   npm run dev
   ```
   App runs at `http://localhost:5173`.

---

## Future Enhancements

- **User Authentication** — Spring Security + JWT for login and personalized content
- **Adaptive Bitrate (ABR)** — Multiple quality levels (360p, 720p, 1080p) in HLS output
- **Course Grouping** — `Course` entity already scaffolded; assign videos to courses
- **Cloud Storage** — Replace local file system with AWS S3 or similar
- **Thumbnail Generation** — Auto-generate preview thumbnails from video frames via FFmpeg
- **Search & Filter** — Search videos by title or description
- **Commenting System** — Allow users to comment on videos
- **Recommendations** — Suggest videos based on watch history
- **Linux/macOS Support** — FFmpeg command currently uses `cmd.exe`; refactor for Unix compatibility

---

## Contributing

Contributions are welcome! To contribute:

1. Fork the repository
2. Create a new branch (`git checkout -b feature/your-feature`)
3. Commit your changes (`git commit -m 'Add some feature'`)
4. Push to your branch (`git push origin feature/your-feature`)
5. Open a Pull Request

---

## License

This project is open source and available under the [MIT License](LICENSE).
