# QVox API Reference

## Base URL

```
Production:  https://api.education1.uk
Local:       http://localhost:8000
```

---

## Authentication

All endpoints require API key authentication:

### Batch Transcription (`POST /v1/transcribe`)
```
Authorization: Bearer <api_key>
```

### Streaming (`WS /v1/stream`)
```
wss://api.education1.uk/v1/stream?api_key=<api_key>
```

---

## Endpoints Overview

| Method | Endpoint | Auth | Purpose | Latency |
|--------|----------|------|---------|---------|
| `POST` | `/v1/transcribe` | ✅ Bearer header | Batch transcription | ~2-5s |
| `WS` | `/v1/stream` | ✅ Query param | Real-time chunk-by-chunk streaming | ~100-500ms per chunk |
| `GET` | `/` | ❌ | Health check | <10ms |

---

## POST /v1/transcribe

Batch transcription with speaker diarization.

**Request:**
```
POST /v1/transcribe
Content-Type: multipart/form-data
Authorization: Bearer <api_key>
```

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | ✅ | Must be `"QVox"` (case-sensitive). Invalid model returns 400. |
| `file` | file | ❌ | Audio file (max 500MB) |
| `url` | string | ❌ | Audio URL (YouTube supported) |
| `lang` | string | ❌ | Language code (see below) |

> Provide `file` **OR** `url`, not both.

**Supported Languages:**
```
as, bn, brx, doi, gu, hi, kn, kok, ks, mai, ml, mni, mr, ne, or, pa, sa, sat, sd, ta, te, ur
```

- `lang` not provided → Canary ASR (English) **+ Speaker Diarization** (parallel)
- `lang` provided + supported → Indic ASR only (no diarization — Indic ASR does not return segments)

> **Note:** When `lang` is provided, `segments` will be an empty array because Indic ASR only returns full text, not per-speaker segments.

**Supported Audio Formats (Batch):**
```
WAV, MP3, FLAC, OGG, WebM, M4A, AAC, OPUS
```

**Response:**
```json
{
  "model": "QVox",
  "text": "Full transcription text...",
  "segments": [
    {
      "speaker": "speaker_0",
      "start": 0.5,
      "end": 2.3,
      "text": "Hello world"
    }
  ]
}
```

> **`segments` behavior:** Only populated when `lang` is **not** provided (Canary ASR + Diarization). When `lang` is provided (Indic ASR), `segments` is an empty array `[]`.

**Errors:**

| Status | Cause |
|--------|-------|
| 400 | No file or URL provided |
| 400 | Invalid model name (must be "QVox") |
| 400 | Unsupported language |
| 400 | YouTube download failed |
| 401 | Missing Authorization header |
| 403 | Invalid API key |
| 413 | File exceeds 500MB |
| 500 | ASR service error |

**Logs (production):**
```
[HH:MM:SS] YouTube download started: https://youtube.com/...
[HH:MM:SS] Downloading:  10.0% @  xx.xxMiB/s
[HH:MM:SS] Downloading:  20.0% @  xx.xxMiB/s
...
[HH:MM:SS] Download finished, processing with FFmpeg...
[HH:MM:SS] YouTube downloaded: ... -> /tmp/xxxxx.mp3
[HH:MM:SS] File saved: /tmp/xxxxx.mp3
[HH:MM:SS] Starting pipeline
[HH:MM:SS] Calling Indic ASR: ... (lang=bn)   # if lang provided
[HH:MM:SS] Calling Canary ASR: ...            # if no lang
[HH:MM:SS] Calling Diarization: ...            # if no lang
Pipeline finished, Segments: XX
POST /v1/transcribe 200 OK
```

---

## WS /v1/stream

Real-time **chunk-by-chunk** streaming transcription via WebSocket.

**Connection:**
```
ws://host/v1/stream?api_key=<api_key>
```

### Protocol

**Client → Server:**
| Message | Description |
|---------|-------------|
| Binary audio (WAV or PCM) | Raw audio bytes per chunk |
| `{"type":"end"}` | End session |

**Server → Client:**
| Message | Description |
|---------|-------------|
| `{"type": "ready"}` | Connection established, auth passed |
| `{"type": "transcript", "text": "..."}` | Transcription result for chunk |
| `{"type": "done"}` | Session complete |
| `{"type": "error", "message": "..."}` | Error occurred |

### Supported Streaming Formats

> **IMPORTANT:** Streaming endpoint only supports **WAV** and **raw PCM** formats for real-time chunk-by-chunk processing.

