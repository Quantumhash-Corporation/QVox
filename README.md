# QVox Client Integration Guide

This guide focuses on **client-side integration** with QVox API. For server details, see the architecture section.

## Quick Reference

| Environment | Base URL | WebSocket |
|-------------|----------|-----------|
| **Production** | `https://api.education1.uk` | `wss://api.education1.uk/v1/stream` |

---

## Authentication

### Batch API (HTTP)
```
Authorization: Bearer <api_key>
```

### Streaming API (WebSocket)
```
wss://api.education1.uk/v1/stream?api_key=<api_key>
```

---

## Streaming Protocol

The WebSocket protocol for streaming transcription:

### Client → Server
| Message | Description |
|---------|-------------|
| Binary audio (16kHz WAV/PCM) | Raw audio bytes per chunk |
| `{"type":"end"}` | End session |

### Server → Client
| Message | Description |
|---------|-------------|
| `{"type": "ready"}` | Connection established |
| `{"type": "transcript", "text": "..."}` | Transcription result |
| `{"type": "done"}` | Session complete |
| `{"type": "error", "message": "..."}` | Error occurred |

### Audio Requirements
| Property | Value |
|----------|-------|
| Sample Rate | 16,000 Hz |
| Bit Depth | 16-bit |
| Channels | 1 (mono) |
| Encoding | PCM (signed int16) or WAV |

### Chunk Size
```
Recommended: 32KB-48KB per chunk (~2-3 seconds of audio)
Smaller = lower latency, Larger = better accuracy per chunk
```

---

## Client Implementations

### Browser - WebSocket Streaming

The most common use case: **browser microphone streaming**.

```javascript
// stream.js - WebSocket service for QVox streaming

let ws = null;
const callbacks = {
  onOpen: null,
  onClose: null,
  onError: null,
  onTranscript: null,
  onDone: null,
};

export function initStreamService(config) {
  if (config.onOpen) callbacks.onOpen = config.onOpen;
  if (config.onClose) callbacks.onClose = config.onClose;
  if (config.onError) callbacks.onError = config.onError;
  if (config.onTranscript) callbacks.onTranscript = config.onTranscript;
  if (config.onDone) callbacks.onDone = config.onDone;
}

export function connect(apiKey) {
  if (ws && ws.readyState === WebSocket.OPEN) return;

  // Configure your backend URL here
  const WS_URL = import.meta.env.VITE_WS_URL || 'ws://localhost:port';
  const wsUrl = `${WS_URL}/v1/stream?api_key=${apiKey}`;
  
  ws = new WebSocket(wsUrl);

  ws.onopen = () => {
    callbacks.onOpen?.();
  };

  ws.onclose = () => {
    callbacks.onClose?.();
  };

  ws.onerror = (error) => {
    callbacks.onError?.(new Error("WebSocket error"));
  };

  ws.onmessage = (event) => {
    try {
      const msg = JSON.parse(event.data);
      
      switch (msg.type) {
        case 'transcript':
          callbacks.onTranscript?.(msg.text, msg);
          break;
        case 'done':
          callbacks.onDone?.();
          ws.close();
          break;
        case 'error':
          callbacks.onError?.(new Error(msg.message));
          break;
      }
    } catch (err) {
      console.error('Failed to parse message:', err);
    }
  };
}

export function sendAudioChunk(chunk) {
  if (ws && ws.readyState === WebSocket.OPEN) {
    ws.send(chunk);
    return true;
  }
  return false;
}

export function disconnect() {
  if (ws) {
    ws.send(JSON.stringify({ type: 'end' }));
    ws.close();
    ws = null;
  }
}

export function isConnected() {
  return ws && ws.readyState === WebSocket.OPEN;
}
```

### Browser - MediaRecorder Streaming

