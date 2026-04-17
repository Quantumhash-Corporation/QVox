# QVox API Reference

## Base URL

```
Production:  https://api.education1.uk
Local:       http://localhost:8000
```

---

## Authentication

All endpoints require:

```
Authorization: Bearer <api_key>
```

---

## Endpoints Overview

| Method | Endpoint | Purpose | Latency |
|--------|----------|---------|---------|
| `POST` | `/v1/transcribe` | Batch transcription | ~2-5s |
| `WS` | `/v1/stream` | Real-time streaming | ~150-300ms |

---

## Batch Transcription

```
POST /v1/transcribe
Content-Type: multipart/form-data
```

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | ✅ | Model name (e.g., "QVox") |
| `file` | file | ❌ | Audio file (max 500MB) |
| `url` | string | ❌ | Audio URL |
| `lang` | string | ❌ | Language code (see below) |

> Provide `file` **OR** `url`, not both.

### Supported Languages

```
as, bn, brx, doi, gu, hi, kn, kok, ks, mai, ml, mni, mr, ne, or, pa, sa, sat, sd, ta, te, ur
```

- `lang` not provided → routes to Canary ASR (English)
- `lang` provided + supported → routes to Indic ASR

### Supported Audio Formats

```
WAV, MP3, FLAC, OGG, WebM, M4A, AAC, OPUS
```

### Response

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

### Errors

| Status | Cause |
|--------|-------|
| 400 | No file or URL |
| 400 | Unsupported language |
| 400 | YouTube download failed |
| 401 | Invalid API key |
| 413 | File > 500MB |
| 500 | ASR service error |

### Examples

```bash
# File upload
curl -X POST https://api.education1.uk/v1/transcribe \
  -H "Authorization: Bearer <key>" \
  -F "model=QVox" \
  -F "file=@audio.wav"

# URL input
curl -X POST https://api.education1.uk/v1/transcribe \
  -H "Authorization: Bearer <key>" \
  -F "model=QVox" \
  -F "url=https://example.com/audio.mp3"

# With language hint
curl -X POST https://api.education1.uk/v1/transcribe \
  -H "Authorization: Bearer <key>" \
  -F "model=QVox" \
  -F "file=@audio.wav" \
  -F "lang=hi"
```

---

## Streaming Transcription

Real-time audio streaming via WebSocket. For conversational AI applications.

```
WS /v1/stream
ws://localhost:8000/v1/stream
```

### Protocol

**Client → Server**

| Message | Description |
|---------|-------------|
| Binary frame | Raw audio bytes |
| `{"type":"end"}` | End session |

**Server → Client**

| Message | Description |
|---------|-------------|
| `{"type":"ready"}` | Connection established |
| `{"type":"transcript","text":"..."}` | Transcription result |
| `{"type":"done"}` | Session complete |
| `{"type":"error","message":"..."}` | Error occurred |

### Audio Formats

| Format | Supported | Notes |
|--------|-----------|-------|
| WebM/Opus | ✅ | Browser MediaRecorder default |
| WAV | ✅ | RIFF header required |
| MP3 | ✅ | |
| FLAC | ✅ | |
| Raw PCM | ❌ | Requires WAV header |

### Recommended Chunk Size

```
2-3 seconds of audio per chunk
```

Smaller chunks = lower latency but higher API calls.
Larger chunks = better accuracy but higher latency.

---

## Client Examples

### Python

```python
import asyncio
import websockets
import json

async def stream_transcribe(audio_path: str):
    uri = "ws://localhost:8000/v1/stream"

    async with websockets.connect(uri, ping_timeout=None) as ws:
        await ws.recv()  # {"type":"ready"}

        with open(audio_path, "rb") as f:
            await ws.send(f.read())

        result = await ws.recv()
        data = json.loads(result)

        if data["type"] == "transcript":
            print(f"Text: {data['text']}")

        await ws.send(json.dumps({"type": "end"}))
        await ws.recv()  # {"type":"done"}

asyncio.run(stream_transcribe("audio.wav"))
```

### Python — Real-time Microphone

```python
import asyncio
import websockets
import json
import pyaudio

async def stream_mic(duration=10):
    CHUNK = 1024
    FORMAT = pyaudio.paInt16
    CHANNELS = 1
    RATE = 16000

    uri = "ws://localhost:8000/v1/stream"

    async with websockets.connect(uri, ping_timeout=None) as ws:
        await ws.recv()  # ready

        p = pyaudio.PyAudio()
        stream = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True)

        for _ in range(int(RATE / CHUNK * duration)):
            data = stream.read(CHUNK, exception_on_overflow=False)
            ws.send(data)

        stream.stop_stream()
        p.terminate()

        await ws.send(json.dumps({"type": "end"}))
        await ws.recv()

asyncio.run(stream_mic())
```

### Node.js

```javascript
const WebSocket = require('ws');
const fs = require('fs');

const ws = new WebSocket('ws://localhost:8000/v1/stream');

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

const ws = new WebSocket('ws://localhost:8000/v1/stream');

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

### React.js — Browser MediaRecorder

```jsx
import { useState, useRef } from 'react';

function AudioTranscriber() {
  const [transcript, setTranscript] = useState('');
  const wsRef = useRef(null);
  const recorderRef = useRef(null);

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

    recorder.start(1000);  // chunk every 1s
    recorderRef.current = recorder;

    // Connect WebSocket
    const ws = new WebSocket('ws://localhost:8000/v1/stream');
    ws.onmessage = (e) => {
      const data = JSON.parse(e.data);
      if (data.type === 'transcript') {
        setTranscript(data.text);
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

### Next.js — Socket.IO Proxy

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
    const qvoxWs = new WebSocket('ws://localhost:8000/v1/stream');

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

---

## Health Check

```
GET /
```

Response:

```json
{ "Status": "QVox--Running....." }
```

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `RT_ASR_URL` | Canary batch ASR URL |
| `INDIC_ASR_URL` | Indic languages ASR URL |
| `DIARIZATION_URL` | Speaker diarization URL |
| `API_KEYS` | Comma-separated API keys |
| `EXPECTED_MODEL` | Valid model name |