| Format | Magic Bytes | Processing | Streaming |
|--------|-------------|------------|----------|
| WAV | `RIFF....WAVE` | Header stripped → int16 | ✅ Supported |
| Raw PCM | None | Direct int16→float32 | ✅ Supported |
| MP3 | `ID3`, `0xFF FB/F3/F2/FA` | librosa decode | ❌ Use /transcribe |
| WebM | `EBML` header | FFmpeg decode | ❌ Use /transcribe |
| OGG | `OggS` | librosa decode | ❌ Use /transcribe |
| FLAC | `fLaC` | librosa decode | ❌ Use /transcribe |

> **For MP3, WebM, OGG, FLAC audio:** Use the `/v1/transcribe` batch endpoint instead.

### Chunk Size Recommendations

```
2-3 seconds of audio per chunk (16kHz, 16-bit mono)
= 32,000 to 48,000 bytes per chunk
```

- Smaller chunks = lower latency per response
- Larger chunks = better accuracy per chunk
- Recommended: 32KB-48KB per chunk for optimal balance

### Audio Requirements

| Property | Value |
|----------|-------|
| Sample Rate | 16,000 Hz |
| Bit Depth | 16-bit |
| Channels | 1 (mono) |
| Encoding | PCM (signed integer) or WAV |

### Example Flow

```
Client sends chunk1 (32KB WAV)  →  Server processes  →  {"type":"transcript","text":"Hello"}
Client sends chunk2 (32KB WAV)  →  Server processes  →  {"type":"transcript","text":"world"}
Client sends {"type":"end"}     →  Server sends     →  {"type":"done"}
```

---

## Client Examples

### Python — Batch Transcription

```python
import requests

# Production: https://api.education1.uk/v1/transcribe
# Local: http://localhost:8000/v1/transcribe
url = "https://api.education1.uk/v1/transcribe"
headers = {"Authorization": "Bearer <api_key>"}

# File upload
with open("audio.wav", "rb") as f:
    response = requests.post(url, headers=headers, files={"file": f}, data={"model": "QVox"})

print(response.json())

# With language hint (Hindi)
with open("audio.wav", "rb") as f:
    response = requests.post(url, headers=headers, files={"file": f}, data={"model": "QVox", "lang": "hi"})
```

### Python — Streaming (WAV/PCM only)

```python
import asyncio
import websockets
import json

async def stream_transcribe(audio_path: str, api_key: str):
    """
    Real-time streaming transcription.
    Sends audio in chunks, receives partial transcripts.
    NOTE: Only WAV and raw PCM formats supported for streaming.
    Production: wss://api.education1.uk/v1/stream
    Local: ws://localhost:8000/v1/stream
    """
    # Production
    uri = f"wss://api.education1.uk/v1/stream?api_key={api_key}"
    # Local development
    # uri = f"ws://localhost:8000/v1/stream?api_key={api_key}"

    async with websockets.connect(uri, ping_timeout=None) as ws:
        await ws.recv()  # {"type":"ready"}

        with open(audio_path, "rb") as f:
            # Read and send in chunks (32KB each = ~2 sec at 16kHz)
            chunk_size = 32000
            while chunk := f.read(chunk_size):
                await ws.send(chunk)
                # Receive partial transcript for this chunk
                msg = await ws.recv()
                data = json.loads(msg)
                if data.get('type') == 'transcript':
                    print(f"Partial: {data['text']}")

        # Send end signal and wait for final response
        await ws.send(json.dumps({"type": "end"}))
        final = await ws.recv()  # {"type":"done"}

asyncio.run(stream_transcribe("audio.wav", "<api_key>"))
```

### Python — Real-time Microphone (WAV/PCM)

```python
import asyncio
import websockets
import json
import pyaudio

async def stream_mic(duration: int, api_key: str):
    """
    Real-time microphone streaming.
    Captures audio in WAV/PCM format and streams to server.
    Production: wss://api.education1.uk/v1/stream
    Local: ws://localhost:8000/v1/stream
    """
    CHUNK = 32000  # ~2 seconds at 16kHz
    FORMAT = pyaudio.paInt16  # 16-bit PCM
    CHANNELS = 1  # mono
    RATE = 16000   # 16kHz

    # Production
    uri = f"wss://api.education1.uk/v1/stream?api_key={api_key}"
    # Local development
    # uri = f"ws://localhost:8000/v1/stream?api_key={api_key}"

    async with websockets.connect(uri, ping_timeout=None) as ws:
        await ws.recv()  # ready

        p = pyaudio.PyAudio()
        stream = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True)

        for _ in range(int(RATE / CHUNK * duration)):
            data = stream.read(CHUNK, exception_on_overflow=False)
            await ws.send(data)
            # Receive partial transcript
            try:
                msg = await asyncio.wait_for(ws.recv(), timeout=10)
                data = json.loads(msg)
                if data.get('type') == 'transcript':
                    print(f"Transcript: {data['text']}")
            except asyncio.TimeoutError:
                pass  # No transcript for this chunk yet

        stream.stop_stream()
        p.terminate()

        await ws.send(json.dumps({"type": "end"}))
        await ws.recv()  # done

asyncio.run(stream_mic(10, "<api_key>"))
```

