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
ws://host/v1/stream?api_key=<api_key>
```

---

## Endpoints Overview

| Method | Endpoint | Auth | Purpose | Latency |
|--------|----------|------|---------|---------|
| `POST` | `/v1/transcribe` | ✅ Bearer header | Batch transcription | ~2-5s |
| `WS` | `/v1/stream` | ✅ Query param | Real-time streaming | ~150-300ms |
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

**Supported Audio Formats:**
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

Real-time streaming transcription via WebSocket.

**Connection:**
```
ws://host/v1/stream?api_key=<api_key>
```

### Protocol

**Client → Server:**
| Message | Description |
|---------|-------------|
| Binary audio | Raw audio bytes (any format) |
| `{"type":"end"}` | End session |

**Server → Client:**
| Message | Description |
|---------|-------------|
| `{"type": "ready"}` | Connection established, auth passed |
| `{"type": "transcript", "text": "..."}` | Transcription result |
| `{"type": "done"}` | Session complete |
| `{"type": "error", "message": "..."}` | Error occurred |

### Audio Format Detection (Magic Bytes)

| Format | Magic Bytes | Notes |
|--------|-------------|-------|
| WAV | `RIFF....WAVE` | |
| MP3 | `ID3` or `\xff\xfb` | Sync word |
| WebM | `\x1aE\xdf\xa3` | Browser default |
| FLAC | `fLaC` | |
| OGG | `OggS` | |

### Recommended Chunk Size

```
2-3 seconds of audio per chunk
```

Smaller chunks = lower latency
Larger chunks = better accuracy

---

## Client Examples

### Python — Batch Transcription

```python
import requests

url = "http://localhost:8000/v1/transcribe"
headers = {"Authorization": "Bearer <api_key>"}

# File upload
with open("audio.wav", "rb") as f:
    response = requests.post(url, headers=headers, files={"file": f}, data={"model": "QVox"})

print(response.json())

# With language hint (Hindi)
with open("audio.wav", "rb") as f:
    response = requests.post(url, headers=headers, files={"file": f}, data={"model": "QVox", "lang": "hi"})
```

### Python — Streaming

```python
import asyncio
import websockets
import json

async def stream_transcribe(audio_path: str, api_key: str):
    uri = f"ws://localhost:8000/v1/stream?api_key={api_key}"

    async with websockets.connect(uri, ping_timeout=None) as ws:
        await ws.recv()  # {"type":"ready"}

        with open(audio_path, "rb") as f:
            await ws.send(f.read())

        result = await ws.recv()
        data = json.loads(result)
        print(f"Transcript: {data['text']}")

        await ws.send(json.dumps({"type": "end"}))
        await ws.recv()  # {"type":"done"}

asyncio.run(stream_transcribe("audio.wav", "<api_key>"))
```

### Python — Real-time Microphone

```python
import asyncio
import websockets
import json
import pyaudio

async def stream_mic(duration: int, api_key: str):
    CHUNK = 1024
    FORMAT = pyaudio.paInt16
    CHANNELS = 1
    RATE = 16000

    uri = f"ws://localhost:8000/v1/stream?api_key={api_key}"

    async with websockets.connect(uri, ping_timeout=None) as ws:
        await ws.recv()

        p = pyaudio.PyAudio()
        stream = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True)

        for _ in range(int(RATE / CHUNK * duration)):
            data = stream.read(CHUNK, exception_on_overflow=False)
            await ws.send(data)

        stream.stop_stream()
        p.terminate()

        await ws.send(json.dumps({"type": "end"}))
        await ws.recv()

asyncio.run(stream_mic(10, "<api_key>"))
```

### Node.js — Batch Transcription

```javascript
const FormData = require('form-data');
const fs = require('fs');
const axios = require('axios');

const apiKey = '<api_key>';
const url = 'http://localhost:8000/v1/transcribe';

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

### Node.js — Streaming

```javascript
const WebSocket = require('ws');
const fs = require('fs');

const apiKey = '<api_key>';
const ws = new WebSocket(`ws://localhost:8000/v1/stream?api_key=${apiKey}`);

ws.on('open', () => {
    const audio = fs.readFileSync('audio.wav');
    ws.send(audio);
});

ws.on('message', (data) => {
    const msg = JSON.parse(data.toString());

    if (msg.type === 'transcript') {
        console.log('Text:', msg.text);
    } else if (msg.type === 'done') {
        ws.send(JSON.stringify({ type: 'end' }));
        ws.close();
    }
});
```

### Node.js — Microphone Streaming

```javascript
const WebSocket = require('ws');
const { MicrophoneStream } = require('microphone-stream');

const apiKey = '<api_key>';
const ws = new WebSocket(`ws://localhost:8000/v1/stream?api_key=${apiKey}`);

