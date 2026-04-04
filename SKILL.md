---
name: see-video
description: "영상 첨부(mp4, mov, mkv 등)가 있거나 영상 내용 분석 요청을 받았을 때 사용. 영상에서 프레임을 추출해 LLM이 직접 볼 수 있는 이미지 그리드로 변환한다. uniform 모드(기본): 균등 간격 샘플링. highlight 모드: 장면 전환 기반 샘플링. ⚠️ 이미지 입력(멀티모달) 지원 모델 필수."
metadata:
  {
    "openclaw": {
      "emoji": "🎬",
      "requires": { "bins": ["ffmpeg", "node"] },
      "install": [
        {
          "id": "brew",
          "kind": "brew",
          "formula": "ffmpeg",
          "bins": ["ffmpeg"],
          "label": "Install ffmpeg (brew)",
        },
        {
          "id": "apt",
          "kind": "apt",
          "package": "ffmpeg",
          "bins": ["ffmpeg"],
          "label": "Install ffmpeg (apt)",
        },
      ],
    },
  }
---

# see-video

영상 → 프레임 그리드 이미지 + XML 타임스탬프를 LLM 컨텍스트에 주입한다.

## 초기 설정 (최초 1회)

스킬 디렉토리에서 의존성 설치:

```bash
cd <이 SKILL.md가 있는 디렉토리>
npm install
```

## 실행

```bash
node {baseDir}/scripts/inject.mjs <video_path> [--mode uniform|highlight] [--start N] [--end N]
```

성공 시 JSON stdout 출력:

```json
{
  "gridPath": "/tmp/video_llm-frames.jpg",
  "description": "<video_frames>...</video_frames>",
  "duration": 1326,
  "frameCount": 28,
  "layout": { "cols": 4, "rows": 7, "cellW": 384, "cellH": 216 },
  "videoWidth": 854,
  "videoHeight": 480,
  "inputSizeMb": 42.3
}
```

에러 시 stderr에 `ERROR: <메시지>` + `Hint: <진단>` 출력 후 exit 1.

## 주입 절차

**1단계 — 스크립트 실행 (bash 툴):**

```bash
node {baseDir}/scripts/inject.mjs "/path/to/video.mp4"
```

**2단계 — JSON 파싱:**
`gridPath`와 `description` 추출.

**3단계 — 이미지 주입 (read 툴):**

```
read <gridPath>
```

`read` 툴이 jpg를 멀티모달 이미지 블록으로 직접 컨텍스트에 주입한다.
이미지를 확인한 후 `description` XML의 타임스탬프를 참고해 분석:

> "그리드 이미지를 봤다면, 위 description XML의 타임스탬프를 참고해서 영상 내용을 분석해줘. 각 셀 좌상단 숫자가 프레임 인덱스."

**에러 발생 시:**
- stderr의 `Hint:` 메시지를 사용자에게 자연스러운 언어로 전달. raw 에러 메시지 그대로 붙여넣기 금지.
- `read <gridPath>` 실패 시 — `/tmp/`는 임시 디렉토리이므로 파일이 사라졌을 수 있음. 스크립트 재실행 후 즉시 read 할 것.

## 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--mode uniform` | 균등 간격 샘플링 | ✅ 기본 |
| `--mode highlight` | 장면 전환 기반 샘플링 | |
| `--start N` | 구간 시작 (초) | 0 |
| `--end N` | 구간 끝 (초) | 영상 끝 |

## 진단 가이드

| 에러 패턴 | 원인 | 해결 |
|-----------|------|------|
| `Input file not found` | 파일 없음. 채널 미디어 용량 제한으로 drop됐을 가능성 | 파일 경로를 텍스트로 직접 전달 요청 |
| `corrupt, incomplete, or unsupported format` | 파일 손상 / 전송 중단 / 지원하지 않는 코덱 | 다른 파일 시도, 또는 `--start`/`--end`로 구간 지정 |
| `moov atom not found` | mp4 전송이 완료되지 않은 파일 (스트리밍 미완료) | 완전한 파일로 재시도 |
| `ffmpeg not found` | ffmpeg 미설치 | `ffmpeg` 설치 확인 |

## 참고

- 프레임 수·셀 크기는 영상 길이/비율 기준 자동 결정 (라이브러리 내부)
- 그리드 ~1500×1500px, 셀 긴 변 384~512px
- 타임스탬프는 이미지가 아닌 `description` XML에만 포함
- 세로/가로 영상 모두 지원
