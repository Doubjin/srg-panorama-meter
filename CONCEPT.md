# Web Loudness Meter — Hero Page Concept
### Inspired by TC Electronic Clarity M Stereo

---

## 1. 프로젝트 개요

TC Electronic Clarity M Stereo의 핵심 기능을 브라우저에서 구현하는 **Web Audio API 기반 라우드니스 미터 히어로 페이지**.
실시간 LUFS 측정, 원형 클락와이즈 히스토리, 오디오 디바이스 입출력 선택이 핵심 UX다.

---

## 2. 핵심 기능 정의

### 2-1. 라우드니스 측정 (LUFS Meter)
| 지표 | 설명 | 기준 |
|------|------|------|
| **Momentary LUFS** | 400ms 윈도우 순간 라우드니스 | ITU-R BS.1770-4 |
| **Short-term LUFS** | 3초 윈도우 단기 라우드니스 | ITU-R BS.1770-4 |
| **Integrated LUFS** | 측정 시작부터 전체 평균 | ITU-R BS.1770-4 |
| **LRA** | Loudness Range — 다이나믹 레인지 | EBU R128 |
| **True Peak** | 인터샘플 피크 (dBTP) | ITU-R BS.1770-4 |

### 2-2. 원형 클락와이즈 LUFS 히스토리
- 시계 형태의 **원형 레이더 차트**로 LUFS 이력 기록
- 12시 방향이 측정 시작점, 시계 방향으로 시간 흐름 표시
- 색상 그라디언트: 그린(-23 LUFS 이하) → 옐로우(-18~-14) → 레드(-14 이상)
- 중심에서 외곽으로 갈수록 라우드니스 레벨 증가
- 최근 60초 / 5분 / 전체 3가지 뷰 모드

### 2-3. 오디오 파일 입력 (File Upload / Drag & Drop)
- **파일 업로드**: MP3, WAV, FLAC, AAC 등 브라우저 지원 포맷
- **Drag & Drop**: 화면 전체에 파일을 드래그하여 즉시 로드
- **재생 컨트롤**:
  - Play / Pause / Stop
  - Seek Bar (진행 바) & 타임스탬프 (00:00 / 03:45)
  - Loop 재생 옵션

### 2-4. 오디오 출력 설정 패널
- **모니터링**: 업로드된 오디오를 브라우저 기본 출력으로 재생
- **모니터 볼륨** 슬라이더 (0 ~ -∞ dB)

---

## 3. 화면 구성 (Hero Page Layout)

```
┌─────────────────────────────────────────────────────────────┐
│  HEADER: 로고 + 파일 정보 (파일명, 포맷, 길이)               │
├────────────────────┬────────────────────────────────────────┤
│                    │                                        │
│   [ 원형 LUFS     │   LUFS 수치 패널                       │
│     클락와이즈    │   ┌─────────────────────────┐          │
│     히스토리     ]│   │  Momentary   -14.2 LUFS │          │
│                    │   │  Short-term  -16.8 LUFS │          │
│    ◉ 중심에        │   │  Integrated  -18.1 LUFS │          │
│    현재 LUFS 표시  │   │  LRA          4.2 LU   │          │
│                    │   │  True Peak   -1.0 dBTP  │          │
│                    │   └─────────────────────────┘          │
│                    │                                        │
│                    │   타겟 기준선 선택 (라디오)             │
│                    │   ○ Streaming (-14 LUFS)               │
│                    │   ● Broadcast (-23 LUFS)               │
│                    │   ○ CD (-9 LUFS)                       │
├────────────────────┴────────────────────────────────────────┤
│  BOTTOM PANEL: 플레이어 컨트롤 & 파일 드롭존                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ [DRAG AUDIO FILE HERE OR CLICK TO UPLOAD]             │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌──────────┐ ┌──────────────────────────────────────┐      │
│  │ [▶] [II] │ │ [ WAVEFORM CANVAS (Interactive)    ] │      │
│  │ [■]      │ │ 00:45 / 03:20                        │      │
│  └──────────┘ └──────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 기술 스택

| 영역 | 기술 |
|------|------|
| **오디오 소스** | `AudioBufferSourceNode` (파일 디코딩) |
| **오디오 처리** | Web Audio API (`AudioWorkletProcessor`) |
| **LUFS 계산** | ITU-R BS.1770-4 K-weighting 필터 + 게이팅 |
| **시각화** | Canvas API 또는 SVG (원형 클락와이즈) |
| **UI 프레임워크** | Vanilla HTML/CSS/JS |

---

## 5. 오디오 처리 파이프라인

```
[File Input / Drag & Drop]
        ↓
