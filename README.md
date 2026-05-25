# practice

# 실시간 영상 송출 및 재생 (macOS)

VLC로 띄운 RTSP 스트림을 FFmpeg으로 HLS로 변환하고,
Express로 서빙해 React 앱에서 재생하는 전 과정을 정리합니다.

```
[VLC: RTSP 송출] ──▶ [FFmpeg: HLS 변환] ──▶ [Express: HLS 서빙] ──▶ [React: 재생]
   rtsp://...           .m3u8 + .ts          http://localhost:4000        <video />
```

## 목차

- [VLC로 RTSP 스트림 송출](#vlc로-rtsp-스트림-송출)
- [FFmpeg으로 HLS 변환](#ffmpeg으로-hls-변환)
- [Express 서버로 HLS 파일 서빙](#express-서버로-hls-파일-서빙)
- [React에서 영상 재생](#react에서-영상-재생)

<br />

## 사전 준비

```bash
# VLC: 공식 사이트에서 설치 (https://www.videolan.org/)
brew install --cask vlc

# FFmpeg
brew install ffmpeg

# (선택) GStreamer — WebRTC 변환을 시도할 때
brew install gstreamer
```

> macOS에서 VLC를 CLI로 실행하려면 `.app` 번들 내부의 바이너리(`/Applications/VLC.app/Contents/MacOS/VLC`)를 직접 호출해야 합니다.

<br />
<br />

## VLC로 RTSP 스트림 송출

로컬 mp4 파일을 RTSP 스트림으로 무한 송출합니다.
실시간 카메라가 없어도 RTSP 클라이언트 동작을 테스트할 수 있어 유용합니다.

```bash
/Applications/VLC.app/Contents/MacOS/VLC ~/Downloads/test.mp4 \
  --rtsp-host=127.0.0.1 \
  --sout '#transcode{vcodec=h264,vb=800,scale=1,acodec=mp3,ab=128,channels=2,samplerate=44100}:rtp{sdp=rtsp://127.0.0.1:8554/stream}' \
  --sout-all \
  --sout-keep \
  --loop
```

### 옵션 풀이

| 옵션 | 설명 |
| --- | --- |
| `--rtsp-host=127.0.0.1` | RTSP 서버가 바인딩할 호스트 주소 |
| `--sout '#transcode{...}:rtp{...}'` | 트랜스코딩 + RTP 출력 체인 정의 |
| `vcodec=h264, vb=800` | 비디오 코덱 H.264, 비트레이트 800kbps |
| `acodec=mp3, ab=128` | 오디오 코덱 MP3, 비트레이트 128kbps |
| `sdp=rtsp://127.0.0.1:8554/stream` | 클라이언트가 접속할 RTSP URL |
| `--sout-all` | 비디오·오디오 등 모든 elementary stream 출력 |
| `--sout-keep` | 입력이 끝나도 출력 스트림 유지 (loop와 짝꿍) |
| `--loop` | 입력 파일 무한 반복 |

### 송출 확인

다른 터미널에서 ffprobe로 스트림이 살아있는지 확인합니다.

```bash
ffprobe rtsp://127.0.0.1:8554/stream
```

또는 VLC로 직접 열어볼 수도 있습니다.

```bash
/Applications/VLC.app/Contents/MacOS/VLC rtsp://127.0.0.1:8554/stream
```

> **자주 발생하는 문제**
> - 8554 포트가 이미 사용 중이면 `--rtsp-port=8555` 같은 식으로 포트를 변경하세요.
> - RTSP 송출 자체가 안 된다면 VLC의 *Preferences → Show All → Stream output → RTSP*에서 포트/주소가 충돌하지 않는지 확인합니다.

<br />
<br />

## FFmpeg으로 HLS 변환

브라우저는 RTSP를 직접 재생할 수 없습니다.
따라서 RTSP를 HTTP 기반 스트리밍 프로토콜로 바꿔야 하며,
이 글에서는 가장 널리 쓰이는 **HLS(HTTP Live Streaming)** 로 변환합니다.

### 1. FFmpeg을 사용한 HLS 변환

```bash
mkdir -p ./hls
ffmpeg -i rtsp://127.0.0.1:8554/stream \
  -c:v copy -c:a aac \
  -f hls \
  -hls_time 5 \
  -hls_list_size 3 \
  -hls_flags delete_segments \
  ./hls/output.m3u8
```

| 옵션 | 설명 |
| --- | --- |
| `-i ${입력소스}` | 입력 소스 (여기서는 VLC가 띄운 RTSP URL) |
| `-c:v copy` | 비디오는 재인코딩 없이 그대로 사용 (CPU 절약) |
| `-c:a aac` | 오디오는 HLS 호환을 위해 AAC로 변환 |
| `-f hls` | 출력 포맷을 HLS로 지정 — 영상을 짧은 조각(`.ts`)으로 나눠 스트리밍 |
| `-hls_time 5` | 세그먼트 길이 5초 → 5초마다 `.ts` 파일 생성 |
| `-hls_list_size 3` | 재생 목록에 유지할 세그먼트 수 (최근 3개만) |
| `-hls_flags delete_segments` | 목록에서 빠진 세그먼트 파일을 디스크에서도 자동 삭제 |
| `./hls/output.m3u8` | 출력될 플레이리스트 파일 경로 |

### 생성되는 파일

```
./hls
├── output.m3u8      # 재생목록 (어떤 .ts를 어떤 순서로 재생할지)
├── output0.ts
├── output1.ts
└── output2.ts       # 최신 3개만 유지됨
```

`.m3u8`는 텍스트로 된 인덱스 파일이고, 실제 영상 데이터는 `.ts` 세그먼트에 담깁니다.
플레이어는 `.m3u8`을 주기적으로 다시 받아오면서 새 세그먼트를 이어붙여 재생합니다.

> **지연 시간 줄이기.** `-hls_time`을 2초 이하로 낮추고 `-hls_list_size`도 함께 줄이면 지연을 줄일 수 있지만, 네트워크 환경에 따라 끊김이 늘어날 수 있습니다.

### 2. GStreamer를 사용한 RTSP → WebRTC 변환 (참고)

지연 시간이 중요한 시나리오에서는 WebRTC가 HLS보다 유리합니다.
GStreamer의 `webrtcbin` 엘리먼트로 RTSP 소스를 WebRTC 트랙에 연결할 수 있습니다.

```bash
gst-launch-1.0 rtspsrc location=rtsp://127.0.0.1:8554/stream ! \
  rtph264depay ! h264parse ! ...
```

> WebRTC는 시그널링 서버(보통 WebSocket) 구현이 추가로 필요하므로, 이번 실습에서는 HLS만 다루고 WebRTC는 별도 주제로 분리합니다.

<br />
<br />

## Express 서버로 HLS 파일 서빙

브라우저(React 앱)에서 `.m3u8`을 fetch하려면 HTTP로 노출해야 합니다.
간단한 Express 서버로 `./hls` 디렉토리를 정적 파일로 서빙합니다.

```js
// server.js
const express = require('express');
const path = require('path');
const cors = require('cors');

const app = express();
const PORT = 4000;

// React 앱이 다른 포트(예: 3000)에서 실행되므로 CORS 허용 필수
app.use(cors());

// ./hls 디렉토리를 정적 파일 루트로 설정
//   → http://localhost:4000/output.m3u8 로 접근 가능
app.use(express.static(path.join(__dirname, 'hls')));

app.listen(PORT, () => {
  console.log(`HLS server running at http://localhost:${PORT}`);
});
```

실행:

```bash
npm install express cors
node server.js
```

접근 확인:

```bash
curl -I http://localhost:4000/output.m3u8
# HTTP/1.1 200 OK  가 떨어지면 성공
```

> **캐싱 주의.** HLS는 `.m3u8`이 계속 갱신되어야 합니다.
> Express의 `express.static`은 기본적으로 캐시를 짧게 잡지만,
> 운영 환경에서는 명시적으로 `Cache-Control: no-cache`를 `.m3u8`에 지정하는 것이 안전합니다.

```js
app.use(express.static(path.join(__dirname, 'hls'), {
  setHeaders: (res, filePath) => {
    if (filePath.endsWith('.m3u8')) {
      res.setHeader('Cache-Control', 'no-cache');
    }
  },
}));
```

<br />
<br />

## React에서 영상 재생

실시간 스트리밍 재생에는 보통 **HLS** 또는 **WebRTC** 를 사용합니다.
여기서는 HLS 재생만 다룹니다.

### 1. HLS 재생

Safari는 `<video>` 태그에 `.m3u8` URL을 그대로 넣으면 재생됩니다.
하지만 **Chrome / Firefox는 네이티브로 HLS를 지원하지 않으므로**, [hls.js](https://github.com/video-dev/hls.js) 같은 라이브러리가 필요합니다.

설치:

```bash
npm install hls.js
```

컴포넌트:

```jsx
import { useEffect, useRef } from 'react';
import Hls from 'hls.js';

const HLS_URL = 'http://localhost:4000/output.m3u8';

export default function LiveVideo() {
  const videoRef = useRef(null);

  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    // Safari: 네이티브 HLS 지원
    if (video.canPlayType('application/vnd.apple.mpegurl')) {
      video.src = HLS_URL;
      return;
    }

    // Chrome / Firefox 등: hls.js로 폴백
    if (Hls.isSupported()) {
      const hls = new Hls({ lowLatencyMode: true });
      hls.loadSource(HLS_URL);
      hls.attachMedia(video);

      hls.on(Hls.Events.ERROR, (_, data) => {
        console.error('HLS error:', data);
      });

      return () => hls.destroy();
    }
  }, []);

  return (
    <video
      ref={videoRef}
      autoPlay
      muted          // 모바일/브라우저 자동재생 정책 회피
      playsInline
      controls
      style={{ width: '100%' }}
    />
  );
}
```

### 동작 흐름 정리

```
VLC (test.mp4 → RTSP 송출)
    │  rtsp://127.0.0.1:8554/stream
    ▼
FFmpeg (RTSP → HLS 변환)
    │  ./hls/output.m3u8 + output*.ts
    ▼
Express (./hls 정적 서빙, CORS 허용)
    │  http://localhost:4000/output.m3u8
    ▼
React + hls.js (<video>로 재생)
```

> **참고: 지연(latency)**
> HLS는 세그먼트 단위로 동작하기 때문에 본질적으로 수초 ~ 수십 초의 지연이 발생합니다.
> 보안 카메라처럼 “지금 이 순간”이 중요한 시나리오에는 WebRTC가 더 적합하며,
> VOD/생방송처럼 1~3초 지연을 허용할 수 있는 경우에는 HLS가 안정적이고 호환성이 좋습니다.

### 2. WebRTC 재생 (별도 주제)

WebRTC는 시그널링 서버, ICE/STUN/TURN 설정 등 추가 작업이 필요하므로
별도 문서에서 다룰 예정입니다.