### Node.js — Batch Transcription

```javascript
const FormData = require('form-data');
const fs = require('fs');
const axios = require('axios');

const apiKey = '<api_key>';
// Production: https://api.education1.uk/v1/transcribe
// Local: http://localhost:8000/v1/transcribe
const url = 'https://api.education1.uk/v1/transcribe';

// File upload
const form = new FormData();
form.append('model', 'QVox');
form.append('file', fs.createReadStream('audio.wav'));

const response = await axios.post(url, form, {
    headers: {
        ...form.getHeaders(),
        'Authorization': `Bearer ${apiKey}`
    }
});

console.log(response.data);

// With language hint (Hindi)
const formHi = new FormData();
formHi.append('model', 'QVox');
formHi.append('file', fs.createReadStream('audio.wav'));
formHi.append('lang', 'hi');

const resHi = await axios.post(url, formHi, {
    headers: {
        ...formHi.getHeaders(),
        'Authorization': `Bearer ${apiKey}`
    }
});
```

### Node.js — Streaming (WAV/PCM only)

```javascript
const WebSocket = require('ws');
const fs = require('fs');

const apiKey = '<api_key>';
// Production
const wsUrl = `wss://api.education1.uk/v1/stream?api_key=${apiKey}`;
// Local development
// const wsUrl = `ws://localhost:8000/v1/stream?api_key=${apiKey}`;

const ws = new WebSocket(wsUrl);

ws.on('open', () => {
    // Read audio file and send in chunks (32KB each)
    const chunkSize = 32000;
    const audio = fs.readFileSync('audio.wav');

    let offset = 0;
    while (offset < audio.length) {
        const chunk = audio.slice(offset, offset + chunkSize);
        ws.send(chunk);
        offset += chunkSize;
    }

    ws.send(JSON.stringify({ type: 'end' }));
});

ws.on('message', (data) => {
    const msg = JSON.parse(data.toString());

    if (msg.type === 'transcript') {
        console.log('Partial:', msg.text);
    } else if (msg.type === 'done') {
        ws.close();
    }
});
```

### React.js — Browser Streaming (WAV/PCM)

```jsx
import { useState, useRef, useEffect } from 'react';

function AudioTranscriber() {
  const [transcript, setTranscript] = useState('');
  const [isRecording, setIsRecording] = useState(false);
  const wsRef = useRef(null);
  const apiKey = '<api_key>';

  // Production: wss://api.education1.uk/v1/stream
  // Local: ws://localhost:8000/v1/stream
  const WS_URL = 'wss://api.education1.uk/v1/stream';

  const startRecording = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    const ctx = new AudioContext({ sampleRate: 16000 });
    const source = ctx.createMediaStreamSource(stream);
    const processor = ctx.createScriptProcessor(4096, 1, 1);

    processor.onaudioprocess = (e) => {
      if (wsRef.current?.readyState === WebSocket.OPEN) {
        const monoData = e.inputBuffer.getChannelData(0);
        const pcm = new Int16Array(monoData.length);
        for (let i = 0; i < monoData.length; i++) {
          pcm[i] = Math.max(-1, Math.min(1, monoData[i])) * 0x7FFF;
        }
        wsRef.current.send(pcm.buffer);
      }
    };

    source.connect(processor);
    processor.connect(ctx.destination);

    const ws = new WebSocket(`${WS_URL}?api_key=${apiKey}`);
    ws.onmessage = (e) => {
      const data = JSON.parse(e.data);
      if (data.type === 'transcript') {
        setTranscript(prev => prev + ' ' + data.text);
      }
    };
    wsRef.current = ws;
    setIsRecording(true);
  };

  const stopRecording = () => {
    wsRef.current?.send(JSON.stringify({ type: 'end' }));
    wsRef.current?.close();
    setIsRecording(false);
  };

  useEffect(() => {
    return () => {
      wsRef.current?.close();
    };
  }, []);

  return (
    <div>
      <button onClick={startRecording} disabled={isRecording}>Start</button>
      <button onClick={stopRecording} disabled={!isRecording}>Stop</button>
      <p>{transcript}</p>
    </div>
  );
}
```

### Next.js — Streaming with API Route Proxy

For Next.js, best practice hai WebSocket ko API route se proxy karna:

```typescript
// app/api/transcribe-stream/route.ts
import { NextRequest } from 'next/server';
import { useSocket } from '@socket.io/nextjs';

const API_KEY = process.env.QVOX_API_KEY!;
const WS_URL = process.env.NEXT_PUBLIC_QVOX_WS_URL || 'wss://api.education1.uk/v1/stream';

