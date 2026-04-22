# ts-skill-image-optimizer

Claude Code 스킬 — 이미지를 원하는 사이즈/용량/포맷으로 일괄 변환합니다.

## 기능

- 폴더 내 이미지 일괄 리사이즈 및 용량 압축
- 앱스토어 프리셋 지원 (`iphone` 1284×2778, `ipad` 2048px 너비 비율유지 크롭, `google` 1440×3040)
- JPG / PNG / WebP 포맷 변환
- 목표 용량 자동 품질 조정 (이진 탐색)
- 원본 파일 보존 (결과물은 별도 폴더에 저장)

## 설치 방법

Claude Code에 아래 문장을 그대로 붙여넣으세요:

> https://github.com/codinghyunman2/ts-skill-image-optimizer 를
> ~/.claude/skills/image-optimizer 에 클론해서 설치해줘

## 사용법

```
/image-optimizer [폴더경로] [목표 또는 프리셋]
```

**예시:**

```
/image-optimizer ./screenshots iphone
/image-optimizer ./photos 가로 1200px, 1MB 이하로
/image-optimizer images/ 800x600, 500KB 이하, jpg로
/image-optimizer . webp로 변환하고 300KB 이하로
```

## 프리셋

| 키워드 | 해상도 | 용도 |
|--------|--------|------|
| `iphone` | 1284×2778 | 애플 앱스토어 iPhone 스크린샷 (cover 크롭) |
| `ipad` | 2048×(원본비율–위아래420px) | 애플 앱스토어 iPad 스크린샷 |
| `google` | 1440×3040 | 구글 플레이스토어 스크린샷 (cover 크롭) |

## 요구사항

- Python 3 + Pillow (`pip install Pillow`)
- Claude Code CLI
