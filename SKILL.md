---
name: image-optimizer
description: 이미지를 특정 사이즈로 변경하고 용량을 줄이는 스킬. "이미지 압축", "이미지 리사이즈", "image resize", "이미지 최적화", "용량 줄여", "사진 크기 줄여" 등의 키워드로 트리거.
---

# 이미지 최적화 스킬

> 폴더 안의 이미지를 한 번에 리사이즈하고 용량을 줄여줍니다. 원본은 항상 보존됩니다.

## 트리거

- 명령어: `/image-optimizer [폴더경로] [목표 또는 프리셋] [옵션]`
- 예시:
  - `/image-optimizer ./photos 가로 1200px, 1MB 이하로`
  - `/image-optimizer images/ 800x600, 500KB 이하`
  - `/image-optimizer . webp로 변환하고 300KB 이하로`
  - `/image-optimizer ./screenshots iphone` → 1284×2778 (애플 앱스토어 iPhone)
  - `/image-optimizer ./screenshots ipad`   → 2048px 너비, 위아래 420px 크롭 (애플 앱스토어 iPad)
  - `/image-optimizer ./screenshots google` → 1440×3040 (구글 플레이스토어)
  - `/image-optimizer ./photos iphone --dry-run` → 실제 저장 없이 미리보기
  - `/image-optimizer ./gallery 800px --recursive` → 하위 폴더까지 재귀 처리
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
- **`--dry-run`**: 실제 저장 없이 처리 결과만 미리보기
- **`--recursive`**: 하위 폴더까지 재귀적으로 처리
- **`--skip-existing`**: 출력 폴더에 이미 있는 파일 건너뜀 (없으면 재실행 시 사용자에게 확인)

프리셋과 수동 크기가 동시에 주어지면 **프리셋 우선**이다.

프리셋 키워드가 감지되면 Read 도구로 스킬 기본 디렉토리의 `presets.json`을 읽어 해당 키의 값을 가져온다. 스킬 기본 디렉토리는 시스템 컨텍스트의 `Base directory for this skill:` 값이다.

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

**HEIC 파일 감지 및 pillow-heif 확인**:

폴더에 `.heic` / `.heif` 파일이 있는지 확인한다:

```bash
$PYTHON_CMD -c "
import os
folder = '[폴더경로]'
heic = [f for f in os.listdir(folder) if f.lower().endswith(('.heic', '.heif'))]
if not heic:
    print('NO_HEIC')
else:
    try:
        import pillow_heif
        pillow_heif.register_heif_opener()
        print('HEIC_SUPPORTED')
    except ImportError:
        print(f'HEIC_NOT_SUPPORTED:{len(heic)}')
"
```

- `NO_HEIC` → 아무 조치 없이 계속
- `HEIC_SUPPORTED` → HEIC 파일도 처리 대상에 포함 (`HEIC_OK = True`로 기억)
- `HEIC_NOT_SUPPORTED:N` → 사용자에게 안내 출력:
  > "HEIC 파일 N개가 발견됐습니다. 처리하려면 `$PYTHON_CMD -m pip install pillow-heif`를 실행하세요. 지금은 HEIC 파일을 건너뜁니다."
  그리고 `HEIC_OK = False`로 계속 진행한다.

### Step 3. 파일 스캔 & 사전 확인

아래 스크립트로 처리 대상 파일 수, 총 용량, 기존 출력 폴더 상태를 스캔한다:

```bash
$PYTHON_CMD -c "
import os, sys
from pathlib import Path

FOLDER = '[폴더경로]'
RECURSIVE = [True 또는 False]
OUT_DIR = '[출력 폴더경로]'
HEIC_OK = [True 또는 False]

SUPPORTED = {'.jpg', '.jpeg', '.png', '.webp', '.gif', '.bmp', '.tiff'}
if HEIC_OK:
    import pillow_heif; pillow_heif.register_heif_opener()
    SUPPORTED |= {'.heic', '.heif'}

folder = Path(FOLDER)
if not folder.exists():
    print('FOLDER_NOT_FOUND'); sys.exit(0)

if RECURSIVE:
    files = [f for f in folder.rglob('*') if f.suffix.lower() in SUPPORTED and f.is_file()]
else:
    files = [f for f in folder.iterdir() if f.suffix.lower() in SUPPORTED and f.is_file()]

if not files:
    print('NO_FILES'); sys.exit(0)

total_kb = sum(f.stat().st_size for f in files) / 1024
print(f'FOUND:{len(files)}:{total_kb:.0f}')

out_dir = Path(OUT_DIR)
if out_dir.exists():
    existing = [f for f in out_dir.iterdir() if f.suffix.lower() in SUPPORTED and f.is_file()]
    print(f'EXISTING:{len(existing)}')
else:
    print('EXISTING:0')
"
```

