# ts-skill-image-optimizer

Claude Code 스킬 — 이미지를 원하는 사이즈/용량/포맷으로 일괄 변환합니다.

## 기능

- 폴더 내 이미지 일괄 리사이즈 및 용량 압축
- 앱스토어 프리셋 지원 (`iphone` 1284×2778, `ipad` 2048×2732)
- JPG / PNG / WebP 포맷 변환
- 목표 용량 자동 품질 조정 (이진 탐색)
- 원본 파일 보존 (결과물은 별도 폴더에 저장)

## 설치 방법

```bash
# 1. 스킬 디렉토리에 복사
mkdir -p ~/.claude/skills/image-optimizer
curl -L https://raw.githubusercontent.com/codinghyunman2/ts-skill-image-optimizer/main/SKILL.md \
  -o ~/.claude/skills/image-optimizer/SKILL.md
```

또는 이 레포를 클론:

```bash
git clone https://github.com/codinghyunman2/ts-skill-image-optimizer.git \
  ~/.claude/skills/image-optimizer
```

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
| `iphone` | 1284×2778 | 애플 앱스토어 iPhone 스크린샷 |
| `ipad` | 2048×2732 | 애플 앱스토어 iPad 스크린샷 |

## 요구사항

- Python 3 + Pillow (`pip install Pillow`)
- Claude Code CLI
