# imageinfo

أداة Bash لعرض معلومات تفصيلية عن ملفات الصور على لينكس.

## اسهل طريقه للتحميل 
```bash
cat <<'EOF' | sudo tee /usr/local/bin/imageinfo > /dev/null
#!/usr/bin/env bash
# imageinfo - أداة لعرض معلومات تفصيلية عن ملف صورة في لينكس (system-wide)
# Usage:
#   imageinfo /path/to/image
#   imageinfo
#   imageinfo --save /path/to/image

set -euo pipefail

PRINT_USAGE() {
  cat <<USAGE
Usage:
  imageinfo [--save] /path/to/image
  imageinfo [--save]
Options:
  --save   حفظ التقرير النصي بجانب ملف الصورة (اسم الملف: <image>.info.txt)
USAGE
}

# parse args
SAVE_REPORT=0
ARGS=()
for arg in "$@"; do
  case "$arg" in
    --save) SAVE_REPORT=1 ;;
    -h|--help) PRINT_USAGE; exit 0 ;;
    *) ARGS+=("$arg") ;;
  esac
done

IMG="${ARGS[0]:-}"

if [[ -z "$IMG" ]]; then
  echo "🚀 اسحب ملف الصورة هنا واضغط Enter:"
  read -r IMG
fi

# إزالة أي علامات اقتباس مفردة/مزدوجة وتقليم مسافات
IMG="${IMG#\"}"; IMG="${IMG%\"}"
IMG="${IMG#\'}"; IMG="${IMG%\'}"
IMG="$(printf '%s' "$IMG" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//')"

if [[ -z "$IMG" ]]; then
  echo "❌ لم يتم إدخال مسار صالح."
  exit 1
fi

if [[ ! -f "$IMG" ]]; then
  echo "❌ File not found: $IMG"
  exit 2
fi

REPORT_TMP="$(mktemp)"
cleanup() { rm -f "$REPORT_TMP"; }
trap cleanup EXIT

log() {
  printf '%s\n' "$*" | tee -a "$REPORT_TMP"
}

log "===== imageinfo report ====="
log "File: $IMG"
log "Timestamp: $(date --iso-8601=seconds 2>/dev/null || date)"

log ""
log "===== Basic filesystem info ====="
if command -v stat >/dev/null 2>&1; then
  stat --printf="Size: %s bytes\nPermissions: %A\nOwner: %U:%G\n" -- "$IMG" 2>/dev/null || ls -lh -- "$IMG"
else
  ls -lh -- "$IMG"
fi

log ""
log "===== file(1) type ====="
if command -v file >/dev/null 2>&1; then
  file --brief --mime-type -- "$IMG" 2>/dev/null || file --brief "$IMG"
else
  log "file(1) not available"
fi

log ""
log "===== Hashes ====="
if command -v md5sum >/dev/null 2>&1; then
  md5sum -- "$IMG" | tee -a "$REPORT_TMP"
fi
if command -v sha1sum >/dev/null 2>&1; then
  sha1sum -- "$IMG" | tee -a "$REPORT_TMP"
fi
if command -v shasum >/dev/null 2>&1; then
  shasum -a 256 -- "$IMG" | tee -a "$REPORT_TMP" || true
fi

log ""
log "===== ImageMagick (identify) ====="
if command -v identify >/dev/null 2>&1; then
  identify -format "Format: %m\nGeometry: %wx%h\nColorspace: %r\nDepth: %z\nCompression: %C\nQuality: %Q\n" -- "$IMG" 2>/dev/null | tee -a "$REPORT_TMP" || true
  log ""
  log "identify -verbose (partial):"
  identify -verbose -- "$IMG" 2>/dev/null | sed -n '1,200p' | tee -a "$REPORT_TMP" || true
else
  log "identify (ImageMagick) not found"
fi

log ""
log "===== exiftool (full metadata) ====="
if command -v exiftool >/dev/null 2>&1; then
  exiftool -a -u -g1 -- "$IMG" | tee -a "$REPORT_TMP" || true
else
  log "exiftool not installed"
fi

log ""
log "===== exiv2 (EXIF/IPTC/XMP) ====="
if command -v exiv2 >/dev/null 2>&1; then
  exiv2 -pa -- "$IMG" | tee -a "$REPORT_TMP" || true
else
  log "exiv2 not installed"
fi

log ""
log "===== mediainfo (if installed) ====="
if command -v mediainfo >/dev/null 2>&1; then
  mediainfo --Output=JSON "$IMG" 2>/dev/null | tee -a "$REPORT_TMP" || mediainfo "$IMG" | tee -a "$REPORT_TMP"
else
  log "mediainfo not installed"
fi

log ""
log "===== ffprobe (ffmpeg) ====="
if command -v ffprobe >/dev/null 2>&1; then
  ffprobe -v quiet -print_format json -show_format -show_streams "$IMG" 2>/dev/null | tee -a "$REPORT_TMP" || true
else
  log "ffprobe not installed"
fi

log ""
log "===== Python/Pillow quick summary ====="
if command -v python3 >/dev/null 2>&1; then
  python3 - <<PY | tee -a "$REPORT_TMP"
from PIL import Image, ImageStat
import sys, json
p = r"${IMG}"
try:
    img = Image.open(p)
except Exception as e:
    print("PIL cannot open image:", e)
    sys.exit(0)

info = {}
info['format'] = getattr(img, 'format', None)
info['mode'] = getattr(img, 'mode', None)
info['size'] = getattr(img, 'size', None)
info['info_keys'] = list(img.info.keys())
try:
    info['dpi'] = img.info.get('dpi')
except:
    info['dpi'] = None

try:
    stat = ImageStat.Stat(img)
    info['mean_channel'] = stat.mean
    info['extrema'] = stat.extrema
except Exception as e:
    info['stat_error'] = str(e)

print(json.dumps(info, indent=2, ensure_ascii=False))
PY
else
  log "python3 not installed"
fi

log ""
log "===== Optional: color histogram (ImageMagick convert) ====="
if command -v convert >/dev/null 2>&1; then
  log "Top colors (up to 20):"
  convert -- "$IMG" -colors 256 -depth 8 -format %c histogram:info:- | sort -r -n | head -n 20 | tee -a "$REPORT_TMP" || true
else
  log "convert (ImageMagick) not found"
fi

log ""
log "===== Done ====="

if [[ "$SAVE_REPORT" -eq 1 ]]; then
  OUT="$(dirname -- "$IMG")/$(basename -- "$IMG").info.txt"
  cat "$REPORT_TMP" >> "$OUT"
  echo "✅ تم حفظ التقرير في: $OUT"
else
  echo ""
  echo "ملاحظة: لتخزين التقرير استخدم الخيار --save"
fi

exit 0
EOF

sudo chmod +x /usr/local/bin/imageinfo
echo "✅ تم إنشاء الأداة: /usr/local/bin/imageinfo"
echo "✨ شغّلها بكتابة: imageinfo  أو imageinfo --save /path/to/image"

                                              