스캔 결과를 해석한다:
- `FOLDER_NOT_FOUND` → "오류: 폴더를 찾을 수 없습니다." 출력 후 중단
- `NO_FILES` → "해당 폴더에 처리할 이미지가 없습니다." 출력 후 중단
- `FOUND:N:총KB` → 파일 N개, 총 용량 표시

**사전 확인 조건** — AskUserQuestion으로 묻는다:

| 조건 | 질문 내용 |
|------|-----------|
| 파일 **10개 이상** | "총 N개 파일 (X.XMB)을 처리합니다. 계속할까요?" → 예 / 아니요 |
| 출력 폴더에 **기존 파일 존재** & `--skip-existing` 플래그 없음 | "출력 폴더에 이미 N개 파일이 있습니다." → 덮어쓰기 / 건너뛰기 / 취소 |

두 조건이 동시에 해당하면 하나의 질문으로 합쳐서 묻는다. 사용자가 취소하면 중단한다.

### Step 4. 배치 처리

아래 Python 스크립트를 Bash 도구로 실행한다. `[파라미터]`를 앞 단계에서 추출한 값으로 교체한다.

```python
#!/usr/bin/env python3
import os
import sys
import io
from pathlib import Path
from PIL import Image, ImageOps

# ── 파라미터 (Step 1~3에서 추출한 값으로 교체) ──────────
FOLDER = "[폴더경로]"
TARGET_WIDTH = None        # 예: 1200
TARGET_HEIGHT = None       # 예: 2778 (cover 모드) 또는 None (scale_crop 모드)
TARGET_KB = None           # 예: 1024 (1MB = 1024KB)
OUTPUT_FORMAT = None       # 예: "webp" (없으면 원본 포맷 유지)
OUTPUT_DIR = None          # None이면 자동 생성
PRESET_NAME = None         # 예: "iphone" (폴더명에 사용)
PRESET_MODE = None         # "cover" / "scale_crop" / None
PRESET_LABEL = ""
CROP_TOP = 0
CROP_BOTTOM = 0
EXACT_RESIZE = PRESET_MODE == 'cover'
DRY_RUN = False            # --dry-run 플래그
RECURSIVE = False          # --recursive 플래그
SKIP_EXISTING = False      # --skip-existing 또는 Step 3에서 사용자가 선택
HEIC_OK = False            # pillow-heif 설치 여부

# ── HEIC 지원 등록 ────────────────────────────────────────
if HEIC_OK:
    try:
        import pillow_heif
        pillow_heif.register_heif_opener()
    except ImportError:
        HEIC_OK = False

# ── 지원 포맷 ────────────────────────────────────────────
SUPPORTED = {'.jpg', '.jpeg', '.png', '.webp', '.gif', '.bmp', '.tiff'}
if HEIC_OK:
    SUPPORTED |= {'.heic', '.heif'}

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

if not DRY_RUN:
    out_dir.mkdir(exist_ok=True)

# 이미지 목록 (재귀 여부 반영)
if RECURSIVE:
    files = [f for f in folder.rglob('*') if f.suffix.lower() in SUPPORTED and f.is_file()]
else:
    files = [f for f in folder.iterdir() if f.suffix.lower() in SUPPORTED and f.is_file()]

if not files:
    print(f"오류: '{FOLDER}'에서 이미지를 찾을 수 없습니다.")
    sys.exit(1)

if DRY_RUN:
    print("[DRY-RUN] 실제 저장 없이 미리보기 모드")
    print()

if PRESET_LABEL:
    print(PRESET_LABEL)
    print()

def resize_cover(img, tw, th):
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
        # 출력 경로 구성 (재귀 모드에서 서브폴더 구조 유지)
        if RECURSIVE:
            rel = img_path.relative_to(folder)
            out_path_dir = out_dir / rel.parent
            if not DRY_RUN:
                out_path_dir.mkdir(parents=True, exist_ok=True)
        else:
            out_path_dir = out_dir

        original_size = img_path.stat().st_size

        # 출력 포맷 및 확장자 결정 (저장 전에 먼저 결정 — skip 검사에 필요)
        fmt_raw = OUTPUT_FORMAT.upper() if OUTPUT_FORMAT else img_path.suffix.lstrip('.').upper()
        if fmt_raw in ('JPG', 'JPEG', 'HEIC', 'HEIF'):
            fmt, save_ext = 'JPEG', '.jpg'
        elif fmt_raw == 'PNG':
            fmt, save_ext = 'PNG', '.png'
        elif fmt_raw == 'WEBP':
            fmt, save_ext = 'WEBP', '.webp'
        else:
            fmt, save_ext = 'JPEG', '.jpg'

        out_path = out_path_dir / (img_path.stem + save_ext)

        # 기존 파일 건너뜀
        if SKIP_EXISTING and out_path.exists():
            print(f"⏭ {img_path.name}: 건너뜀 (이미 존재)")
            results.append({'file': img_path.name, 'status': 'skipped'})
            continue

        with Image.open(img_path) as img:
            img = ImageOps.exif_transpose(img)
            original_w, original_h = img.size

            # 화질 저하 경고 (업스케일 감지)
            upscale_warning = ""
            if PRESET_MODE == 'scale_crop' and original_w < TARGET_WIDTH:
                upscale_warning = " ⚠️ 원본보다 확대됨 (화질 저하 가능)"
            elif EXACT_RESIZE and (original_w < TARGET_WIDTH or (TARGET_HEIGHT and original_h < TARGET_HEIGHT)):
                upscale_warning = " ⚠️ 원본보다 확대됨 (화질 저하 가능)"

            # 리사이즈
            if PRESET_MODE == 'scale_crop':
                scaled_h = round(original_h * TARGET_WIDTH / original_w)
                img = img.resize((TARGET_WIDTH, scaled_h), Image.LANCZOS)
                # 안전 검사: 크롭 후 높이 유효성 확인
                if scaled_h <= CROP_TOP + CROP_BOTTOM:
                    raise ValueError(
                        f"크롭 후 높이가 0 이하입니다 "
                        f"(스케일 후 {scaled_h}px, 크롭 {CROP_TOP + CROP_BOTTOM}px). "
                        f"이미지가 너무 가로로 길거나 세로가 짧습니다."
                    )
                img = img.crop((0, CROP_TOP, TARGET_WIDTH, scaled_h - CROP_BOTTOM))
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

            # RGBA → RGB 변환 (JPEG 저장 시 필요)
            if fmt == 'JPEG' and img.mode in ('RGBA', 'P', 'LA'):
                bg = Image.new('RGB', img.size, (255, 255, 255))
                bg.paste(img, mask=img.split()[-1] if img.mode in ('RGBA', 'LA') else None)
                img = bg

            # DRY-RUN: 저장 없이 용량 추정
            if DRY_RUN:
                buf = io.BytesIO()
                if fmt == 'PNG':
                    img.save(buf, format=fmt, optimize=True)
                else:
                    img.save(buf, format=fmt, quality=85)
                estimated_size = buf.tell()
                ratio = (1 - estimated_size / original_size) * 100 if original_size > 0 else 0
                print(f"[예상] {img_path.name}: {original_size/1024:.0f}KB → ~{estimated_size/1024:.0f}KB ({new_w}×{new_h}){upscale_warning}")
                results.append({
                    'file': img_path.name, 'original_size': original_size,
                    'new_size': estimated_size, 'ratio': ratio,
                    'warning': upscale_warning, 'status': 'dry_run'
                })
                continue

            # 목표 용량 달성을 위한 품질 자동 조정 (이진 탐색)
            if TARGET_KB and fmt in ('JPEG', 'WEBP'):
                low, high, best_quality = 10, 95, 95
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
                'file': img_path.name, 'original_size': original_size,
                'new_size': new_size, 'new_dim': f"{new_w}×{new_h}",
                'ratio': ratio, 'warning': warning, 'status': 'success'
            })
            print(f"✓ {img_path.name}: {original_size/1024:.0f}KB → {new_size/1024:.0f}KB ({new_w}×{new_h}){warning}")

    except Exception as e:
        results.append({'file': img_path.name, 'status': 'error', 'error': str(e)})
        print(f"✗ {img_path.name}: 실패 — {e}")

# 요약
done = [r for r in results if r['status'] in ('success', 'dry_run')]
skipped = [r for r in results if r['status'] == 'skipped']
failed = [r for r in results if r['status'] == 'error']
total_before = sum(r['original_size'] for r in done)
total_after = sum(r['new_size'] for r in done)
saved = total_before - total_after

print(f"\n{'='*50}")
if DRY_RUN:
    print(f"[DRY-RUN] 예상 결과: {len(done)}/{len(results)}개")
    print(f"예상 용량: {total_before/1024/1024:.1f}MB → ~{total_after/1024/1024:.1f}MB (절감: ~{saved/1024/1024:.1f}MB)")
    print(f"저장 위치 (예정): {out_dir}")
else:
    skip_note = f" | {len(skipped)}개 건너뜀" if skipped else ""
    print(f"완료: {len(done)}/{len(results)}개 처리{skip_note}")
    print(f"총 용량: {total_before/1024/1024:.1f}MB → {total_after/1024/1024:.1f}MB (절감: {saved/1024/1024:.1f}MB)")
    print(f"저장 위치: {out_dir}")
```