ws.on('open', async () => {
    const mic = new MicrophoneStream();

    mic.on('data', (chunk) => {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(Buffer.from(chunk));
        }
    });

    await mic.start({ sampleRate: 16000, channels: 1 });
    console.log('Mic active');
});

ws.on('message', (data) => {
    const msg = JSON.parse(data.toString());
    if (msg.type === 'transcript') {
        console.log('Text:', msg.text);
    }
});
```

### React.js — Batch Transcription

```jsx
import axios from 'axios';

const apiKey = '<api_key>';
const url = 'http://localhost:8000/v1/transcribe';

async function transcribeFile(file, lang = null) {
    const formData = new FormData();
    formData.append('model', 'QVox');
    formData.append('file', file);
    if (lang) formData.append('lang', lang);

    const response = await axios.post(url, formData, {
        headers: { 'Authorization': `Bearer ${apiKey}` }
    });

    return response.data;
}

// Usage
const result = await transcribeFile(audioFile);      // English
const resultHi = await transcribeFile(audioFile, 'hi'); // Hindi
```

### React.js — Browser Streaming

```jsx
import { useState, useRef } from 'react';

function AudioTranscriber() {
  const [transcript, setTranscript] = useState('');
  const wsRef = useRef(null);
  const recorderRef = useRef(null);
  const apiKey = '<api_key>';

  const startRecording = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    const recorder = new MediaRecorder(stream, {
      mimeType: 'audio/webm; codecs=opus'
    });

    recorder.ondataavailable = (e) => {
      if (e.data.size > 0 && wsRef.current?.OPEN) {
        wsRef.current.send(e.data);
      }
    };

    recorder.start(1000);
    recorderRef.current = recorder;

    const ws = new WebSocket(`ws://localhost:8000/v1/stream?api_key=${apiKey}`);
    ws.onmessage = (e) => {
      const data = JSON.parse(e.data);
      if (data.type === 'transcript') {
        setTranscript(prev => prev + ' ' + data.text);
      }
    };
    wsRef.current = ws;
  };

  const stopRecording = () => {
    recorderRef.current?.stop();
    wsRef.current?.send(JSON.stringify({ type: 'end' }));
  };

  return (
    <div>
      <button onClick={startRecording}>Start</button>
      <button onClick={stopRecording}>Stop</button>
      <p>{transcript}</p>
    </div>
  );
}
```

### Next.js — Batch Transcription (API Route)

```typescript
// app/api/transcribe/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
    const apiKey = process.env.QVOX_API_KEY;
    const formData = await request.formData();
    formData.append('model', 'QVox');

    const response = await fetch('http://localhost:8000/v1/transcribe', {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${apiKey}` },
        body: formData
    });

    return NextResponse.json(await response.json());
}
```

### Next.js — Streaming with Socket.IO

**Server (server.js):**

```javascript
const { createServer } = require('http');
const next = require('next');
const { Server } = require('socket.io');

const dev = process.env.NODE_ENV !== 'production';
const app = next({ dev });
const handle = app.getRequestHandler();

app.prepare().then(() => {
  const server = createServer();
  const io = new Server(server, { cors: { origin: '*' } });

  io.on('connection', (socket) => {
    const WebSocket = require('ws');
    const apiKey = process.env.QVOX_API_KEY;
    const qvoxWs = new WebSocket(`ws://localhost:8000/v1/stream?api_key=${apiKey}`);

    qvoxWs.on('message', (data) => socket.emit('transcript', data.toString()));

    socket.on('audio', (audioData) => qvoxWs.send(audioData));
    socket.on('end', () => qvoxWs.send(JSON.stringify({ type: 'end' })));
    socket.on('disconnect', () => qvoxWs.close());
  });

  server.listen(3000);
});
```

**Client (Next.js page):**

```tsx
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

export default function Home() {
  const [transcript, setTranscript] = useState('');
  const [socket, setSocket] = useState(null);

  useEffect(() => {
    const s = io('http://localhost:3000');
    s.on('transcript', (data) => {
      const msg = JSON.parse(data);
      if (msg.type === 'transcript') setTranscript(msg.text);
    });
    setSocket(s);
    return () => s.disconnect();
  }, []);

  const startRecording = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    const recorder = new MediaRecorder(stream, { mimeType: 'audio/webm' });
    recorder.ondataavailable = (e) => socket?.emit('audio', e.data);
    recorder.start(1000);
  };

  return (
    <div>
      <button onClick={startRecording}>Start</button>
      <p>{transcript}</p>
    </div>
  );
}
```

### cURL

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

---

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `RT_ASR_URL` | Canary batch ASR URL | `http://localhost:9005/transcribe` |
| `INDIC_ASR_URL` | Indic languages ASR URL | `http://localhost:9002/realtime` |
| `DIARIZATION_URL` | Speaker diarization URL | `http://localhost:9003/diarize` |
| `API_KEYS` | Comma-separated API keys | - |
| `MODEL_NAME` | Valid model name (must be "QVox") | `QVox` |

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