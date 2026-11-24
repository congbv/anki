"""
Oxford Audio Downloader - Anki add-on (chạy được trên macOS và Windows)

Với mỗi note có link Oxford trong trường Sound:
  - chọn 1 link giọng Mỹ (nếu không có thì lấy link cuối cùng)
  - tải mp3 về thư mục media của Anki
  - thay TOÀN BỘ các thẻ <audio>/[sound:] Oxford bằng 1 thẻ <audio> trỏ tới file local

File nằm trong media nên sync được sang các thiết bị khác.
Template cần phát "audio cuối cùng" (xem front_template_fixed.txt).
"""

import os
import random
import re
import time
from urllib.parse import urlparse

import requests
from anki.collection import OpChanges
from aqt import mw
from aqt.operations import CollectionOp
from aqt.qt import QAction
from aqt.utils import askUser, showInfo, showText

# ====== CẤU HÌNH ======
SOUND_FIELD = "Sound"
PREFER = "us"   # "us" = giọng Mỹ, "gb" = giọng Anh
LIMIT = 400       # tối đa N note mỗi lần chạy (thử trước, sau đó tăng dần); None = tất cả

# Nếu bị Cloudflare chặn, dán cookie cf_clearance lấy từ trình duyệt vào COOKIE
# và đặt USER_AGENT đúng bằng user-agent của trình duyệt đó.
USER_AGENT = (
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36"
)
COOKIE = ""  # ví dụ: "cf_clearance=xxxxxxxx"
DELAY = (0.8, 1.6)  # nghỉ ngẫu nhiên giữa 2 lần tải (giây)
# ======================

DOMAIN = "oxfordlearnersdictionaries.com"
AUDIO_RE = re.compile(r"<audio\b[^>]*>.*?</audio>", re.S | re.I)
SOUND_TAG_RE = re.compile(r"\[sound:https?://[^\]]*\]")
SOUND_URL_RE = re.compile(r"\[sound:(https?://[^\]]+)\]")
SRC_RE = re.compile(r'<audio\b[^>]*\bsrc="(https?://[^"]+)"', re.I)
PATTERNS = {"us": r"us_pron|__us_", "gb": r"uk_pron|__gb_"}
MAX_CONSECUTIVE_FAILS = 5


class Blocked(Exception):
    pass


def _pick(urls):
    oxford = [u for u in urls if DOMAIN in u]
    if not oxford:
        return None
    preferred = [u for u in oxford if re.search(PATTERNS[PREFER], u)]
    return preferred[0] if preferred else oxford[-1]


def _fetch(url):
    headers = {
        "User-Agent": USER_AGENT,
        "Referer": "https://www.oxfordlearnersdictionaries.com/",
        "Accept": "*/*",
    }
    if COOKIE:
        headers["Cookie"] = COOKIE
    r = requests.get(url, headers=headers, timeout=30)
    if r.status_code in (403, 429, 503):
        raise Blocked()
    r.raise_for_status()
    if "text/html" in r.headers.get("Content-Type", ""):
        raise Blocked()
    if len(r.content) < 500:
        raise RuntimeError("file tải về quá nhỏ, có thể không phải audio")
    return r.content


def _replace_field(field, fname):
    """Thay mọi <audio>/[sound:] Oxford bằng đúng 1 thẻ audio local."""
    tag = (
        f'<audio src="{fname}" type="audio/mpeg" preload="auto" autobuffer="" '
        f'controls="">[sound:{fname}]</audio>'
    )
    state = {"done": False}

    def sub(m):
        if DOMAIN not in m.group(0):
            return m.group(0)
        if state["done"]:
            return ""
        state["done"] = True
        return tag

    new = AUDIO_RE.sub(sub, field)
    new = SOUND_TAG_RE.sub(sub, new)
    return new, state["done"]


def run():
    query = f'"{SOUND_FIELD}:*{DOMAIN}*"'
    stats = {
        "total": 0, "done": 0, "remaining": 0,
        "skipped": [], "failed": [], "blocked": False, "aborted": False,
    }

    def _work(col):
        all_nids = list(col.find_notes(query))
        nids = all_nids[:LIMIT] if LIMIT else all_nids
        stats["total"] = len(all_nids)

        notes = []
        consecutive = 0
        for n, nid in enumerate(nids):
            note = col.get_note(nid)
            if SOUND_FIELD not in note.keys():
                stats["skipped"].append(f"note {nid}: không có trường {SOUND_FIELD}")
                continue
            field = note[SOUND_FIELD]
            url = _pick(SOUND_URL_RE.findall(field) or SRC_RE.findall(field))
            if not url:
                stats["skipped"].append(f"note {nid}: không tìm thấy link Oxford")
                continue

            fname = "oxford_" + os.path.basename(urlparse(url).path)
            mw.taskman.run_on_main(
                lambda n=n, f=fname: mw.progress.update(label=f"{n + 1}/{len(nids)}: {f}")
            )

            if not col.media.have(fname):
                try:
                    data = _fetch(url)
                except Blocked:
                    stats["blocked"] = True
                    break
                except Exception as e:  # noqa: BLE001
                    stats["failed"].append(f"{url}  ->  {e}")
                    consecutive += 1
                    if consecutive >= MAX_CONSECUTIVE_FAILS:
                        stats["aborted"] = True
                        break
                    continue
                consecutive = 0
                fname = col.media.write_data(fname, data)
                time.sleep(random.uniform(*DELAY))

            new, changed = _replace_field(field, fname)
            if not changed:
                stats["skipped"].append(f"note {nid}: không nhận dạng được định dạng trường Sound")
                continue
            note[SOUND_FIELD] = new
            notes.append(note)
            stats["done"] += 1

        stats["remaining"] = len(all_nids) - stats["done"]
        if not notes:
            return OpChanges()
        return col.update_notes(notes)

    def _report(_changes=None):
        lines = [
            f"Số note còn link Oxford lúc bắt đầu: {stats['total']}",
            f"Đã tải và chuyển sang file local: {stats['done']}",
            f"Còn lại chưa chuyển: {stats['remaining']}",
        ]
        if stats["blocked"]:
            lines.append(
                "\nBị Cloudflare chặn. Hãy lấy cookie cf_clearance và User-Agent từ "
                "trình duyệt, điền vào COOKIE và USER_AGENT đầu file, khởi động lại Anki "
                "rồi chạy lại. Phần đã tải được lưu, sẽ không tải lại."
            )
        if stats["aborted"]:
            lines.append(
                f"\nDừng vì lỗi liên tiếp {MAX_CONSECUTIVE_FAILS} lần (kiểm tra kết nối mạng)."
            )
        if stats["skipped"]:
            lines.append("\nBỏ qua:\n" + "\n".join(stats["skipped"]))
        if stats["failed"]:
            lines.append("\nLỗi:\n" + "\n".join(stats["failed"]))
        showText("\n".join(lines), title="Kết quả")

    if not askUser(
        "Hãy chắc chắn bạn đã backup (File → Export).\n"
        f"Giọng: {PREFER}, tối đa {LIMIT or 'tất cả'} note mỗi lần. Bắt đầu?"
    ):
        return
    CollectionOp(parent=mw, op=_work).success(_report).run_in_background()


action = QAction("Tải audio Oxford về máy (1 file)", mw)
action.triggered.connect(run)
mw.form.menuTools.addAction(action)
