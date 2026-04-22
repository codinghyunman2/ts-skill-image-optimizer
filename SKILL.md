---
name: image-optimizer
description: 이미지를 특정 사이즈로 변경하고 용량을 줄이는 스킬. "이미지 압축", "이미지 리사이즈", "image resize", "이미지 최적화", "용량 줄여", "사진 크기 줄여" 등의 키워드로 트리거.
---

# 이미지 최적화 스킬

> 폴더 안의 이미지를 한 번에 리사이즈하고 용량을 줄여줍니다. 원본은 항상 보존됩니다.

## 트리거

- 명령어: `/image-optimizer [폴더경로] [목표 또는 프리셋]`
- 예시:
  - `/image-optimizer ./photos 가로 1200px, 1MB 이하로`
  - `/image-optimizer images/ 800x600, 500KB 이하`
  - `/image-optimizer . webp로 변환하고 300KB 이하로`
  - `/image-optimizer ./screenshots iphone` → 1284×2778 (애플 앱스토어 iPhone)
  - `/image-optimizer ./screenshots ipad`   → 2048px 너비, 위아래 420px 크롭 (애플 앱스토어 iPad)
  - `/image-optimizer ./screenshots google` → 1440×3040 (구글 플레이스토어)
- 자연어 트리거: "images 폴더 이미지 압축해줘", "photos/ 안 사진들 리사이즈해줘"

## 프리셋 목록

| 키워드 | 가로(px) | 세로(px) | 용도 |
|--------|----------|----------|------|
| `iphone` | 1284 | 2778 | 애플 앱스토어 iPhone 스크린샷 (cover 크롭) |
| `ipad`   | 2048 | 원본 비율 – 위아래 420px 크롭 | 애플 앱스토어 iPad 스크린샷 |
| `google` | 1440 | 3040 | 구글 플레이스토어 스크린샷 (cover 크롭) |

- `iphone` / `google`: 비율과 무관하게 **정확한 픽셀**로 출력 (cover & center-crop 방식)
- `ipad`: **2048px 너비**로 비율 유지 스케일 후 위아래 420px 크롭 (원본 비율 반영)

---

## 실행 절차

### Step 1. 입력 파악

사용자 명령에서 다음을 추출한다:
- **폴더 경로**: 이미지가 있는 디렉토리
- **프리셋 키워드**: `iphone` / `ipad` / `google` 중 하나면 프리셋 모드로 진행
- **목표 크기**: 가로(width) px, 세로(height) px (선택, 프리셋 없을 때)
- **목표 용량**: KB 또는 MB 단위 (선택)
- **출력 포맷**: jpg / png / webp (없으면 원본 포맷 유지)

프리셋과 수동 크기가 동시에 주어지면 **프리셋 우선**이다.

프리셋 키워드가 감지되면 Read 도구로 `~/.claude/skills/image-optimizer/presets.json`을 읽어 해당 키의 값을 가져온다. 스크립트 파라미터는 JSON에서 읽은 값으로 채운다.

정보가 부족하면 AskUserQuestion 도구로 확인한다. 단, 명확한 경우엔 바로 진행한다.

### Step 2. 환경 점검

**Python 명령어 감지** (Mac/Linux는 `python3`, Windows는 `python`):

```bash
python3 --version 2>/dev/null && echo "PYTHON_CMD=python3" || python --version 2>/dev/null && echo "PYTHON_CMD=python" || echo "PYTHON_NOT_FOUND"
```

- `python3` 성공 → 이후 모든 명령에 `python3` 사용
- `python` 성공 → 이후 모든 명령에 `python` 사용
- 둘 다 실패 → Python 미설치 안내 후 중단

감지된 명령어를 `PYTHON_CMD`로 기억하고 이후 모든 단계에서 사용한다.

**Pillow 설치 확인**:

```bash
$PYTHON_CMD -c "import PIL; print('ok')" 2>/dev/null || echo "not_installed"
```

`not_installed`가 출력되면 자동 설치를 시도한다 (`pip` 대신 `-m pip` 사용으로 크로스플랫폼 보장):

```bash
$PYTHON_CMD -m pip install Pillow
```

### Step 3. 폴더 스캔

처리할 이미지 목록을 출력한다 (Step 2에서 감지한 `$PYTHON_CMD` 사용):

```bash
$PYTHON_CMD -c "
import os
folder = '[폴더경로]'
exts = {'.jpg', '.jpeg', '.png', '.webp', '.gif', '.bmp', '.tiff'}
files = [f for f in os.listdir(folder) if os.path.splitext(f.lower())[1] in exts]
for f in files:
    size = os.path.getsize(os.path.join(folder, f))
    print(f'{f}: {size/1024:.0f}KB')
print(f'총 {len(files)}개 파일')
"
```

파일이 없으면 안내 메시지를 출력하고 중단한다.

### Step 4. 배치 처리