```javascript
// Start recording with microphone
const stream = await navigator.mediaDevices.getUserMedia({
  audio: {
    deviceId: selectedDevice !== 'default' ? { exact: selectedDevice } : undefined,
    echoCancellation: true,
    noiseSuppression: true,
    autoGainControl: true,
  }
});

// Create MediaRecorder (WebM format)
const mediaRecorder = new MediaRecorder(stream, {
  mimeType: 'audio/webm;codecs=opus'
});

const audioChunks = [];

mediaRecorder.ondataavailable = (e) => {
  if (e.data.size > 0) {
    audioChunks.push(e.data);
    sendAudioChunk(e.data);  // Send to WebSocket
  }
};

mediaRecorder.start(1000);  // Collect chunks every 1 second

// Stop recording
mediaRecorder.stop();
stream.getTracks().forEach(track => track.stop());
disconnect();  // Send end signal
```

### React.js - Live Stream Component

```jsx
import { useState, useRef, useEffect } from 'react';
import { connect, disconnect, sendAudioChunk, initStreamService, isConnected } from './stream';

export default function LiveStream() {
  const [transcript, setTranscript] = useState('');
  const [isRecording, setIsRecording] = useState(false);
  const mediaRecorderRef = useRef(null);
  const streamRef = useRef(null);

  useEffect(() => {
    // Initialize stream callbacks
    initStreamService({
      onOpen: () => console.log('Connected'),
      onClose: () => setIsRecording(false),
      onTranscript: (text) => {
        setTranscript(prev => prev + ' ' + text);
      },
      onDone: () => console.log('Stream ended'),
      onError: (err) => console.error('Error:', err),
    });

    return () => disconnect();
  }, []);

  const startRecording = async () => {
    // Connect to WebSocket first
    connect(apiKey);
    
    // Wait for connection
    await new Promise(resolve => setTimeout(resolve, 500));
    
    if (!isConnected()) {
      console.error('Failed to connect');
      return;
    }

    // Get microphone
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    streamRef.current = stream;

    // Create MediaRecorder
    const recorder = new MediaRecorder(stream, { mimeType: 'audio/webm;codecs=opus' });
    
    recorder.ondataavailable = (e) => {
      if (e.data.size > 0) {
        sendAudioChunk(e.data);
      }
    };

    recorder.start(1000);
    mediaRecorderRef.current = recorder;
    setIsRecording(true);
  };

  const stopRecording = () => {
    mediaRecorderRef.current?.stop();
    streamRef.current?.getTracks().forEach(track => track.stop());
    disconnect();
    setIsRecording(false);
  };

  return (
    <div>
      <button onClick={isRecording ? stopRecording : startRecording}>
        {isRecording ? 'Stop' : 'Start Recording'}
      </button>
      <div className="transcript">{transcript}</div>
    </div>
  );
}
```

### Node.js - Streaming

```javascript
const WebSocket = require('ws');
const fs = require('fs');

const apiKey = '<your_api_key>';
const wsUrl = `wss://api.education1.uk/v1/stream?api_key=${apiKey}`;

const ws = new WebSocket(wsUrl);

ws.on('open', () => {
  console.log('Connected to QVox');
  
  // Send audio file in chunks
  const chunkSize = 32000;
  const audio = fs.readFileSync('audio.wav');
  
  let offset = 0;
  const sendChunk = () => {
    if (offset >= audio.length) {
      ws.send(JSON.stringify({ type: 'end' }));
      return;
    }
    
    const chunk = audio.slice(offset, offset + chunkSize);
    ws.send(chunk);
    offset += chunkSize;
    
    setTimeout(sendChunk, 100);  // ~100ms between chunks
  };
  
  sendChunk();
});

ws.on('message', (data) => {
  const msg = JSON.parse(data.toString());
  
  if (msg.type === 'transcript') {
    console.log('Transcript:', msg.text);
  } else if (msg.type === 'done') {
    console.log('Done');
    ws.close();
  }
});

ws.on('error', (err) => {
  console.error('WebSocket error:', err);
});
```

### Python - Streaming

```python
import asyncio
import websockets
import json