**주의**: `Write` 도구는 기존 파일을 먼저 읽어야 하므로 새 임시 파일에 사용할 수 없다. OS에 따라 아래 방법으로 저장 후 실행한다.

**Step 2에서 OS 타입을 추가로 감지한다:**

```bash
$PYTHON_CMD -c "import sys; print('windows' if sys.platform=='win32' else 'unix')"
```

- `unix` (Mac/Linux) → `cat` heredoc 방식 사용
- `windows` → PowerShell here-string 방식 사용

---

**Mac/Linux — 단일 Bash 호출로 작성 + 실행:**

```bash
cat > /tmp/image_optimizer_run.py << 'PYEOF'
[위 스크립트 전체 내용 — 파라미터를 실제 값으로 교체한 상태]
PYEOF
python3 /tmp/image_optimizer_run.py
```

heredoc 구분자는 반드시 `'PYEOF'`처럼 따옴표로 감싸야 변수 치환이 방지된다.

---

**Windows — PowerShell here-string 방식:**

```powershell
@'
[위 스크립트 전체 내용 — 파라미터를 실제 값으로 교체한 상태]
'@ | Set-Content -Path "$env:TEMP\image_optimizer_run.py" -Encoding UTF8
python "$env:TEMP\image_optimizer_run.py"
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

`--dry-run` 결과라면 "처리 완료!" 대신 "[DRY-RUN] 예상 결과"로 표시하고, 수치에 `~`를 붙여 추정임을 표시한다.

---

## 오류 처리

| 상황 | 대응 |
|------|------|
| 폴더 없음 | `오류: '{경로}' 폴더를 찾을 수 없습니다.` 출력 후 중단 |
| 이미지 없음 | `해당 폴더에 처리할 이미지가 없습니다.` 출력 후 중단 |
| Pillow 미설치 | `$PYTHON_CMD -m pip install Pillow` 자동 시도 |
| HEIC 미지원 | `$PYTHON_CMD -m pip install pillow-heif` 안내 후 HEIC 파일 건너뜀 |
| 손상된 파일 | 해당 파일만 건너뛰고 나머지 계속 처리 |
| 쓰기 권한 없음 | `오류: '{경로}'에 쓰기 권한이 없습니다.` 출력 |
| PNG 목표 용량 미달 | 경고 표시 후 최대한 압축한 결과 저장 |
| ipad 크롭 높이 부족 | `크롭 후 높이가 0 이하입니다` 에러 출력 후 해당 파일만 건너뜀 |

**원본 보호 원칙**: 결과물은 항상 별도 폴더(`{폴더명}_resized/`)에 저장. 원본 파일은 절대 덮어쓰지 않는다.

---

## 완료 보고

처리 후 반드시 다음을 출력한다:
1. 처리된 파일 수 / 전체 파일 수 (건너뛴 파일 수 포함)
2. 총 용량 변화 (MB 단위) — dry-run이면 예상값(`~`)으로 표시
3. 결과 저장 경로
4. 실패한 파일이 있다면 이유와 함께 목록