아래 Python 스크립트를 Bash 도구로 실행한다. `[파라미터]`를 Step 1에서 추출한 값으로 교체한다.

```python
#!/usr/bin/env python3
import os
import sys
import io
from pathlib import Path
from PIL import Image, ImageOps

# ── 파라미터 (Step 1에서 추출. 프리셋은 presets.json에서 읽어 아래에 직접 채워 넣음) ──
FOLDER = "[폴더경로]"     # 예: "./photos"
TARGET_WIDTH = None        # 예: 1200
TARGET_HEIGHT = None       # 예: 2778 (cover 모드) 또는 None (scale_crop 모드)
TARGET_KB = None           # 예: 1024 (1MB = 1024KB)
OUTPUT_FORMAT = None       # 예: "webp" (없으면 원본 포맷 유지)
OUTPUT_DIR = None          # None이면 자동 생성
PRESET_NAME = None         # 예: "iphone" (폴더명에 사용)
PRESET_MODE = None         # "cover" / "scale_crop" / None
PRESET_LABEL = ""          # 예: "[IPHONE 앱스토어 프리셋: 1284×2778]"
CROP_TOP = 0               # scale_crop 모드에서만 사용
CROP_BOTTOM = 0            # scale_crop 모드에서만 사용
EXACT_RESIZE = PRESET_MODE == 'cover'

# ── 지원 포맷 ────────────────────────────────────────────
SUPPORTED = {'.jpg', '.jpeg', '.png', '.webp', '.gif', '.bmp', '.tiff'}

folder = Path(FOLDER)
if not folder.exists():
    print(f"오류: '{FOLDER}' 폴더를 찾을 수 없습니다.")
    sys.exit(1)

# 출력 폴더 설정
if OUTPUT_DIR:
    out_dir = Path(OUTPUT_DIR)
elif PRESET_NAME:
    out_dir = folder.parent / f"{folder.name}_{PRESET_NAME.lower()}"
else:
    out_dir = folder.parent / f"{folder.name}_resized"
out_dir.mkdir(exist_ok=True)

# 이미지 목록
files = [f for f in folder.iterdir() if f.suffix.lower() in SUPPORTED]
if not files:
    print(f"오류: '{FOLDER}'에서 이미지를 찾을 수 없습니다.")
    sys.exit(1)

if PRESET_LABEL:
    print(PRESET_LABEL)
    print()

def resize_cover(img, tw, th):
    """비율 유지하며 목표 크기를 완전히 덮은 뒤 중앙 크롭 → 정확히 tw×th"""
    ow, oh = img.size
    scale = max(tw / ow, th / oh)
    new_w = round(ow * scale)
    new_h = round(oh * scale)
    img = img.resize((new_w, new_h), Image.LANCZOS)
    left = (new_w - tw) // 2
    top  = (new_h - th) // 2
    return img.crop((left, top, left + tw, top + th))

results = []

for img_path in sorted(files):
    try:
        original_size = img_path.stat().st_size

        with Image.open(img_path) as img:
            img = ImageOps.exif_transpose(img)
            original_w, original_h = img.size

            # 화질 저하 경고 (프리셋 사용 시 업스케일 감지)
            upscale_warning = ""
            if PRESET_MODE == 'scale_crop' and original_w < TARGET_WIDTH:
                upscale_warning = " ⚠️ 원본보다 확대됨 (화질 저하 가능)"
            elif EXACT_RESIZE and (original_w < TARGET_WIDTH or (TARGET_HEIGHT and original_h < TARGET_HEIGHT)):
                upscale_warning = " ⚠️ 원본보다 확대됨 (화질 저하 가능)"

            # 리사이즈
            if PRESET_MODE == 'scale_crop':
                new_h = round(original_h * TARGET_WIDTH / original_w)
                img = img.resize((TARGET_WIDTH, new_h), Image.LANCZOS)
                img = img.crop((0, CROP_TOP, TARGET_WIDTH, new_h - CROP_BOTTOM))
            elif TARGET_WIDTH and TARGET_HEIGHT:
                if EXACT_RESIZE:
                    img = resize_cover(img, TARGET_WIDTH, TARGET_HEIGHT)
                else:
                    img.thumbnail((TARGET_WIDTH, TARGET_HEIGHT), Image.LANCZOS)
            elif TARGET_WIDTH or TARGET_HEIGHT:
                tw = TARGET_WIDTH or original_w
                th = TARGET_HEIGHT or original_h
                img.thumbnail((tw, th), Image.LANCZOS)

            new_w, new_h = img.size

            # 출력 포맷 결정
            fmt = OUTPUT_FORMAT.upper() if OUTPUT_FORMAT else img_path.suffix.lstrip('.').upper()
            if fmt in ('JPG', 'JPEG'):
                fmt = 'JPEG'
                save_ext = '.jpg'
            elif fmt == 'PNG':
                save_ext = '.png'
            elif fmt == 'WEBP':
                save_ext = '.webp'
            else:
                fmt = 'JPEG'
                save_ext = '.jpg'

            # RGBA → RGB 변환 (JPEG 저장 시 필요)
            if fmt == 'JPEG' and img.mode in ('RGBA', 'P', 'LA'):
                bg = Image.new('RGB', img.size, (255, 255, 255))
                bg.paste(img, mask=img.split()[-1] if img.mode in ('RGBA', 'LA') else None)
                img = bg

            out_path = out_dir / (img_path.stem + save_ext)

            # 목표 용량 달성을 위한 품질 자동 조정 (이진 탐색)
            if TARGET_KB and fmt in ('JPEG', 'WEBP'):
                low, high = 10, 95
                best_quality = high
                for _ in range(8):
                    mid = (low + high) // 2
                    buf = io.BytesIO()
                    img.save(buf, format=fmt, quality=mid)
                    if buf.tell() / 1024 <= TARGET_KB:
                        low = mid + 1
                        best_quality = mid
                    else:
                        high = mid - 1
                img.save(out_path, format=fmt, quality=best_quality)
            elif fmt == 'PNG':
                img.save(out_path, format=fmt, optimize=True)
            else:
                img.save(out_path, format=fmt, quality=85)

            new_size = out_path.stat().st_size
            ratio = (1 - new_size / original_size) * 100 if original_size > 0 else 0

            warning = upscale_warning
            if TARGET_KB and fmt == 'PNG' and new_size / 1024 > TARGET_KB:
                warning += " ⚠️ PNG는 무손실 포맷이라 목표 용량 달성 불가"

            results.append({
                'file': img_path.name,
                'original_size': original_size,
                'new_size': new_size,
                'original_dim': f"{original_w}×{original_h}",
                'new_dim': f"{new_w}×{new_h}",
                'ratio': ratio,
                'warning': warning,
                'status': 'success'
            })
            print(f"✓ {img_path.name}: {original_size/1024:.0f}KB → {new_size/1024:.0f}KB ({new_w}×{new_h}){warning}")

    except Exception as e:
        results.append({'file': img_path.name, 'status': 'error', 'error': str(e)})
        print(f"✗ {img_path.name}: 실패 — {e}")

# 요약
success = [r for r in results if r['status'] == 'success']
total_before = sum(r['original_size'] for r in success)
total_after = sum(r['new_size'] for r in success)
saved = total_before - total_after

print(f"\n{'='*50}")
print(f"완료: {len(success)}/{len(results)}개 처리")
print(f"총 용량: {total_before/1024/1024:.1f}MB → {total_after/1024/1024:.1f}MB (절감: {saved/1024/1024:.1f}MB)")
print(f"저장 위치: {out_dir}")
```