export async function POST(req: NextRequest) {
  const formData = await req.formData();
  const audioFile = formData.get('file') as File;

  if (!audioFile) {
    return Response.json({ error: 'No audio file' }, { status: 400 });
  }

  // For streaming, connect directly from client to WSS endpoint
  // This API route is for batch transcription proxy
  const response = await fetch('https://api.education1.uk/v1/transcribe', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${API_KEY}`
    },
    body: formData
  });

  return Response.json(await response.json());
}
```

```typescript
// app/components/StreamTranscriber.tsx
'use client';

import { useState, useRef } from 'react';

export function StreamTranscriber() {
  const [transcript, setTranscript] = useState('');
  const wsRef = useRef<WebSocket | null>(null);

  // Production: wss://api.education1.uk/v1/stream
  // Local: ws://localhost:8000/v1/stream
  const WS_URL = process.env.NEXT_PUBLIC_QVOX_WS_URL || 'wss://api.education1.uk/v1/stream';

  const connect = (apiKey: string) => {
    const ws = new WebSocket(`${WS_URL}?api_key=${apiKey}`);
    ws.onmessage = (e) => {
      const data = JSON.parse(e.data);
      if (data.type === 'transcript') {
        setTranscript(prev => prev + ' ' + data.text);
      }
    };
    wsRef.current = ws;
  };

  return <div>{/* UI components */}</div>;
}
```

```bash
# .env.local
NEXT_PUBLIC_QVOX_WS_URL=wss://api.education1.uk/v1/stream
QVOX_API_KEY=your_api_key_here
```

### cURL — Batch Only

```bash
# Batch file upload
curl -X POST http://localhost:8000/v1/transcribe \
  -H "Authorization: Bearer <api_key>" \
  -F "model=QVox" \
  -F "file=@audio.wav"

# Batch with URL
curl -X POST http://localhost:8000/v1/transcribe \
  -H "Authorization: Bearer <api_key>" \
  -F "model=QVox" \
  -F "url=https://example.com/audio.mp3"

# Batch with language (Hindi)
curl -X POST http://localhost:8000/v1/transcribe \
  -H "Authorization: Bearer <api_key>" \
  -F "model=QVox" \
  -F "file=@audio.wav" \
  -F "lang=hi"
```

> **Note:** Streaming via cURL is not practical since you need to send binary audio chunks in real-time.

---

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `RT_ASR_URL` | Canary batch ASR URL | `http://localhost:9005/transcribe` |
| `RT_ASR_WS_URL` | Canary WebSocket streaming URL | `ws://localhost:9005/ws` |
| `INDIC_ASR_URL` | Indic languages ASR URL | `http://localhost:9002/realtime` |
| `DIARIZATION_URL` | Speaker diarization URL | `http://localhost:9003/diarize` |
| `API_KEYS` | Comma-separated API keys | - |
| `MODEL_NAME` | Valid model name (must be "QVox") | `QVox` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT                                    │
│   (Browser App / Mobile / IoT Device)                            │
└─────────────────────────┬───────────────────────────────────────┘
                          │  HTTPS/WSS
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                   QVox API Gateway (:8000)                       │
│                                                                  │
│  ┌──────────────┐    ┌─────────────────┐    ┌──────────────┐   │
│  │  /v1/stream │    │ /v1/transcribe   │    │  / (health) │   │
│  │  (WebSocket) │    │  (HTTP POST)     │    │              │   │
│  └──────┬───────┘    └────────┬────────┘    └──────────────┘   │
│         │                      │                                   │
│  ┌──────▼──────────────────────▼──────────────────────────────┐   │
│  │              Security Layer (API Key Validation)            │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                         │                                        │
│  ┌──────────────────────▼───────────────────────────────────┐   │
│  │              Routing / Service Layer                      │   │
│  │  - StreamingSession (WebSocket proxy)                     │   │
│  │  - run_pipeline() (batch ASR + Diarization)              │   │
│  │  - download_from_url() (YouTube/URL)                      │   │
│  └──────────────────────┬───────────────────────────────────┘   │
└─────────────────────────┼───────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────────┐
    │ Canary   │   │  Indic   │   │ Diarization  │
    │ /ws/:9005│   │ ASR/:9002│   │  /:9003      │
    │ /transcrb│   │          │   │              │
    └──────────┘   └──────────┘   └──────────────┘
```

---

## Docker Deployment

```bash
# Build
docker compose build

# Run
docker compose up -d

# Logs
docker compose logs -f

# Shell into container
docker compose exec qvox sh
```

**Health check:**
```bash
curl http://localhost:8000/
# Response: {"Status":"QVox--Running....."}
```