async def stream_audio(api_key: str, audio_path: str):
    uri = f"wss://api.education1.uk/v1/stream?api_key={api_key}"
    
    async with websockets.connect(uri) as ws:
        # Wait for ready
        await ws.recv()
        
        # Send audio chunks
        with open(audio_path, "rb") as f:
            while chunk := f.read(32000):
                await ws.send(chunk)
                
                # Receive transcript
                msg = await ws.recv()
                data = json.loads(msg)
                if data.get('type') == 'transcript':
                    print(f"Transcript: {data['text']}")
        
        # End session
        await ws.send(json.dumps({"type": "end"}))
        await ws.recv()  # done

asyncio.run(stream_audio("<api_key>", "audio.wav"))
```

### Python - Microphone Streaming

```python
import asyncio
import websockets
import json
import pyaudio

async def stream_mic(api_key: str, duration_seconds: int):
    CHUNK = 32000  # ~2 seconds
    RATE = 16000
    
    uri = f"wss://api.education1.uk/v1/stream?api_key={api_key}"
    
    async with websockets.connect(uri) as ws:
        await ws.recv()  # ready
        
        p = pyaudio.PyAudio()
        stream = p.open(format=pyaudio.paInt16, channels=1, rate=RATE, input=True)
        
        for _ in range(int(RATE / CHUNK * duration_seconds)):
            data = stream.read(CHUNK, exception_on_overflow=False)
            await ws.send(data)
            
            try:
                msg = await asyncio.wait_for(ws.recv(), timeout=5)
                data = json.loads(msg)
                if data.get('type') == 'transcript':
                    print(data['text'])
            except asyncio.TimeoutError:
                pass
        
        stream.stop_stream()
        p.terminate()
        
        await ws.send(json.dumps({"type": "end"}))
        await ws.recv()

asyncio.run(stream_mic("<api_key>", 10))
```

---

## Batch Transcription (HTTP)

### cURL

```bash
# File upload
curl -X POST https://api.education1.uk/v1/transcribe \
  -H "Authorization: Bearer <api_key>" \
  -F "model=QVox" \
  -F "file=@audio.wav"

# With language (Hindi)
curl -X POST https://api.education1.uk/v1/transcribe \
  -H "Authorization: Bearer <api_key>" \
  -F "model=QVox" \
  -F "file=@audio.wav" \
  -F "lang=hi"
```

### Python

```python
import requests

url = "https://api.education1.uk/v1/transcribe"
headers = {"Authorization": "Bearer <api_key>"}

# File upload
with open("audio.wav", "rb") as f:
    response = requests.post(url, headers=headers, files={"file": f}, data={"model": "QVox"})

print(response.json())
```

### Node.js

```javascript
const FormData = require('form-data');
const fs = require('fs');
const axios = require('axios');

const url = 'https://api.education1.uk/v1/transcribe';
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
```

---

## Supported Languages

For languages, use `lang` parameter:

```
as, bn, brx, doi, gu, hi, kn, kok, ks, mai, ml, mni, mr, ne, or, pa, sa, sat, sd, ta, te, ur
```

**Without `lang`**: English ASR + Speaker Diarization (returns speaker labels)
**With `lang`**: ASR only (no diarization)

---

## Troubleshooting

### Connection Issues

**"WebSocket connection failed"**
- Check if API key is valid
- Verify network allows WebSocket connections

**"Invalid or missing API key"**
- API key not provided in query param
- API key not in allowed keys list

### Audio Issues

**No transcript returned**
- Ensure audio is 16kHz, 16-bit, mono
- Check microphone permissions in browser
- Verify audio chunks are being sent (check browser console logs)

**Poor transcription quality**
- Use larger chunks (48KB instead of 32KB)
- Ensure clean audio input
- Check for background noise

### Protocol Flow

```
1. Client connects WS → Server sends {"type": "ready"}
2. Client sends audio bytes → Server processes → sends {"type": "transcript", "text": "..."}
3. Client sends {"type": "end"} → Server sends {"type": "done"}
```

Make sure your client handles all message types: `ready`, `transcript`, `done`, `error`.