임시 파일 경로를 감지한 뒤 스크립트를 저장하고 실행한다:

```bash
# 크로스플랫폼 임시 경로 감지 (Mac: /tmp, Windows: %TEMP%)
TMPFILE=$($PYTHON_CMD -c "import tempfile, os; print(os.path.join(tempfile.gettempdir(), 'image_optimizer_run.py'))")
# 위 경로에 스크립트를 Write 도구로 저장한 뒤:
$PYTHON_CMD "$TMPFILE"
```

### Step 5. 결과 리포트 출력

처리 완료 후 다음 형식으로 결과를 보여준다:

```
처리 완료!

파일명              원본        결과        해상도             절감
photo1.jpg         2.3MB  →   487KB       1200×800          79%
photo2.png         1.1MB  →   934KB ⚠️    원본유지 (PNG 한계)
photo3.jpg         890KB  →   312KB       1200×750          65%

총계: 3개 처리 | 4.3MB → 1.7MB | 2.6MB 절감
저장 위치: photos_resized/
원본 파일은 photos/ 에 그대로 보존됩니다.
```

---

## 오류 처리

| 상황 | 대응 |
|------|------|
| 폴더 없음 | `오류: '{경로}' 폴더를 찾을 수 없습니다.` 출력 후 중단 |
| 이미지 없음 | `해당 폴더에 이미지 파일이 없습니다.` 출력 후 중단 |
| Pillow 미설치 | `$PYTHON_CMD -m pip install Pillow` 실행 안내 |
| 손상된 파일 | 해당 파일만 건너뛰고 나머지 계속 처리 |
| 쓰기 권한 없음 | `오류: '{경로}'에 쓰기 권한이 없습니다.` 출력 |
| PNG 목표 용량 미달 | 경고 표시 후 최대한 압축한 결과 저장 |

**원본 보호 원칙**: 결과물은 항상 별도 폴더(`{폴더명}_resized/`)에 저장. 원본 파일은 절대 덮어쓰지 않는다.

---

## 완료 보고

처리 후 반드시 다음을 출력한다:
1. 처리된 파일 수 / 전체 파일 수
2. 총 용량 변화 (MB 단위)
3. 결과 저장 경로
4. 실패한 파일이 있다면 이유와 함께 목록
