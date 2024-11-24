# practice



# 실시간 영상 송출 및 재생

- VLC를 사용하여 로컬에서 RTSP 스트림을 송출
- FFmpeg로 RTSP 스트림을 HLS 포맷으로 변환
- HTTP 서버 제공: Express를 사용해 변환된 HLS 파일을 HTTP로 서빙
- React에서 변환된 HLS 스트림을 재생



### VLC로 RTSP 스트림 송출

```bash
 /Applications/VLC.app/Contents/MacOS/VLC ~/Downloads/test.mp4 \
--rtsp-host=127.0.0.1 \
--sout '#transcode{vcodec=h264,vb=800,scale=1,acodec=mp3,ab=128,channels=2,samplerate=44100}:rtp{sdp=rtsp://127.0.0.1:8554/stream}' \
--sout-all \
--sout-keep \
--loop
```

### FFmpeg으로 변환
RTSP를 브라우저에서 바로 재생할 수 없으므로 HTTP 스트리밍으로 변환

1. <b>FFmpeg를 사용한 변환 (HSL 재생)</b>

```bash
ffmpeg -i rtsp://127.0.0.1:8554/stream -f hls -hls_time 5 -hls_list_size 3 -hls_flags delete_segments ./hls/output.m3u8
```
- `-i {입력소스}` : 입력 소스 지정
- `-f hls`: 출력 포맷을 HLS로 지정 (스트리밍 프로토콜로 영상을 작은 조각 (.ts파일들)로 나눠 스트리밍
- `-hls_time 5`: 세그먼트는 5초단위 -> 5초 단위로 .ts파일 생성
- `-hls_list_size 3` : 세그먼트수 3개로 제한
- `-hls_flags delete_segments`: 재생목록에서 제거된 세그먼트 파일을 디스크에서도 자동으로 삭제
- `./hls/output.m3u8`: 출력 파일 저장경로


<br/>
<br/>
<br/>


2. <b>GStreamer를 사용하여 RTSP -> WebRTC 변환</b>


###  Express 서버를 활용하여 HLS 파일을 서빙

- React 앱에서 .m3u8 파일을 재생하기위해 Express 서버를 활용하여 HLS 파일을 서빙

```js

const express = require('express');
const path = require('path');
const cors = require('cors');

const app = express();
const PORT = 4000;

// CORS 허용 설정
app.use(cors());

// 현재 디렉토리를 정적 파일 경로로 설정
app.use(express.static(path.join(__dirname, 'hls')));
// 서버 실행
app.listen(PORT, () => {
  console.log(`Server is running at http://localhost:${PORT}`);
});
```


### React 에서 HLS로 영상 재생
- 리액트에서는 실시간 스트리밍을 위해 HLS, WebRtc를 사용하여 재생한다.

1. HLS 재생

   