[File Reader -> ArrayBuffer]
        ↓
[AudioContext.decodeAudioData()]
        ↓
[AudioBufferSourceNode] ──▶ [GainNode (Monitor Volume)] ──▶ [AudioDestinationNode (Speakers)]
        ↓
[AudioWorkletNode: LUFS Processor]
    ├─ K-weighting 필터 적용
    ├─ Mean Square 계산
    └─ LUFS / LRA / True Peak → Main Thread 전달
```

---

## 6. K-Weighting 필터 계수 (48kHz 기준)

```
Stage 1 Pre-filter (High-shelf):
  b = [1.53512485958697, -2.69169618940638, 1.19839281085285]
  a = [1.0, -1.69065929318241, 0.73248077421585]

Stage 2 RLB High-pass:
  b = [1.0, -2.0, 1.0]
  a = [1.0, -1.99004745483398, 0.99007225036621]
```

---

## 7. 원형 클락와이즈 비주얼 설계

### 렌더링 로직
```
- 전체 원 = 오디오 트랙 전체 길이 (예: 3분 20초 = 360도)
- 현재 재생 위치 = 라우드니스 히스토리 작성 위치 (헤드)
- 반지름 = LUFS 레벨 매핑 (-60 LUFS → 중심, 0 LUFS → 외곽)
```

### 색상 코드 기준
| LUFS 범위 | 색상 | HEX |
|-----------|------|-----|
| -60 ~ -24 | 딥 그린 | `#00C851` |
| -24 ~ -18 | 그린 | `#76FF03` |
| -18 ~ -14 | 옐로우 | `#FFD600` |
| -14 ~ -9  | 오렌지 | `#FF6D00` |
| -9 이상   | 레드 | `#FF1744` |

---

## 8. 파일 재생 UX 플로우

```
1. 파일 업로드 (Click or Drag)
   └─ decodeAudioData() 로 오디오 버퍼 생성
   └─ "Ready to Play" 상태 전환, 재생 길이 표시

2. 재생 (Play)
   └─ AudioBufferSourceNode 생성 및 start()
   └─ AudioContext.resume()
   └─ requestAnimationFrame() 루프 시작 (Time update)

3. 탐색 (Seek)
   └─ 현재 SourceNode stop()
   └─ 새로운 offset으로 start() 재시작
```

---

## 9. 데이터 내보내기 (Export)

- **CSV 내보내기**: 타임스탬프, Momentary, Short-term, Integrated, True Peak 컬럼
- **PNG 스냅샷**: 원형 히스토리 캔버스 이미지 저장
- **JSON 리포트**: 측정 메타데이터 + 전체 이력 배열

---

## 10. 브라우저 호환성

| 기능 | Chrome | Firefox | Safari |
|------|--------|---------|--------|
| Web Audio API | ✅ | ✅ | ✅ |
| AudioWorklet | ✅ | ✅ | ✅ (15.4+) |
| enumerateDevices | ✅ | ✅ | ✅ |
| setSinkId (출력 선택) | ✅ | ❌ | ❌ |
| SharedArrayBuffer | ✅ (COOP/COEP) | ✅ | ✅ |

> **권장 환경**: Chrome 110+ / Edge 110+ (출력 디바이스 선택 기능 포함)

---

## 11. 디자인 레퍼런스

- **배경**: 딥 다크 (`#0A0A0F`) — 하드웨어 미터 감성
- **타이포**: 모노스페이스 (JetBrains Mono / Roboto Mono)
- **액센트 컬러**: TC Electronic 특유의 민트 그린 (`#00E5CC`)
- **컨트롤 패널**: 반투명 글래스모피즘 카드
- **애니메이션**: 60fps Canvas requestAnimationFrame 루프

---

## 12. 개발 단계 로드맵

| Phase | 작업 | 산출물 |
|-------|------|--------|
| **Phase 1** | 기본 마이크 입력 + Momentary LUFS 수치 표시 | MVP HTML 단일 파일 |
| **Phase 2** | AudioWorklet LUFS 프로세서 구현 (전체 지표) | `lufs-processor.js` |
| **Phase 3** | 원형 클락와이즈 Canvas 렌더러 | `clock-meter.js` |
| **Phase 4** | 디바이스 입출력 선택 패널 | `device-panel.js` |
| **Phase 5** | 디자인 폴리싱 + Export 기능 | 완성 히어로 페이지 |
