import os
import re
import io
import html
import base64
import logging
import threading
import json
import hashlib
import secrets
import requests
import tempfile
import time
from datetime import datetime, timezone, timedelta
from collections import OrderedDict
from concurrent.futures import ThreadPoolExecutor
from contextlib import asynccontextmanager
from pathlib import Path
from typing import Optional

from PIL import Image, ImageDraw, ImageFont

try:
    import cv2
except ImportError:
    cv2 = None

from dotenv import load_dotenv
from fastapi import FastAPI, Query, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import FileResponse, JSONResponse, Response
from pydantic import BaseModel

import telebot
from telebot.types import (
    InlineKeyboardMarkup,
    InlineKeyboardButton,
    ReplyKeyboardMarkup,
    KeyboardButton,
    WebAppInfo,
)
from supabase import create_client, Client

# --------------------------------------------------
# Configuration
# --------------------------------------------------

load_dotenv()

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)
log = logging.getLogger("xfanster-user-webapp")

CAT_STORE_PREFIX = "__CAT__"
SYN_STORE_PREFIX = "__SYN__"
FAV_STORE_PREFIX = "__FAV__"
RATE_STORE_PREFIX = "__RATE__"
LANG_STORE_PREFIX = "__LANG__"
CODE_STORE_PREFIX = "__CODE__"
DATE_STORE_PREFIX = "__DATE__"
DUR_STORE_PREFIX = "__DUR__"
TAGS_STORE_PREFIX = "__TAGS__"
GIF_STORE_PREFIX = "__GIF__"
ALBUM_STORE_PREFIX = "__ALBUM__"
EP_COV_STORE_PREFIX = "__EPCOV__"

REQ_LIMIT_KEY = "__REQ_LIMIT__"
USER_START_KEY = "__USER_START__"
CFG_ADMIN_GROUP_KEY = "__CFG__ADMIN_GROUP_ID"
CFG_ADMINS_KEY = "__CFG__ADMIN_IDS"
CFG_TRENDING_KEY = "__CFG__TOP_TRENDING"
CFG_USER_BOT_TOKEN_KEY = "__CFG__USER_BOT_TOKEN"
CFG_MINI_APP_URL_KEY = "__CFG__MINI_APP_URL"
CFG_NOTIF_QUEUE_KEY = "__CFG__NOTIF_QUEUE"
CFG_REQ_CHANNEL_ID_KEY = "__CFG__REQ_CHANNEL_ID"
CFG_REQ_CHANNEL_LINK_KEY = "__CFG__REQ_CHANNEL_LINK"
CFG_REQ_CHANNEL_JSON_KEY = "__CFG__REQUIRED_CHANNEL"
NOTIF_STATS_KEY = "__CFG__NOTIF_STATS"
NOTIF_WATCH_PREFIX = "__NOTIF_WATCH__"
CHANNEL_DISPLAY_NAME = "χƒαη 18 קɭυѕ"
ADMIN_BOT_TOKEN_FALLBACK = (os.getenv("ADMIN_BOT_TOKEN", "") or "").strip()

CAMBODIA_TZ = timezone(timedelta(hours=7))

ADMIN_GROUP_INVITE_LINK = "https://t.me/+bu9JNK94p8w5NDE1"

BTN_PREV_EP = "◀️ ភាគមុន"
BTN_NEXT_EP = "ភាគបន្ត ▶️"
BTN_ALL_EPS = "📂 បញ្ជីភាគ"
BTN_MINI_APP = "✨ Mini App"
BTN_REFRESH = "🔄 refresh"
BTN_CLEAR = "សម្អាត 🗑️"

FALLBACK_SVG_BYTES = (
    b'<svg xmlns="http://www.w3.org/2000/svg" width="400" height="530">'
    b'<rect width="100%" height="100%" fill="#121212"/>'
    b'</svg>'
)


def required_env(*names):
    for name in names:
        value = os.getenv(name)
        if value and value.strip():
            return value.strip()
    return None


def _parse_int_safely(raw_val: Optional[str], name: str, allow_none: bool = False) -> Optional[int]:
    cleaned = str(raw_val or "").strip()
    if not cleaned:
        if allow_none:
            return None
        raise RuntimeError(f"Missing numeric environment variable: {name}")
    if cleaned.lstrip("-").isdigit():
        return int(cleaned)
    if "t.me/" in cleaned or cleaned.startswith(("http://", "https://")):
        if allow_none:
            log.warning("%s contains a URL (%s) instead of a numeric chat ID; ignoring.", name, cleaned)
            return None
    m = re.search(r"-?\d+", cleaned)
    if m and not allow_none:
        return int(m.group(0))
    if allow_none:
        log.warning("Could not parse optional %s=%r as integer; ignoring.", name, raw_val)
        return None
    raise RuntimeError(f"{name} must be a numeric ID (got: {raw_val!r}).")


BOT_TOKEN = required_env("BOT_TOKEN", "USER_BOT_TOKEN")
CHANNEL_ID_RAW = required_env("CHANNEL_ID", "SOURCE_CHANNEL_ID")
ADMIN_ID_RAW = required_env("ADMIN_ID")
ADMIN_IDS_RAW = os.getenv("ADMIN_IDS", "").strip()
ADMIN_GROUP_ID_RAW = required_env("ADMIN_GROUP_ID", "REQUEST_GROUP_ID")

MINI_APP_URL = required_env("MINI_APP_URL", "RENDER_EXTERNAL_URL", "PUBLIC_BASE_URL") or ""
if MINI_APP_URL and not MINI_APP_URL.startswith(("http://", "https://")):
    MINI_APP_URL = "https://" + MINI_APP_URL

SUPABASE_URL = required_env("SUPABASE_URL")
SUPABASE_KEY = required_env(
    "SUPABASE_KEY",
    "SUPABASE_SERVICE_ROLE_KEY",
)

missing = []
for key, value in [
    ("BOT_TOKEN", BOT_TOKEN),
    ("CHANNEL_ID or SOURCE_CHANNEL_ID", CHANNEL_ID_RAW),
    ("ADMIN_ID", ADMIN_ID_RAW),
    ("SUPABASE_URL", SUPABASE_URL),
    ("SUPABASE_KEY or SUPABASE_SERVICE_ROLE_KEY", SUPABASE_KEY),
]:
    if not value:
        missing.append(key)

if missing:
    raise RuntimeError("Missing environment variables: " + ", ".join(missing))

CHANNEL_ID = _parse_int_safely(CHANNEL_ID_RAW, "CHANNEL_ID")
REQUIRED_CHANNEL_ID = _parse_int_safely(
    os.getenv("REQUIRED_CHANNEL_ID", "-1004467396484"), "REQUIRED_CHANNEL_ID"
)
REQUIRED_CHANNEL_LINK = (os.getenv("REQUIRED_CHANNEL_LINK", "https://t.me/xfanster").strip()
                         or "https://t.me/xfanster")
ADMIN_ID = _parse_int_safely(ADMIN_ID_RAW, "ADMIN_ID")
ADMIN_GROUP_ID = _parse_int_safely(ADMIN_GROUP_ID_RAW, "ADMIN_GROUP_ID", allow_none=True)

ENV_ADMIN_IDS = {ADMIN_ID}
if ADMIN_IDS_RAW:
    for part in ADMIN_IDS_RAW.replace(";", ",").split(","):
        part = part.strip()
        if part.lstrip("-").isdigit():
            ENV_ADMIN_IDS.add(int(part))

if not SUPABASE_URL.startswith(("https://", "http://")):
    raise RuntimeError("SUPABASE_URL must be the real Supabase Project URL")

# --------------------------------------------------
# Supabase, Telegram Bot & Server Caches
# --------------------------------------------------

bot = telebot.TeleBot(BOT_TOKEN)
supabase: Client = create_client(SUPABASE_URL, SUPABASE_KEY)

user_last_message = {}
user_cover_message = {}
user_cover_drama = {}
user_main_menu_id = {}
user_trash_messages = {}
user_current_drama = {}
user_current_ep = {}
user_drama_multi_ep = {}

# Only populated when Telegram sends a real @bot.chat_join_request_handler event for that channel
PENDING_JOIN_REQUEST_USERS: set[tuple[int, int]] = set()
PENDING_WATCH_AFTER_JOIN: dict[int, dict] = {}

detected_admin_group_id: Optional[int] = ADMIN_GROUP_ID
dynamic_admin_ids = set(ENV_ADMIN_IDS)

admin_request_msg_map: "OrderedDict[tuple[int, int], dict]" = OrderedDict()
MAX_REQUEST_MAP_SIZE = 1000
request_map_lock = threading.Lock()

COVER_MEMORY_CACHE: "OrderedDict[str, tuple[bytes, str]]" = OrderedDict()
MAX_SERVER_COVER_CACHE = 220
cover_cache_lock = threading.Lock()

GIF_MEMORY_CACHE: "OrderedDict[str, tuple[bytes, str]]" = OrderedDict()
MAX_SERVER_GIF_CACHE = 60
gif_cache_lock = threading.Lock()

ep_extract_lock = threading.Lock()
cover_refresh_lock = threading.Lock()

dynamic_req_channel_lock = threading.Lock()
DYNAMIC_REQ_CHANNEL_CACHE = {
    "timestamp": 0.0,
    "channel_id": REQUIRED_CHANNEL_ID,
    "channel_link": REQUIRED_CHANNEL_LINK,
}
REQ_CHANNEL_CACHE_TTL = 15.0

DRAMAS_CACHE_TTL = 15.0
DRAMAS_LIST_CACHE = {
    "timestamp": 0.0,
    "payload": None,
}
dramas_cache_lock = threading.Lock()


def invalidate_dramas_cache():
    with dramas_cache_lock:
        DRAMAS_LIST_CACHE["timestamp"] = 0.0
        DRAMAS_LIST_CACHE["payload"] = None


def get_cambodia_now() -> datetime:
    return datetime.now(CAMBODIA_TZ)


def get_cambodia_date_int() -> int:
    return int(get_cambodia_now().strftime("%Y%m%d"))


def get_cambodia_date_str() -> str:
    return get_cambodia_now().strftime("%Y-%m-%d")


def get_cambodia_date_ddmmyyyy() -> str:
    return get_cambodia_now().strftime("%d-%m-%Y")


def normalize_date_to_ddmmyyyy(raw_val: str) -> str:
    if not raw_val:
        return ""
    s = str(raw_val).strip()
    if re.match(r"^\d{2}-\d{2}-\d{4}$", s):
        return s
    if re.match(r"^\d{4}-\d{2}-\d{2}", s):
        try:
            dt = datetime.fromisoformat(s.replace("Z", "+00:00"))
            dt_kh = dt.astimezone(CAMBODIA_TZ)
            return dt_kh.strftime("%d-%m-%Y")
        except Exception:
            parts = s[:10].split("-")
            if len(parts) == 3:
                return f"{parts[2]}-{parts[1]}-{parts[0]}"
    return s


def is_admin_user(user_id: Optional[int]) -> bool:
    if not user_id:
        return False
    uid = int(user_id)
    if uid in dynamic_admin_ids or uid in ENV_ADMIN_IDS:
        return True
    try:
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id")
            .eq("drama_title", CFG_ADMINS_KEY)
            .limit(1)
            .execute()
        )
        if res.data and res.data[0].get("cover_file_id"):
            for part in str(res.data[0]["cover_file_id"]).split(","):
                part = part.strip()
                if part.lstrip("-").isdigit():
                    dynamic_admin_ids.add(int(part))
    except Exception:
        pass
    return uid in dynamic_admin_ids


def is_internal_key(title: str) -> bool:
    if not title:
        return True
    t = str(title)
    return (
        t == REQ_LIMIT_KEY
        or t == USER_START_KEY
        or t.startswith(CAT_STORE_PREFIX)
        or t.startswith(SYN_STORE_PREFIX)
        or t.startswith(FAV_STORE_PREFIX)
        or t.startswith(RATE_STORE_PREFIX)
        or t.startswith(LANG_STORE_PREFIX)
        or t.startswith(CODE_STORE_PREFIX)
        or t.startswith(DATE_STORE_PREFIX)
        or t.startswith(DUR_STORE_PREFIX)
        or t.startswith(TAGS_STORE_PREFIX)
        or t.startswith(GIF_STORE_PREFIX)
        or t.startswith(ALBUM_STORE_PREFIX)
        or t.startswith(EP_COV_STORE_PREFIX)
        or t.startswith("__CFG__")
        or t.startswith(NOTIF_WATCH_PREFIX)
        or t.startswith("__REQ")
    )


def get_required_channel_config(force_refresh: bool = False) -> tuple[int, str]:
    """
    Point 4: Dynamically loads REQUIRED_CHANNEL_ID and REQUIRED_CHANNEL_LINK from Supabase (`drama_covers`)
    so Admin can change the Force-Join Channel/Group ID and Invite Link anytime via Admin Bot Chat.
    """
    now = time.time()
    with dynamic_req_channel_lock:
        if not force_refresh and (now - DYNAMIC_REQ_CHANNEL_CACHE["timestamp"]) < REQ_CHANNEL_CACHE_TTL:
            return int(DYNAMIC_REQ_CHANNEL_CACHE["channel_id"]), str(DYNAMIC_REQ_CHANNEL_CACHE["channel_link"])

    cid = REQUIRED_CHANNEL_ID
    clink = REQUIRED_CHANNEL_LINK
    try:
        res = (
            supabase.table("drama_covers")
            .select("drama_title, cover_file_id")
            .in_(
                "drama_title",
                [CFG_REQ_CHANNEL_JSON_KEY, CFG_REQ_CHANNEL_ID_KEY, CFG_REQ_CHANNEL_LINK_KEY],
            )
            .execute()
        )
        for row in (res.data or []):
            k = (row.get("drama_title") or "").strip()
            val = (row.get("cover_file_id") or "").strip()
            if not val:
                continue
            if k == CFG_REQ_CHANNEL_JSON_KEY:
                try:
                    parsed = json.loads(val)
                    if isinstance(parsed, dict):
                        raw_id = str(parsed.get("channel_id") or "").strip()
                        raw_link = str(parsed.get("channel_link") or "").strip()
                        if raw_id.lstrip("-").isdigit():
                            cid = int(raw_id)
                        if raw_link.startswith(("http://", "https://", "tg://")):
                            clink = raw_link
                except Exception:
                    pass
            elif k == CFG_REQ_CHANNEL_ID_KEY and val.lstrip("-").isdigit():
                cid = int(val)
            elif k == CFG_REQ_CHANNEL_LINK_KEY and val.startswith(("http://", "https://", "tg://")):
                clink = val
    except Exception:
        log.warning("Could not fetch dynamic required channel config from Supabase; using fallback")

    with dynamic_req_channel_lock:
        DYNAMIC_REQ_CHANNEL_CACHE["channel_id"] = cid
        DYNAMIC_REQ_CHANNEL_CACHE["channel_link"] = clink
        DYNAMIC_REQ_CHANNEL_CACHE["timestamp"] = now

    return int(cid), str(clink)


def get_required_channel_id() -> int:
    cid, _ = get_required_channel_config()
    return cid


def get_required_channel_link() -> str:
    _, clink = get_required_channel_config()
    return clink


def save_required_channel_config_in_db(channel_id: Optional[int] = None, channel_link: Optional[str] = None) -> tuple[int, str]:
    cur_id, cur_link = get_required_channel_config(force_refresh=True)
    new_id = int(channel_id) if channel_id is not None else cur_id
    new_link = (channel_link or "").strip() if channel_link is not None else cur_link
    if not new_link:
        new_link = cur_link

    try:
        supabase.table("drama_covers").upsert(
            [
                {
                    "drama_title": CFG_REQ_CHANNEL_ID_KEY,
                    "cover_file_id": str(new_id),
                    "cover_channel_msg_id": 0,
                },
                {
                    "drama_title": CFG_REQ_CHANNEL_LINK_KEY,
                    "cover_file_id": new_link,
                    "cover_channel_msg_id": 0,
                },
                {
                    "drama_title": CFG_REQ_CHANNEL_JSON_KEY,
                    "cover_file_id": json.dumps(
                        {"channel_id": new_id, "channel_link": new_link},
                        ensure_ascii=False,
                    ),
                    "cover_channel_msg_id": 0,
                },
            ],
            on_conflict="drama_title",
        ).execute()
    except Exception:
        log.exception("Failed to save required channel config in DB")

    with dynamic_req_channel_lock:
        DYNAMIC_REQ_CHANNEL_CACHE["channel_id"] = new_id
        DYNAMIC_REQ_CHANNEL_CACHE["channel_link"] = new_link
        DYNAMIC_REQ_CHANNEL_CACHE["timestamp"] = time.time()

    return new_id, new_link


def register_user_bot_config_in_db():
    try:
        if BOT_TOKEN:
            res = (
                supabase.table("drama_covers")
                .select("drama_title")
                .eq("drama_title", CFG_USER_BOT_TOKEN_KEY)
                .limit(1)
                .execute()
            )
            payload = {
                "drama_title": CFG_USER_BOT_TOKEN_KEY,
                "cover_file_id": BOT_TOKEN,
                "cover_channel_msg_id": 0,
            }
            if res.data:
                supabase.table("drama_covers").update(payload).eq("drama_title", CFG_USER_BOT_TOKEN_KEY).execute()
            else:
                supabase.table("drama_covers").insert(payload).execute()

        app_url = build_mini_app_url()
        if app_url:
            clean_url = app_url.split("?")[0]
            res_u = (
                supabase.table("drama_covers")
                .select("drama_title")
                .eq("drama_title", CFG_MINI_APP_URL_KEY)
                .limit(1)
                .execute()
            )
            payload_u = {
                "drama_title": CFG_MINI_APP_URL_KEY,
                "cover_file_id": clean_url,
                "cover_channel_msg_id": 0,
            }
            if res_u.data:
                supabase.table("drama_covers").update(payload_u).eq("drama_title", CFG_MINI_APP_URL_KEY).execute()
            else:
                supabase.table("drama_covers").insert(payload_u).execute()
    except Exception:
        log.warning("Could not register user bot config in DB")


# --------------------------------------------------
# Track /start Users, Notification Broadcast & Analytics Helpers
# --------------------------------------------------

def record_user_start(user_id: Optional[int]):
    if not user_id:
        return
    uid = int(user_id)
    today_int = get_cambodia_date_int()

    def _bg():
        try:
            res = (
                supabase.table("user_history")
                .select("last_ep")
                .eq("user_id", uid)
                .eq("drama_title", USER_START_KEY)
                .limit(1)
                .execute()
            )
            if not res.data:
                supabase.table("user_history").upsert(
                    {
                        "user_id": uid,
                        "drama_title": USER_START_KEY,
                        "last_ep": today_int,
                    },
                    on_conflict="user_id,drama_title",
                ).execute()
        except Exception:
            log.warning("Could not record user /start for %s", uid)

    threading.Thread(target=_bg, daemon=True).start()


def get_all_started_user_ids() -> list[int]:
    uids = set()
    try:
        res = supabase.table("user_history").select("user_id, drama_title").execute()
        for r in (res.data or []):
            uid = r.get("user_id")
            t = str(r.get("drama_title") or "")
            if uid is not None:
                uid_int = int(uid)
                if uid_int > 100000 and not (t.startswith(RATE_STORE_PREFIX) and uid_int < 100000000):
                    uids.add(uid_int)
    except Exception:
        log.exception("Could not fetch started user IDs")
    return sorted(list(uids))


def get_notification_stats_from_db() -> dict:
    default_stats = {
        "total_broadcasts": 0,
        "total_delivered": 0,
        "total_failed": 0,
        "last_drama": "មិនទាន់មាន",
        "last_sent_ok": 0,
        "last_failed": 0,
        "last_time": "មិនទាន់មាន",
    }
    try:
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id")
            .eq("drama_title", NOTIF_STATS_KEY)
            .limit(1)
            .execute()
        )
        if res.data and res.data[0].get("cover_file_id"):
            parsed = json.loads(res.data[0]["cover_file_id"])
            if isinstance(parsed, dict):
                default_stats.update(parsed)
    except Exception:
        pass
    return default_stats


def save_notification_stats_to_db(drama_title: str, sent_ok: int, failed_count: int):
    stats = get_notification_stats_from_db()
    stats["total_broadcasts"] = int(stats.get("total_broadcasts", 0)) + 1
    stats["total_delivered"] = int(stats.get("total_delivered", 0)) + int(sent_ok)
    stats["total_failed"] = int(stats.get("total_failed", 0)) + int(failed_count)
    stats["last_drama"] = drama_title
    stats["last_sent_ok"] = int(sent_ok)
    stats["last_failed"] = int(failed_count)
    stats["last_time"] = get_cambodia_now().strftime("%d-%m-%Y %H:%M")

    serialized = json.dumps(stats, ensure_ascii=False)
    try:
        res = (
            supabase.table("drama_covers")
            .select("drama_title")
            .eq("drama_title", NOTIF_STATS_KEY)
            .limit(1)
            .execute()
        )
        payload = {
            "drama_title": NOTIF_STATS_KEY,
            "cover_file_id": serialized,
            "cover_channel_msg_id": 0,
        }
        if res.data:
            supabase.table("drama_covers").update(payload).eq("drama_title", NOTIF_STATS_KEY).execute()
        else:
            supabase.table("drama_covers").insert(payload).execute()
    except Exception:
        log.exception("Failed to save notification stats")


def create_notification_watch_token(drama_title: str, ep_number: int) -> Optional[str]:
    """Persist a short callback token so notification buttons survive bot restarts."""
    try:
        token = secrets.token_hex(5)
        key = f"{NOTIF_WATCH_PREFIX}{token}"
        payload = {
            "drama_title": key,
            "cover_file_id": json.dumps(
                {"drama_title": drama_title, "ep_number": int(ep_number)},
                ensure_ascii=False,
            ),
            "cover_channel_msg_id": 0,
        }
        supabase.table("drama_covers").upsert(
            payload, on_conflict="drama_title"
        ).execute()
        return token
    except Exception:
        log.exception("Could not create notification watch token for %s", drama_title)
        return None


def get_notification_watch_target(token: str) -> Optional[tuple[str, int]]:
    """Resolve a notification callback token from Supabase."""
    if not token or not re.fullmatch(r"[0-9a-f]{10}", token):
        return None
    try:
        key = f"{NOTIF_WATCH_PREFIX}{token}"
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id")
            .eq("drama_title", key)
            .limit(1)
            .execute()
        )
        if not res.data or not res.data[0].get("cover_file_id"):
            return None
        payload = json.loads(res.data[0]["cover_file_id"])
        title = str(payload.get("drama_title") or "").strip()
        ep_number = int(payload.get("ep_number") or 0)
        if not title or ep_number < 1:
            return None
        return title, ep_number
    except Exception:
        log.exception("Could not resolve notification watch token")
        return None


def execute_user_bot_broadcast_async(
    drama_title: str,
    admin_chat_id: Optional[int] = None,
    custom_cover_msg_id: Optional[int] = None,
    custom_cover_file_id: Optional[str] = None,
):
    """
    Point 3.2: Sends Notification to User Bot Chat as a Cover Photo (supports Admin uploading
    a new custom cover before broadcasting or using the existing drama cover), including
    Drama Title + ID Code (e.g. XF-000001) and the '▶️ មើលឥឡូវនេះ' button.
    """
    def _worker():
        try:
            user_ids = get_all_started_user_ids()
            single_meta = fetch_single_drama_metadata(drama_title)
            drama_code = (single_meta.get("drama_code") or "").strip()
            if not drama_code:
                try:
                    meta_maps = fetch_all_drama_metadata_maps()
                    drama_code = (meta_maps.get("codes", {}).get(drama_title) or "").strip()
                except Exception:
                    drama_code = ""

            cov_res = (
                supabase.table("drama_covers")
                .select("cover_file_id, cover_channel_msg_id")
                .eq("drama_title", drama_title)
                .limit(1)
                .execute()
            )
            db_cover_msg_id = cov_res.data[0].get("cover_channel_msg_id") if cov_res.data else None
            db_cover_file_id = cov_res.data[0].get("cover_file_id") if cov_res.data else None

            cover_msg_id = int(custom_cover_msg_id) if custom_cover_msg_id else (int(db_cover_msg_id) if db_cover_msg_id else None)
            cover_file_id = (custom_cover_file_id or "").strip() or (db_cover_file_id or "").strip()

            # Force-join notification: show ONLY the Channel button first.
            # The Watch button is revealed after Telegram confirms membership.
            join_markup = InlineKeyboardMarkup()
            watch_token = None
            target_ep_num = 1
            try:
                available_eps = get_episode_rows(drama_title)
                first_video_ep = next(
                    (
                        row for row in available_eps
                        if row.get("channel_msg_id")
                        and int(row.get("ep_number") or 0) > 0
                    ),
                    None,
                )
                target_ep_num = int(first_video_ep["ep_number"]) if first_video_ep else 1
                watch_token = create_notification_watch_token(drama_title, target_ep_num)
                join_markup.add(
                    InlineKeyboardButton(
                        f"📢 ចូល Channel {CHANNEL_DISPLAY_NAME}",
                        url=get_required_channel_link(),
                    )
                )
            except Exception:
                log.exception("Could not prepare notification join button for %s", drama_title)

            if not join_markup.keyboard:
                join_markup = None

            escaped_title = html.escape(drama_title)
            code_text = html.escape(drama_code or "XF-000001")
            duration_line = (
                f" | ⏱️ រយៈពេល៖ {html.escape(single_meta['duration_text'])}"
                if single_meta.get("duration_text") else ""
            )
            caption_text = (
                f"🔥 <b>រឿងថ្មីទើបមកដល់ក្នុង {CHANNEL_DISPLAY_NAME}!</b>\n"
                f"🎬 ចំណងជើង៖ <b>{escaped_title}</b>\n"
                f"🆔 លេខកូដ៖ <code>{code_text}</code>{duration_line}\n"
                f"👇 ចុចប៊ូតុងខាងក្រោម ដើម្បីទទួលវីដេអូ!"
            )

            # Ensure cover_file_id is valid for this User Bot Token if we need send_photo fallback
            valid_photo_fid = None
            if cover_file_id:
                try:
                    bot.get_file(cover_file_id)
                    valid_photo_fid = cover_file_id
                except Exception:
                    valid_photo_fid = None
            if not valid_photo_fid and cover_msg_id:
                valid_photo_fid = refresh_cover_file_id_from_channel(
                    drama_title if not custom_cover_msg_id else f"__NOTIF_COV__{drama_title}",
                    int(cover_msg_id),
                )

            sent_ok = 0
            failed_count = 0

            for uid in user_ids:
                delivered = False
                is_member_now = is_required_channel_member(int(uid))
                markup = (
                    InlineKeyboardMarkup().add(
                        InlineKeyboardButton(
                            "▶️ មើលឥឡូវនេះ",
                            callback_data=f"notif_watch_{watch_token}",
                        )
                    )
                    if is_member_now and watch_token
                    else join_markup
                )
                sent_message_id = None

                if cover_msg_id:
                    try:
                        sent_copy = bot.copy_message(
                            chat_id=uid,
                            from_chat_id=CHANNEL_ID,
                            message_id=int(cover_msg_id),
                            caption=caption_text,
                            parse_mode="HTML",
                            reply_markup=markup,
                        )
                        sent_message_id = getattr(sent_copy, "message_id", None)
                        delivered = True
                    except Exception:
                        delivered = False

                if not delivered and valid_photo_fid:
                    try:
                        sent_photo = bot.send_photo(
                            chat_id=uid,
                            photo=valid_photo_fid,
                            caption=caption_text,
                            parse_mode="HTML",
                            reply_markup=markup,
                        )
                        sent_message_id = getattr(sent_photo, "message_id", None)
                        delivered = True
                    except Exception:
                        delivered = False

                if not delivered:
                    try:
                        sent_text = bot.send_message(
                            chat_id=uid,
                            text=caption_text,
                            parse_mode="HTML",
                            reply_markup=markup,
                        )
                        sent_message_id = getattr(sent_text, "message_id", None)
                        delivered = True
                    except Exception:
                        delivered = False

                if delivered:
                    if watch_token and sent_message_id and not is_member_now:
                        PENDING_WATCH_AFTER_JOIN[int(uid)] = {
                            "drama_title": drama_title,
                            "ep_number": int(target_ep_num),
                            "notif_msg_id": int(sent_message_id),
                            "watch_token": watch_token,
                        }
                    sent_ok += 1
                else:
                    failed_count += 1

                time.sleep(0.05)

            save_notification_stats_to_db(drama_title, sent_ok, failed_count)
            log.info("User Bot Notification broadcast finished for %s: %s sent, %s failed", drama_title, sent_ok, failed_count)
        except Exception:
            log.exception("User Bot broadcast worker failed")

    threading.Thread(target=_worker, daemon=True).start()


def check_and_process_notif_queue():
    try:
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id")
            .eq("drama_title", CFG_NOTIF_QUEUE_KEY)
            .limit(1)
            .execute()
        )
        if res.data and res.data[0].get("cover_file_id"):
            raw = res.data[0]["cover_file_id"]
            supabase.table("drama_covers").delete().eq("drama_title", CFG_NOTIF_QUEUE_KEY).execute()
            job = json.loads(raw)
            drama_title = (job.get("drama_title") or "").strip()
            admin_chat_id = job.get("admin_chat_id")
            custom_cover_msg_id = job.get("custom_cover_msg_id")
            custom_cover_file_id = job.get("custom_cover_file_id")
            if drama_title:
                execute_user_bot_broadcast_async(
                    drama_title=drama_title,
                    admin_chat_id=admin_chat_id,
                    custom_cover_msg_id=custom_cover_msg_id,
                    custom_cover_file_id=custom_cover_file_id,
                )
    except Exception:
        pass


def compute_user_analytics_payload() -> dict:
    today_int = get_cambodia_date_int()
    seven_days_ago_int = int((get_cambodia_now() - timedelta(days=7)).strftime("%Y%m%d"))
    today_iso_prefix = get_cambodia_now().strftime("%Y-%m-%d")

    all_users = set()
    new_users_today = set()
    new_users_7d = set()
    active_today = set()

    try:
        res = supabase.table("user_history").select("user_id, drama_title, last_ep, updated_at").execute()
        for r in (res.data or []):
            uid = r.get("user_id")
            if not uid:
                continue
            uid_int = int(uid)
            t = str(r.get("drama_title") or "")
            if t.startswith(RATE_STORE_PREFIX) and uid_int < 100000000:
                continue

            all_users.add(uid_int)

            if t == USER_START_KEY:
                join_date = int(r.get("last_ep") or 0)
                if join_date == today_int:
                    new_users_today.add(uid_int)
                if join_date >= seven_days_ago_int:
                    new_users_7d.add(uid_int)

            updated_raw = str(r.get("updated_at") or "")
            if updated_raw.startswith(today_iso_prefix):
                active_today.add(uid_int)
    except Exception:
        log.exception("Failed to compute user analytics in user backend")

    total_dramas = 0
    total_episodes = 0
    try:
        ep_res = supabase.table("episodes").select("drama_title, ep_number").execute()
        rows = ep_res.data or []
        total_episodes = len(rows)
        titles_set = {
            (r.get("drama_title") or "").strip()
            for r in rows
            if (r.get("drama_title") or "").strip() and not is_internal_key(r.get("drama_title"))
        }
        total_dramas = len(titles_set)
    except Exception:
        pass

    return {
        "total_users": len(all_users),
        "new_users_today": len(new_users_today),
        "new_users_7d": len(new_users_7d),
        "active_users_today": len(active_today),
        "total_dramas": total_dramas,
        "total_episodes": total_episodes,
        "notif_stats": get_notification_stats_from_db(),
    }


# --------------------------------------------------
# Drama CODE (XF-000001) & Upload Date Helpers
# --------------------------------------------------

def format_drama_code(num: int) -> str:
    return f"XF-{int(num):06d}"


def parse_code_number(code_str: str) -> int:
    if not code_str:
        return 0
    m = re.match(r"^XF-(\d+)$", str(code_str).strip().upper())
    if m:
        return int(m.group(1))
    return 0


def save_drama_code_in_db(drama_title: str, drama_code: str) -> str:
    clean_title = (drama_title or "").strip()
    clean_code = (drama_code or "").strip().upper()
    if not clean_title or not clean_code:
        return clean_code
    key = f"{CODE_STORE_PREFIX}{clean_title}"
    try:
        res = (
            supabase.table("drama_covers")
            .select("drama_title")
            .eq("drama_title", key)
            .limit(1)
            .execute()
        )
        payload = {
            "drama_title": key,
            "cover_file_id": clean_code,
            "cover_channel_msg_id": 0,
        }
        if res.data:
            supabase.table("drama_covers").update(payload).eq("drama_title", key).execute()
        else:
            supabase.table("drama_covers").insert(payload).execute()
    except Exception:
        pass
    try:
        supabase.table("drama_covers").update({"drama_code": clean_code}).eq("drama_title", clean_title).execute()
    except Exception:
        pass
    return clean_code


def save_drama_upload_date_in_db(drama_title: str, date_ddmmyyyy: str) -> str:
    clean_title = (drama_title or "").strip()
    clean_date = normalize_date_to_ddmmyyyy(date_ddmmyyyy) or get_cambodia_date_ddmmyyyy()
    if not clean_title:
        return clean_date
    key = f"{DATE_STORE_PREFIX}{clean_title}"
    try:
        res = (
            supabase.table("drama_covers")
            .select("drama_title")
            .eq("drama_title", key)
            .limit(1)
            .execute()
        )
        if not res.data:
            supabase.table("drama_covers").insert({
                "drama_title": key,
                "cover_file_id": clean_date,
                "cover_channel_msg_id": 0,
            }).execute()
    except Exception:
        pass
    return clean_date


def format_video_duration(seconds: int) -> str:
    seconds = max(0, int(seconds or 0))
    hours, remainder = divmod(seconds, 3600)
    minutes, secs = divmod(remainder, 60)
    if hours:
        return f"{hours}:{minutes:02d}:{secs:02d}"
    return f"{minutes}:{secs:02d}"


def build_smart_search_tokens(
    title: str,
    drama_code: str,
    category: str,
    synopsis: str,
    tags: str,
) -> str:
    combined = f"{title} {drama_code} {category} {synopsis} {tags}".lower()
    extra_keywords = []

    bilingual_pairs = [
        (["ខ្មែរ", "រឿងខ្មែរ", "រឿងសិចខ្មែរ", "khmer", "cambodia", "cambo"], ["ខ្មែរ", "រឿងខ្មែរ", "រឿងសិចខ្មែរ", "khmer"]),
        (["ចិន", "រឿងចិន", "chinese", "china"], ["ចិន", "រឿងចិន", "chinese", "china"]),
        (["កូរ៉េ", "រឿងកូរ៉េ", "korean", "korea"], ["កូរ៉េ", "រឿងកូរ៉េ", "korean", "korea"]),
        (["ជប៉ុន", "រឿងជប៉ុន", "japanese", "japan", "jav"], ["ជប៉ុន", "រឿងជប៉ុន", "japanese", "japan"]),
        (["ថៃ", "រឿងថៃ", "thai", "thailand"], ["ថៃ", "រឿងថៃ", "thai"]),
        (["អឺរ៉ុប", "អាមេរិក", "បរទេស", "western", "english", "usa"], ["អឺរ៉ុប", "បរទេស", "western", "english"]),
        (["ស្នេហា", "រ៉ូមែនទិក", "សិចស៊ី", "រឿងសិច", "romance", "romantic", "sexy", "love", "adult", "18+"], ["ស្នេហា", "រ៉ូមែនទិក", "romance", "romantic"]),
        (["វាយតប់", "សកម្មភាព", "action"], ["វាយតប់", "សកម្មភាព", "action"]),
        (["ai", "រឿងai", "រឿង ai"], ["ai", "រឿងai", "រឿង ai"]),
    ]

    for triggers, expansions in bilingual_pairs:
        if any(t in combined for t in triggers):
            extra_keywords.extend(expansions)

    if drama_code:
        code_low = drama_code.lower()
        extra_keywords.append(code_low)
        extra_keywords.append(code_low.replace("-", ""))
        num_part = code_low.replace("xf-", "")
        if num_part:
            extra_keywords.append(num_part)
            if num_part.isdigit():
                extra_keywords.append(str(int(num_part)))

    return " ".join(dict.fromkeys([title, drama_code, category, synopsis, tags] + extra_keywords))


# --------------------------------------------------
# Daily Request Limit & Admin Group Helpers
# --------------------------------------------------

def has_user_requested_today(user_id: Optional[int]) -> bool:
    if not user_id:
        return False
    today_int = get_cambodia_date_int()
    try:
        res = (
            supabase.table("user_history")
            .select("last_ep")
            .eq("user_id", int(user_id))
            .eq("drama_title", REQ_LIMIT_KEY)
            .limit(1)
            .execute()
        )
        if res.data and res.data[0].get("last_ep") is not None:
            return int(res.data[0]["last_ep"]) == today_int
    except Exception:
        log.warning("Could not check daily request limit for user %s", user_id)
    return False


def mark_user_requested_today(user_id: Optional[int]):
    if not user_id:
        return
    today_int = get_cambodia_date_int()
    try:
        supabase.table("user_history").upsert(
            {
                "user_id": int(user_id),
                "drama_title": REQ_LIMIT_KEY,
                "last_ep": today_int,
            },
            on_conflict="user_id,drama_title",
        ).execute()
    except Exception:
        log.warning("Could not save daily request limit for user %s", user_id)


def load_saved_admin_group_id() -> Optional[int]:
    global detected_admin_group_id
    if detected_admin_group_id:
        return detected_admin_group_id
    try:
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id")
            .eq("drama_title", CFG_ADMIN_GROUP_KEY)
            .limit(1)
            .execute()
        )
        if res.data and res.data[0].get("cover_file_id"):
            raw_id = str(res.data[0]["cover_file_id"]).strip()
            if raw_id.lstrip("-").isdigit():
                detected_admin_group_id = int(raw_id)
                return detected_admin_group_id
    except Exception:
        pass
    return None


def save_detected_admin_group_id(chat_id: int):
    global detected_admin_group_id
    if not chat_id or int(chat_id) == int(CHANNEL_ID):
        return
    detected_admin_group_id = int(chat_id)
    try:
        supabase.table("drama_covers").upsert(
            {
                "drama_title": CFG_ADMIN_GROUP_KEY,
                "cover_file_id": str(chat_id),
            },
            on_conflict="drama_title",
        ).execute()
    except Exception:
        pass


def remember_admin_request_msg(
    chat_id: int,
    msg_id: int,
    target_uid: int,
    drama_title: str = "",
    request_text: str = "",
):
    with request_map_lock:
        key = (int(chat_id), int(msg_id))
        admin_request_msg_map[key] = {
            "uid": int(target_uid),
            "drama_title": (drama_title or "").strip(),
            "request_text": (request_text or "").strip(),
        }
        admin_request_msg_map.move_to_end(key)
        while len(admin_request_msg_map) > MAX_REQUEST_MAP_SIZE:
            admin_request_msg_map.popitem(last=False)


def get_mapped_info_from_admin_msg(chat_id: int, msg_id: int) -> Optional[dict]:
    with request_map_lock:
        info = admin_request_msg_map.get((int(chat_id), int(msg_id)))
        if isinstance(info, int):
            return {"uid": info, "drama_title": "", "request_text": ""}
        return info


def get_mapped_user_from_admin_msg(chat_id: int, msg_id: int) -> Optional[int]:
    info = get_mapped_info_from_admin_msg(chat_id, msg_id)
    if info and info.get("uid"):
        return int(info["uid"])
    return None


def safe_delete_one(chat_id: int, msg_id: int):
    try:
        bot.delete_message(chat_id, int(msg_id))
    except Exception:
        pass


def track_trash_message(user_id: int, message_id: int):
    if not message_id:
        return
    if message_id == user_main_menu_id.get(user_id):
        return
    if user_id not in user_trash_messages:
        user_trash_messages[user_id] = set()
    user_trash_messages[user_id].add(int(message_id))


def clear_user_cover_message(user_id: int):
    old_cov_id = user_cover_message.pop(user_id, None)
    user_cover_drama.pop(user_id, None)
    if old_cov_id:
        safe_delete_one(user_id, old_cov_id)
        if user_id in user_trash_messages:
            user_trash_messages[user_id].discard(int(old_cov_id))


def get_episode_rows(drama_title):
    response = (
        supabase.table("episodes")
        .select("ep_number, channel_msg_id")
        .eq("drama_title", drama_title)
        .order("ep_number")
        .execute()
    )
    return response.data or []


def check_drama_has_multiple_episodes(drama_title: str) -> bool:
    if not drama_title:
        return False
    try:
        res = (
            supabase.table("episodes")
            .select("ep_number")
            .eq("drama_title", drama_title)
            .limit(2)
            .execute()
        )
        return len(res.data or []) > 1
    except Exception:
        log.warning("Could not check episode count for drama: %s", drama_title)
        return False


def refresh_cover_file_id_from_channel(drama_title_key: str, cover_msg_id: int) -> Optional[str]:
    """
    Point 1 Fix: Thread-safe cover_file_id refresh when Bot Token changes.
    1) Serializes refresh calls via `cover_refresh_lock` to avoid concurrent forward_message rate limits on Home Page load.
    2) Checks if another thread already refreshed `cover_file_id` in Supabase while waiting for the lock.
    3) Probes using `CHANNEL_ID` first (since Bot is already Admin in CHANNEL_ID, works 100% even if Admin hasn't sent /start to new Bot),
       then falls back to `load_saved_admin_group_id()` and `ADMIN_ID`.
    """
    if not cover_msg_id:
        return None

    with cover_refresh_lock:
        try:
            check_res = (
                supabase.table("drama_covers")
                .select("cover_file_id")
                .eq("drama_title", drama_title_key)
                .limit(1)
                .execute()
            )
            existing_fid = check_res.data[0].get("cover_file_id") if check_res.data else None
            if existing_fid:
                try:
                    info = bot.get_file(existing_fid)
                    if info and getattr(info, "file_path", None):
                        return existing_fid
                except Exception:
                    pass
        except Exception:
            pass

        probe_targets = []
        for candidate in [CHANNEL_ID, load_saved_admin_group_id(), ADMIN_ID]:
            if candidate and int(candidate) not in probe_targets:
                probe_targets.append(int(candidate))

        for target_probe_chat in probe_targets:
            try:
                fwd = bot.forward_message(
                    chat_id=target_probe_chat,
                    from_chat_id=CHANNEL_ID,
                    message_id=int(cover_msg_id),
                    disable_notification=True,
                )
                new_file_id = None
                if getattr(fwd, "photo", None):
                    new_file_id = fwd.photo[-1].file_id
                elif getattr(fwd, "animation", None):
                    new_file_id = fwd.animation.file_id
                elif getattr(fwd, "document", None):
                    d = fwd.document
                    if getattr(d, "thumbnail", None):
                        new_file_id = d.thumbnail.file_id
                    elif getattr(d, "thumb", None):
                        new_file_id = d.thumb.file_id
                    else:
                        new_file_id = d.file_id
                elif getattr(fwd, "video", None):
                    v = fwd.video
                    if getattr(v, "thumbnail", None):
                        new_file_id = v.thumbnail.file_id
                    elif getattr(v, "thumb", None):
                        new_file_id = v.thumb.file_id

                try:
                    bot.delete_message(target_probe_chat, fwd.message_id)
                except Exception:
                    pass

                if new_file_id:
                    if not drama_title_key.startswith("__NOTIF_COV__"):
                        supabase.table("drama_covers").upsert(
                            {
                                "drama_title": drama_title_key,
                                "cover_file_id": new_file_id,
                                "cover_channel_msg_id": int(cover_msg_id),
                            },
                            on_conflict="drama_title",
                        ).execute()
                        invalidate_dramas_cache()
                    time.sleep(0.12)
                    return new_file_id
            except Exception as e:
                log.warning(
                    "Probe forward_message on chat %s failed for %s (%s)",
                    target_probe_chat,
                    drama_title_key,
                    e,
                )
                continue

    return None


def sync_all_covers_for_bot_token_async():
    """
    Point 1 Fix: Background worker on startup that checks all cover_file_id entries in `drama_covers`
    and automatically refreshes any outdated file_ids tied to a previous Bot Token.
    """
    def _worker():
        time.sleep(2.5)
        try:
            res = (
                supabase.table("drama_covers")
                .select("drama_title, cover_file_id, cover_channel_msg_id")
                .execute()
            )
            rows = res.data or []
            refreshed_count = 0
            for r in rows:
                t = (r.get("drama_title") or "").strip()
                mid = int(r.get("cover_channel_msg_id") or 0)
                fid = (r.get("cover_file_id") or "").strip()
                if not t or mid <= 0:
                    continue
                if t.startswith("__CFG__") or t.startswith(NOTIF_WATCH_PREFIX):
                    continue
                needs_refresh = False
                if not fid:
                    needs_refresh = True
                else:
                    try:
                        bot.get_file(fid)
                    except Exception:
                        needs_refresh = True
                if needs_refresh:
                    new_fid = refresh_cover_file_id_from_channel(t, mid)
                    if new_fid:
                        refreshed_count += 1
                    time.sleep(0.25)
            if refreshed_count > 0:
                invalidate_dramas_cache()
                log.info("Auto-synced %s cover_file_id records for current Bot Token.", refreshed_count)
        except Exception:
            log.exception("Background cover sync worker encountered an error")

    threading.Thread(target=_worker, name="cover-token-sync", daemon=True).start()


def extract_and_cache_episode_cover_from_channel(
    drama_title: str,
    ep_number: int,
    channel_msg_id: int,
) -> Optional[bytes]:
    """
    ទាញយករូប Thumbnail ឬ Frame ផ្ទាល់ពីវីដេអូនៃភាគនីមួយៗក្នុង Channel (channel_msg_id)
    ហើយរក្សាទុកក្នុង __EPCOV__ ដើម្បីឱ្យភាគនីមួយៗបង្ហាញរូប Cover របស់វីដេអូភាគនោះពិតៗ។
    """
    if not channel_msg_id:
        return None

    ep_key = f"{EP_COV_STORE_PREFIX}{drama_title}__{int(ep_number)}"
    target_probe_chat = load_saved_admin_group_id() or ADMIN_ID

    with ep_extract_lock:
        try:
            fwd = bot.forward_message(
                chat_id=target_probe_chat,
                from_chat_id=CHANNEL_ID,
                message_id=int(channel_msg_id),
                disable_notification=True,
            )
            thumb_fid = None
            video_fid = None
            video_size = 0

            if getattr(fwd, "video", None):
                v = fwd.video
                video_fid = v.file_id
                video_size = int(v.file_size or 0)
                if getattr(v, "thumbnail", None):
                    thumb_fid = v.thumbnail.file_id
                elif getattr(v, "thumb", None):
                    thumb_fid = v.thumb.file_id
            elif getattr(fwd, "document", None):
                d = fwd.document
                video_fid = d.file_id
                video_size = int(d.file_size or 0)
                if getattr(d, "thumbnail", None):
                    thumb_fid = d.thumbnail.file_id
                elif getattr(d, "thumb", None):
                    thumb_fid = d.thumb.file_id
            elif getattr(fwd, "photo", None):
                thumb_fid = fwd.photo[-1].file_id

            try:
                bot.delete_message(target_probe_chat, fwd.message_id)
            except Exception:
                pass

            # 1. បើវីដេអូមាន Thumbnail របស់វាស្រាប់ ទាញយកភ្លាមៗ
            if thumb_fid:
                try:
                    f_info = bot.get_file(thumb_fid)
                    download_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{f_info.file_path}"
                    img_res = requests.get(download_url, timeout=20)
                    if img_res.status_code == 200 and img_res.content:
                        supabase.table("drama_covers").upsert(
                            {
                                "drama_title": ep_key,
                                "cover_file_id": thumb_fid,
                                "cover_channel_msg_id": int(channel_msg_id),
                            },
                            on_conflict="drama_title",
                        ).execute()
                        return img_res.content
                except Exception:
                    pass

            # 2. បើអត់មាន Thumbnail តែវីដេអូតូចជាង 20MB ហើយមាន cv2 កាត់យក Frame ពីវីដេអូផ្ទាល់
            if video_fid and (video_size == 0 or video_size <= 19_800_000) and cv2 is not None:
                tmp_path = None
                try:
                    v_info = bot.get_file(video_fid)
                    v_bytes = bot.download_file(v_info.file_path)
                    with tempfile.NamedTemporaryFile(suffix=".mp4", delete=False) as tmp:
                        tmp.write(v_bytes)
                        tmp_path = tmp.name

                    cap = cv2.VideoCapture(tmp_path)
                    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT) or 0)
                    if total_frames > 2:
                        target_idx = max(0, min(total_frames - 1, int(total_frames * 0.25)))
                        cap.set(cv2.CAP_PROP_POS_FRAMES, target_idx)
                        ok, frame = cap.read()
                        if ok and frame is not None:
                            rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
                            pil_img = Image.fromarray(rgb)
                            buf = io.BytesIO()
                            pil_img.save(buf, format="JPEG", quality=88)
                            jpg_bytes = buf.getvalue()
                            cap.release()

                            try:
                                bio = io.BytesIO(jpg_bytes)
                                bio.name = f"ep_{ep_number}_cover.jpg"
                                sent_cov = bot.send_photo(
                                    chat_id=CHANNEL_ID,
                                    photo=bio,
                                    caption=f"🖼️ Cover ភាគទី {ep_number}៖ {drama_title}",
                                )
                                if getattr(sent_cov, "photo", None):
                                    supabase.table("drama_covers").upsert(
                                        {
                                            "drama_title": ep_key,
                                            "cover_file_id": sent_cov.photo[-1].file_id,
                                            "cover_channel_msg_id": sent_cov.message_id,
                                        },
                                        on_conflict="drama_title",
                                    ).execute()
                            except Exception:
                                pass

                            return jpg_bytes
                    cap.release()
                except Exception:
                    pass
                finally:
                    if tmp_path and os.path.exists(tmp_path):
                        try:
                            os.remove(tmp_path)
                        except Exception:
                            pass
        except Exception:
            log.warning("Could not extract episode cover from channel msg %s", channel_msg_id)

    return None


def send_drama_cover_if_needed(user_id: int, drama_title: str, show_ep_buttons: bool = False):
    if user_cover_drama.get(user_id) == drama_title and user_cover_message.get(user_id):
        return

    clear_user_cover_message(user_id)

    try:
        res = (
            supabase.table("drama_covers")
            .select("cover_channel_msg_id, cover_file_id")
            .eq("drama_title", drama_title)
            .limit(1)
            .execute()
        )
    except Exception:
        log.exception("Supabase drama_covers query failed")
        return

    if not res.data:
        return

    cover_msg_id = res.data[0].get("cover_channel_msg_id")
    cover_file_id = res.data[0].get("cover_file_id")

    sent_cov = None
    markup = get_bottom_menu(user_id, has_active_drama=show_ep_buttons)

    if cover_msg_id:
        try:
            sent_cov = bot.copy_message(
                chat_id=user_id,
                from_chat_id=CHANNEL_ID,
                message_id=int(cover_msg_id),
                caption="",
                protect_content=True,
                reply_markup=markup,
            )
        except Exception:
            log.warning("Telegram copy_message for cover failed: %s", drama_title)

    if not sent_cov and cover_file_id:
        try:
            sent_cov = bot.send_photo(
                chat_id=user_id,
                photo=cover_file_id,
                protect_content=True,
                reply_markup=markup,
            )
        except Exception:
            log.warning("Fallback send_photo for cover failed: %s", drama_title)

    if sent_cov:
        user_cover_message[user_id] = sent_cov.message_id
        user_cover_drama[user_id] = drama_title
        track_trash_message(user_id, sent_cov.message_id)


def sweep_and_clean_chat(user_id: int, up_to_msg_id: int, keep_msg_id: int = None):
    trash_ids = set(user_trash_messages.get(user_id, set()))

    last_vid_id = user_last_message.pop(user_id, None)
    if last_vid_id:
        trash_ids.add(int(last_vid_id))

    last_cov_id = user_cover_message.pop(user_id, None)
    user_cover_drama.pop(user_id, None)
    if last_cov_id:
        trash_ids.add(int(last_cov_id))

    old_main_id = user_main_menu_id.get(user_id)
    if old_main_id and old_main_id != keep_msg_id:
        trash_ids.add(int(old_main_id))

    if up_to_msg_id:
        for mid in range(int(up_to_msg_id), max(1, int(up_to_msg_id) - 60), -1):
            trash_ids.add(mid)

    if keep_msg_id and keep_msg_id in trash_ids:
        trash_ids.discard(int(keep_msg_id))

    user_trash_messages[user_id] = set()

    if not trash_ids:
        return

    sorted_ids = sorted(list(trash_ids), reverse=True)

    def _bg_cleaner():
        if hasattr(bot, "delete_messages"):
            for i in range(0, len(sorted_ids), 100):
                chunk = sorted_ids[i : i + 100]
                try:
                    bot.delete_messages(user_id, chunk)
                except Exception:
                    pass

        with ThreadPoolExecutor(max_workers=10) as executor:
            for mid in sorted_ids:
                executor.submit(safe_delete_one, user_id, mid)

    threading.Thread(target=_bg_cleaner, daemon=True).start()


def make_cover_version(cover_file_id: str, cover_channel_msg_id=None) -> str:
    raw = f"{cover_file_id or ''}:{cover_channel_msg_id or ''}"
    if not raw.strip(":"):
        return "v1"
    return hashlib.sha1(raw.encode("utf-8")).hexdigest()[:12]


def get_cached_cover_bytes(cache_key: str):
    with cover_cache_lock:
        if cache_key in COVER_MEMORY_CACHE:
            COVER_MEMORY_CACHE.move_to_end(cache_key)
            return COVER_MEMORY_CACHE[cache_key]
    return None


def set_cached_cover_bytes(cache_key: str, content: bytes, media_type: str):
    with cover_cache_lock:
        COVER_MEMORY_CACHE[cache_key] = (content, media_type)
        COVER_MEMORY_CACHE.move_to_end(cache_key)
        while len(COVER_MEMORY_CACHE) > MAX_SERVER_COVER_CACHE:
            COVER_MEMORY_CACHE.popitem(last=False)


def get_cached_gif_bytes(cache_key: str):
    with gif_cache_lock:
        if cache_key in GIF_MEMORY_CACHE:
            GIF_MEMORY_CACHE.move_to_end(cache_key)
            return GIF_MEMORY_CACHE[cache_key]
    return None


def set_cached_gif_bytes(cache_key: str, content: bytes, media_type: str):
    with gif_cache_lock:
        GIF_MEMORY_CACHE[cache_key] = (content, media_type)
        GIF_MEMORY_CACHE.move_to_end(cache_key)
        while len(GIF_MEMORY_CACHE) > MAX_SERVER_GIF_CACHE:
            GIF_MEMORY_CACHE.popitem(last=False)


def fetch_all_drama_metadata_maps() -> dict:
    global detected_admin_group_id
    versions = {}
    synopses = {}
    langs = {}
    codes = {}
    upload_dates = {}
    durations = {}
    tags_map = {}
    gifs_map = {}
    categories = set()
    top_trending_titles = []

    try:
        cov_res = (
            supabase.table("drama_covers")
            .select("*")
            .execute()
        )
        for r in (cov_res.data or []):
            t = (r.get("drama_title") or "").strip()
            if not t:
                continue
            if t == CFG_ADMIN_GROUP_KEY:
                if not detected_admin_group_id and r.get("cover_file_id"):
                    try:
                        raw_grp = str(r["cover_file_id"]).strip()
                        if raw_grp.lstrip("-").isdigit():
                            detected_admin_group_id = int(raw_grp)
                    except Exception:
                        pass
                continue
            if t == CFG_ADMINS_KEY:
                if r.get("cover_file_id"):
                    for part in str(r["cover_file_id"]).split(","):
                        part = part.strip()
                        if part.lstrip("-").isdigit():
                            dynamic_admin_ids.add(int(part))
                continue
            if t == CFG_TRENDING_KEY:
                if r.get("cover_file_id"):
                    try:
                        parsed_tr = json.loads(r["cover_file_id"])
                        if isinstance(parsed_tr, list):
                            top_trending_titles = [str(x).strip() for x in parsed_tr if str(x).strip()]
                    except Exception:
                        pass
                continue
            if t.startswith(CAT_STORE_PREFIX):
                c_name = t[len(CAT_STORE_PREFIX):].strip()
                if c_name:
                    categories.add(c_name)
                continue
            if t.startswith(SYN_STORE_PREFIX):
                real_title = t[len(SYN_STORE_PREFIX):].strip()
                if real_title:
                    synopses[real_title] = (r.get("cover_file_id") or "").strip()
                continue
            if t.startswith(LANG_STORE_PREFIX):
                real_title = t[len(LANG_STORE_PREFIX):].strip()
                if real_title:
                    val = (r.get("cover_file_id") or "").strip().lower()
                    langs[real_title] = "sub" if val == "sub" else "dub"
                continue
            if t.startswith(CODE_STORE_PREFIX):
                real_title = t[len(CODE_STORE_PREFIX):].strip()
                val = (r.get("cover_file_id") or "").strip().upper()
                if real_title and val.startswith("XF-"):
                    codes[real_title] = val
                continue
            if t.startswith(DATE_STORE_PREFIX):
                real_title = t[len(DATE_STORE_PREFIX):].strip()
                val = normalize_date_to_ddmmyyyy(r.get("cover_file_id") or "")
                if real_title and val:
                    upload_dates[real_title] = val
                continue
            if t.startswith(DUR_STORE_PREFIX):
                real_title = t[len(DUR_STORE_PREFIX):].strip()
                val = str(r.get("cover_file_id") or "").strip()
                if real_title and val.isdigit():
                    durations[real_title] = int(val)
                continue
            if t.startswith(TAGS_STORE_PREFIX):
                real_title = t[len(TAGS_STORE_PREFIX):].strip()
                if real_title:
                    tags_map[real_title] = (r.get("cover_file_id") or "").strip()
                continue
            if t.startswith(GIF_STORE_PREFIX):
                real_title = t[len(GIF_STORE_PREFIX):].strip()
                fid = (r.get("cover_file_id") or "").strip()
                mid = r.get("cover_channel_msg_id") or ""
                if real_title and (fid or mid):
                    gifs_map[real_title] = make_cover_version(fid, mid)
                continue
            if is_internal_key(t):
                continue

            fid = r.get("cover_file_id") or ""
            mid = r.get("cover_channel_msg_id") or ""
            versions[t] = make_cover_version(fid, mid)

            if r.get("drama_code") and t not in codes:
                c_val = str(r.get("drama_code")).strip().upper()
                if c_val.startswith("XF-"):
                    codes[t] = c_val
            if r.get("uploaded_at") and t not in upload_dates:
                d_val = normalize_date_to_ddmmyyyy(r.get("uploaded_at"))
                if d_val:
                    upload_dates[t] = d_val
            if r.get("tags") and t not in tags_map:
                tags_map[t] = str(r.get("tags")).strip()
    except Exception:
        log.warning("Could not fetch metadata map from drama_covers")

    return {
        "versions": versions,
        "synopses": synopses,
        "langs": langs,
        "codes": codes,
        "upload_dates": upload_dates,
        "durations": durations,
        "tags": tags_map,
        "gifs": gifs_map,
        "categories": categories,
        "top_trending_titles": top_trending_titles,
    }


def ensure_codes_and_dates_for_dramas(dramas_dict: OrderedDict, meta: dict):
    codes_map = meta["codes"]
    dates_map = meta["upload_dates"]

    used_numbers = {parse_code_number(c) for c in codes_map.values() if parse_code_number(c) > 0}
    next_num = (max(used_numbers) + 1) if used_numbers else 1

    chronological_titles = sorted(
        dramas_dict.keys(),
        key=lambda t: dramas_dict[t].get("latest_id", 0),
    )

    missing_codes_to_save = []
    missing_dates_to_save = []
    today_ddmmyyyy = get_cambodia_date_ddmmyyyy()

    for title in chronological_titles:
        if title not in codes_map:
            while next_num in used_numbers:
                next_num += 1
            new_code = format_drama_code(next_num)
            used_numbers.add(next_num)
            codes_map[title] = new_code
            missing_codes_to_save.append((title, new_code))

        if title not in dates_map or not dates_map[title]:
            dates_map[title] = today_ddmmyyyy
            missing_dates_to_save.append((title, today_ddmmyyyy))

    if missing_codes_to_save or missing_dates_to_save:
        def _bg_persist():
            for t, c in missing_codes_to_save:
                save_drama_code_in_db(t, c)
            for t, d in missing_dates_to_save:
                save_drama_upload_date_in_db(t, d)
        threading.Thread(target=_bg_persist, daemon=True).start()


def determine_audio_type(drama_title: str, explicit_lang: Optional[str] = None) -> str:
    if explicit_lang in ["dub", "sub"]:
        return explicit_lang
    t_low = drama_title.lower()
    if any(k in t_low for k in ["sub", "សាប់", "subtitle", "អក្សរខ្មែរ"]):
        return "sub"
    return "dub"


def fetch_single_drama_metadata(drama_title: str) -> dict:
    synopsis = ""
    drama_code = ""
    upload_date = ""
    tags = ""
    duration_sec = 0
    has_gif = False
    gif_version = "v1"
    main_cover_version = "v1"
    album_items = []
    is_trending = False

    try:
        keys = [
            drama_title,
            f"{SYN_STORE_PREFIX}{drama_title}",
            f"{CODE_STORE_PREFIX}{drama_title}",
            f"{DATE_STORE_PREFIX}{drama_title}",
            f"{DUR_STORE_PREFIX}{drama_title}",
            f"{TAGS_STORE_PREFIX}{drama_title}",
            f"{GIF_STORE_PREFIX}{drama_title}",
            f"{ALBUM_STORE_PREFIX}{drama_title}__1",
            f"{ALBUM_STORE_PREFIX}{drama_title}__2",
            f"{ALBUM_STORE_PREFIX}{drama_title}__3",
            CFG_TRENDING_KEY,
        ]
        res = (
            supabase.table("drama_covers")
            .select("*")
            .in_("drama_title", keys)
            .execute()
        )
        album_slots_map = {}
        for r in (res.data or []):
            k = (r.get("drama_title") or "").strip()
            val = (r.get("cover_file_id") or "").strip()
            mid = r.get("cover_channel_msg_id") or ""
            if k == f"{SYN_STORE_PREFIX}{drama_title}":
                synopsis = val
            elif k == f"{CODE_STORE_PREFIX}{drama_title}" and val.upper().startswith("XF-"):
                drama_code = val.upper()
            elif k == f"{DATE_STORE_PREFIX}{drama_title}":
                upload_date = normalize_date_to_ddmmyyyy(val)
            elif k == f"{DUR_STORE_PREFIX}{drama_title}" and val.isdigit():
                duration_sec = int(val)
            elif k == f"{TAGS_STORE_PREFIX}{drama_title}":
                tags = val
            elif k == f"{GIF_STORE_PREFIX}{drama_title}" and (val or mid):
                has_gif = True
                gif_version = make_cover_version(val, mid)
            elif k.startswith(f"{ALBUM_STORE_PREFIX}{drama_title}__") and (val or mid):
                slot_str = k.rsplit("__", 1)[-1]
                if slot_str.isdigit():
                    slot_num = int(slot_str)
                    ver = make_cover_version(val, mid)
                    album_slots_map[slot_num] = {
                        "slot": slot_num,
                        "version": ver,
                    }
            elif k == CFG_TRENDING_KEY and val:
                try:
                    tr_list = json.loads(val)
                    if isinstance(tr_list, list) and drama_title in tr_list:
                        is_trending = True
                except Exception:
                    pass
            elif k == drama_title:
                main_cover_version = make_cover_version(val, mid)
                if not drama_code and r.get("drama_code"):
                    drama_code = str(r.get("drama_code")).strip().upper()
                if not upload_date and r.get("uploaded_at"):
                    upload_date = normalize_date_to_ddmmyyyy(r.get("uploaded_at"))
                if not tags and r.get("tags"):
                    tags = str(r.get("tags")).strip()

        for s in sorted(album_slots_map.keys()):
            album_items.append(album_slots_map[s])
    except Exception:
        log.warning("Could not fetch single drama metadata")

    return {
        "synopsis": synopsis,
        "drama_code": drama_code,
        "upload_date": upload_date or get_cambodia_date_ddmmyyyy(),
        "duration_seconds": duration_sec,
        "duration_text": format_video_duration(duration_sec) if duration_sec > 0 else "",
        "tags": tags,
        "has_gif": has_gif,
        "gif_version": gif_version,
        "cover_version": main_cover_version,
        "album": album_items,
        "is_trending": is_trending,
    }


def fetch_episode_cover_versions_map(drama_title: str) -> dict[int, str]:
    ep_versions = {}
    prefix = f"{EP_COV_STORE_PREFIX}{drama_title}__"
    try:
        res = (
            supabase.table("drama_covers")
            .select("drama_title, cover_file_id, cover_channel_msg_id")
            .like("drama_title", f"{prefix}%")
            .execute()
        )
        for r in (res.data or []):
            k = (r.get("drama_title") or "").strip()
            if k.startswith(prefix):
                ep_str = k[len(prefix):].strip()
                if ep_str.isdigit():
                    fid = r.get("cover_file_id") or ""
                    mid = r.get("cover_channel_msg_id") or ""
                    if fid or mid:
                        ep_versions[int(ep_str)] = make_cover_version(fid, mid)
    except Exception:
        log.warning("Could not fetch episode cover versions for %s", drama_title)
    return ep_versions


def fetch_drama_rating_stats(drama_title: str, user_id: Optional[int] = None) -> tuple[float, int, int]:
    rate_key = f"{RATE_STORE_PREFIX}{drama_title}"
    try:
        res = (
            supabase.table("user_history")
            .select("user_id, last_ep")
            .eq("drama_title", rate_key)
            .execute()
        )
        rows = res.data or []
        valid_scores = []
        user_score = 0
        for r in rows:
            s = int(r.get("last_ep") or 0)
            if 1 <= s <= 5:
                valid_scores.append(s)
                if user_id and int(r.get("user_id") or 0) == int(user_id):
                    user_score = s
        if not valid_scores:
            return 0.0, 0, user_score
        avg = round(sum(valid_scores) / len(valid_scores), 1)
        return avg, len(valid_scores), user_score
    except Exception:
        log.warning("Could not fetch rating stats for drama: %s", drama_title)
        return 0.0, 0, 0


def fetch_drama_views_map() -> dict:
    views_map = {}
    try:
        res = supabase.table("user_history").select("drama_title").execute()
        for r in (res.data or []):
            t = (r.get("drama_title") or "").strip()
            if t and not is_internal_key(t):
                views_map[t] = views_map.get(t, 0) + 1
    except Exception:
        log.warning("Could not fetch drama views map")
    return views_map


def check_user_favorite(user_id: int, drama_title: str) -> bool:
    try:
        fav_key = f"{FAV_STORE_PREFIX}{drama_title}"
        res = (
            supabase.table("user_history")
            .select("drama_title")
            .eq("user_id", user_id)
            .eq("drama_title", fav_key)
            .limit(1)
            .execute()
        )
        return bool(res.data)
    except Exception:
        return False


def build_mini_app_url(user_id=None):
    base = (
        MINI_APP_URL
        or os.getenv("RENDER_EXTERNAL_URL", "").strip()
        or os.getenv("PUBLIC_BASE_URL", "").strip()
    ).rstrip("/")
    if not base:
        return ""
    if not base.startswith(("http://", "https://")):
        base = "https://" + base
    sep = "&" if "?" in base else "?"
    if user_id:
        return f"{base}{sep}user_id={user_id}&v=17"
    return f"{base}{sep}v=17"


def ensure_user_state(user_id):
    if user_id not in user_current_drama or not user_current_drama.get(user_id):
        try:
            res = (
                supabase.table("user_history")
                .select("drama_title, last_ep")
                .eq("user_id", user_id)
                .order("updated_at", desc=True)
                .limit(20)
                .execute()
            )
            for row in (res.data or []):
                t = row.get("drama_title") or ""
                if t and not is_internal_key(t):
                    user_current_drama[user_id] = t
                    user_current_ep[user_id] = int(row.get("last_ep") or 1)
                    user_drama_multi_ep[user_id] = check_drama_has_multiple_episodes(t)
                    break
        except Exception:
            log.warning("Could not restore user state from user_history")

    return user_current_drama.get(user_id), user_current_ep.get(user_id)


def should_show_episode_buttons(user_id: int) -> bool:
    if not user_id:
        return False
    drama = user_current_drama.get(user_id)
    if not drama:
        drama, _ = ensure_user_state(user_id)
    if not drama:
        return False
    if user_id not in user_drama_multi_ep:
        user_drama_multi_ep[user_id] = check_drama_has_multiple_episodes(drama)
    return bool(user_drama_multi_ep.get(user_id, False))


def get_bottom_menu(user_id=None, has_active_drama: Optional[bool] = None):
    if has_active_drama is None:
        has_active_drama = should_show_episode_buttons(user_id) if user_id else False

    markup = ReplyKeyboardMarkup(
        resize_keyboard=True,
        is_persistent=True,
        one_time_keyboard=False,
        row_width=2,
    )
    webapp_url = build_mini_app_url(user_id)

    if webapp_url.startswith("https://"):
        mini_app_btn = KeyboardButton(
            BTN_MINI_APP,
            web_app=WebAppInfo(url=webapp_url),
        )
    else:
        mini_app_btn = KeyboardButton(BTN_MINI_APP)

    if has_active_drama:
        markup.row(
            KeyboardButton(BTN_PREV_EP),
            KeyboardButton(BTN_NEXT_EP),
        )
        markup.row(
            KeyboardButton(BTN_ALL_EPS),
            mini_app_btn,
        )
    else:
        markup.row(mini_app_btn)

    markup.row(
        KeyboardButton(BTN_REFRESH),
        KeyboardButton(BTN_CLEAR),
    )
    return markup


def send_temp_notice(user_id, text, delay=2.5):
    def _worker():
        try:
            sent = bot.send_message(
                user_id,
                text,
                reply_markup=get_bottom_menu(user_id),
            )

            track_trash_message(user_id, sent.message_id)

            time.sleep(delay)

            bot.delete_message(
                user_id,
                sent.message_id,
            )

            if user_id in user_trash_messages:
                user_trash_messages[user_id].discard(sent.message_id)

            main_msg = bot.send_message(
                user_id,
                "សូមជ្រើសរើសម៉ឺនុយខាងក្រោម",
                reply_markup=get_bottom_menu(user_id),
            )

            old_main_id = user_main_menu_id.get(user_id)
            user_main_menu_id[user_id] = main_msg.message_id

            if old_main_id and old_main_id != main_msg.message_id:
                safe_delete_one(user_id, old_main_id)

        except Exception:
            log.exception("Temporary notice failed")

    threading.Thread(
        target=_worker,
        daemon=True,
    ).start()


def get_episodes_list_keyboard(page, total_eps):
    markup = InlineKeyboardMarkup()
    per_page = 6

    page = max(0, page)
    start_ep = page * per_page + 1
    end_ep = min((page + 1) * per_page, total_eps)

    buttons = [
        InlineKeyboardButton(
            f"ភាគ {ep}",
            callback_data=f"watch_ep_{ep}",
        )
        for ep in range(start_ep, end_ep + 1)
    ]

    for i in range(0, len(buttons), 3):
        markup.row(*buttons[i : i + 3])

    nav = []

    if page > 0:
        nav.append(
            InlineKeyboardButton(
                "◀️ មុន",
                callback_data=f"show_list_page_{page - 1}",
            )
        )

    if end_ep < total_eps:
        nav.append(
            InlineKeyboardButton(
                "បន្ទាប់ ▶️",
                callback_data=f"show_list_page_{page + 1}",
            )
        )

    if nav:
        markup.row(*nav)

    markup.add(
        InlineKeyboardButton(
            "🔙 បិទបញ្ជីភាគ",
            callback_data="close_list_view",
        )
    )

    return markup


def save_user_history(user_id, drama_title, ep_number):
    try:
        supabase.table("user_history").upsert(
            {
                "user_id": user_id,
                "drama_title": drama_title,
                "last_ep": ep_number,
            },
            on_conflict="user_id,drama_title",
        ).execute()
    except Exception:
        log.exception("Cannot save history; check user_history table/schema")


def clear_previous_message(user_id):
    old_id = user_last_message.get(user_id)
    if old_id:
        try:
            bot.delete_message(user_id, old_id)
        except Exception:
            pass
        if user_id in user_trash_messages:
            user_trash_messages[user_id].discard(old_id)
        user_last_message.pop(user_id, None)


def _build_strict_channel_id_candidates(raw_id: int, raw_link: str) -> tuple[list, Optional[str]]:
    """
    Returns (numeric_candidates, public_username_fallback).
    Ensures -100 prefix is always tested for Telegram Channels/Supergroups.
    """
    num_candidates = []
    if raw_id:
        cid = int(raw_id)
        abs_str = str(abs(cid))
        if abs_str.startswith("100") and len(abs_str) > 5:
            super_id = -int(abs_str)
            if super_id not in num_candidates:
                num_candidates.append(super_id)
        else:
            super_id = int(f"-100{abs_str}")
            if super_id not in num_candidates:
                num_candidates.append(super_id)
        if cid not in num_candidates:
            num_candidates.append(cid)
        if -abs(cid) not in num_candidates:
            num_candidates.append(-abs(cid))

    pub_username = None
    clean_link = (raw_link or "").strip()
    m_user = re.match(r"^(?:https?://)?(?:t\.me|telegram\.me)/([A-Za-z0-9_]{4,64})/?$", clean_link)
    if m_user:
        uname = m_user.group(1)
        if uname.lower() not in {"joinchat", "addlist", "share", "c", "s"}:
            pub_username = f"@{uname}"

    return num_candidates, pub_username


def is_required_channel_member(user_id: int) -> bool:
    """
    STRICT FAIL-CLOSED membership check:
    Users who have NOT joined the required Channel/Group are NEVER allowed to watch videos.
    Checks live Telegram `get_chat_member` every time (no cached bypasses).
    """
    if not user_id:
        return False
    uid = int(user_id)
    req_channel_id, req_channel_link = get_required_channel_config(force_refresh=True)
    num_candidates, pub_username = _build_strict_channel_id_candidates(req_channel_id, req_channel_link)

    clients = [bot]
    if ADMIN_BOT_TOKEN_FALLBACK and ADMIN_BOT_TOKEN_FALLBACK != BOT_TOKEN:
        try:
            clients.append(telebot.TeleBot(ADMIN_BOT_TOKEN_FALLBACK, threaded=False))
        except Exception:
            pass

    chat_found = False
    for client in clients:
        for target_chat in num_candidates:
            try:
                member = client.get_chat_member(target_chat, uid)
                chat_found = True
                status = str(getattr(member, "status", "") or "").lower()
                if status in {"member", "administrator", "creator", "owner"}:
                    return True
                if status == "restricted" and bool(getattr(member, "is_member", False)):
                    return True
                if status in {"left", "kicked"}:
                    if (int(target_chat), uid) in PENDING_JOIN_REQUEST_USERS or (int(req_channel_id), uid) in PENDING_JOIN_REQUEST_USERS:
                        return True
                    return False
            except Exception as e:
                err_low = str(e).lower()
                if "user not found" in err_low or "participant_id_invalid" in err_low:
                    return False
                continue

        # Only if the numeric channel ID was not found by this bot AND the link is a public @username link
        if not chat_found and pub_username:
            try:
                member = client.get_chat_member(pub_username, uid)
                chat_found = True
                status = str(getattr(member, "status", "") or "").lower()
                if status in {"member", "administrator", "creator", "owner"}:
                    return True
                if status == "restricted" and bool(getattr(member, "is_member", False)):
                    return True
                if status in {"left", "kicked"}:
                    return False
            except Exception:
                continue

    if not chat_found:
        log.warning(
            "STRICT FORCE-JOIN: Could not verify membership in channel %s (%s) for user %s — blocking video.",
            req_channel_id,
            req_channel_link,
            uid,
        )
    return False


def build_bot_chat_join_channel_markup(watch_token: Optional[str] = None) -> InlineKeyboardMarkup:
    """Show only the required Channel join button until membership is confirmed."""
    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton(
            f"📢 ចូល Channel {CHANNEL_DISPLAY_NAME}",
            url=get_required_channel_link(),
        )
    )
    return markup


def build_bot_chat_watch_markup(watch_token: str) -> InlineKeyboardMarkup:
    """Reveal the Watch button only after Telegram confirms the user joined."""
    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton(
            "▶️ មើលឥឡូវនេះ",
            callback_data=f"notif_watch_{watch_token}",
        )
    )
    return markup


# Channel join prompt in bot chat removed; join check prompt is displayed inside Mini App only.


def send_bot_chat_join_prompt(user_id: int, drama_title: str, ep_number: int = 1):
    """Sends a strict Join Channel/Group prompt in Bot Chat when a non-member tries to watch in Bot Chat."""
    uid = int(user_id)
    PENDING_WATCH_AFTER_JOIN[uid] = {
        "drama_title": drama_title,
        "ep_number": int(ep_number or 1),
        "notif_msg_id": None,
    }
    watch_token = create_notification_watch_token(drama_title, int(ep_number or 1))
    markup = build_bot_chat_join_channel_markup()
    if watch_token:
        PENDING_WATCH_AFTER_JOIN[uid]["watch_token"] = watch_token
    try:
        sent_prompt = bot.send_message(
            chat_id=uid,
            text=(
                f"🔒 <b>សូមចូលក្នុង Channel ឬ Group ជាមុនសិន!</b>\n\n"
                f"ដើម្បីទស្សនាវីដេអូក្នុង <b>{CHANNEL_DISPLAY_NAME}</b> សូមចុចប៊ូតុងខាងក្រោមដើម្បីចូលជាសមាជិកជាមុនសិន។ "
                f"អ្នកមិនទាន់ចូល មិនអាចមើលវីដេអូបានឡើយ!"
            ),
            parse_mode="HTML",
            reply_markup=markup,
        )
        PENDING_WATCH_AFTER_JOIN[uid]["notif_msg_id"] = int(sent_prompt.message_id)
        track_trash_message(uid, sent_prompt.message_id)
    except Exception:
        log.warning("Could not send join prompt to user %s", uid)


def send_drama_episode(user_id, drama_title, ep_number, success_notice_message_id: Optional[int] = None):
    uid = int(user_id)
    if not is_required_channel_member(uid):
        log.info("Blocked video delivery for non-member user %s", uid)
        send_bot_chat_join_prompt(uid, drama_title, int(ep_number or 1))
        return
    record_user_start(uid)
    try:
        all_eps = get_episode_rows(drama_title)
    except Exception:
        log.exception("Supabase episode query failed")
        err_msg = bot.send_message(
            user_id,
            "❌ មិនអាចទាញទិន្នន័យរឿងបានទេ។ សូមពិនិត្យ Supabase និងឈ្មោះ columns។",
            reply_markup=get_bottom_menu(user_id),
        )
        track_trash_message(user_id, err_msg.message_id)
        return

    target_row = next(
        (r for r in all_eps if int(r.get("ep_number") or 0) == int(ep_number)),
        None,
    )

    if not target_row:
        send_temp_notice(
            user_id,
            "នេះជាភាគចុងក្រោយហើយ។",
            delay=2.5,
        )
        return

    channel_msg_id = target_row.get("channel_msg_id")

    if not channel_msg_id:
        send_temp_notice(user_id, "❌ ភាគនេះមិនមានវីដេអូទេ។", delay=2.5)
        return

    is_multi_ep = len(all_eps) > 1

    user_current_drama[user_id] = drama_title
    user_current_ep[user_id] = ep_number
    user_drama_multi_ep[user_id] = is_multi_ep

    save_user_history(user_id, drama_title, ep_number)

    # Remove the previous video and cover so the chat is left with the video only.
    clear_previous_message(user_id)
    clear_user_cover_message(user_id)

    single_meta = fetch_single_drama_metadata(drama_title)
    drama_code = (single_meta.get("drama_code") or "").strip()
    if not drama_code:
        try:
            meta_maps = fetch_all_drama_metadata_maps()
            drama_code = (meta_maps.get("codes", {}).get(drama_title) or "").strip()
        except Exception:
            drama_code = ""

    ep_suffix = f" - ភាគ {ep_number}" if is_multi_ep else ""
    caption_lines = [f"🎬 <b>{html.escape(drama_title)}{ep_suffix}</b>"]
    if drama_code:
        caption_lines.append(f"🆔 លេខកូដ៖ <code>{html.escape(drama_code)}</code>")
    video_caption = "\n".join(caption_lines)

    try:
        sent = bot.copy_message(
            chat_id=user_id,
            from_chat_id=CHANNEL_ID,
            message_id=int(channel_msg_id),
            caption=video_caption,
            parse_mode="HTML",
            protect_content=True,
            reply_markup=get_bottom_menu(user_id, has_active_drama=is_multi_ep),
        )
        user_last_message[user_id] = sent.message_id
        track_trash_message(user_id, sent.message_id)

        # Delete the join-success notice only after Telegram confirms the video was sent.
        if success_notice_message_id:
            try:
                bot.delete_message(chat_id=uid, message_id=int(success_notice_message_id))
            except Exception:
                log.warning("Could not delete join-success notice %s for user %s", success_notice_message_id, uid)

    except Exception:
        log.exception("Telegram copy_message failed")
        err_msg = bot.send_message(
            user_id,
            "❌ មិនអាចផ្ញើវីដេអូបានទេ។ ពិនិត្យ Bot permission និង CHANNEL_ID។",
            reply_markup=get_bottom_menu(user_id, has_active_drama=is_multi_ep),
        )
        track_trash_message(user_id, err_msg.message_id)


def extract_target_user_id_from_reply(reply_msg) -> Optional[int]:
    if not reply_msg:
        return None

    mapped_uid = get_mapped_user_from_admin_msg(reply_msg.chat.id, reply_msg.message_id)
    if mapped_uid:
        return int(mapped_uid)

    raw_text = (reply_msg.text or reply_msg.caption or "")
    match = re.search(r"#UID(\d+)", raw_text)
    if match:
        return int(match.group(1))

    match_alt = re.search(r"User ID៖\s*(?:#UID)?(\d+)", raw_text)
    if match_alt:
        return int(match_alt.group(1))

    return None


def extract_original_request_context_from_reply(reply_msg) -> tuple[str, str]:
    if not reply_msg:
        return "", ""

    mapped = get_mapped_info_from_admin_msg(reply_msg.chat.id, reply_msg.message_id)
    drama_title = (mapped.get("drama_title") or "").strip() if mapped else ""
    req_text = (mapped.get("request_text") or "").strip() if mapped else ""

    raw_text = (reply_msg.text or reply_msg.caption or "")

    if not drama_title:
        m_drama = re.search(r"🎬\s*ពីទំព័ររឿង៖\s*(.+)", raw_text)
        if m_drama:
            drama_title = m_drama.group(1).strip()

    if not req_text:
        m_req = re.search(
            r"💬\s*សារស្នើសុំ៖\s*\n([\s\S]+?)(?:\n\n💡|\Z)",
            raw_text,
        )
        if m_req:
            req_text = m_req.group(1).strip()

    return drama_title, req_text


def build_user_request_blockquote_html(drama_title: str, req_text: str) -> str:
    lines = []
    if drama_title:
        lines.append(f"🎬 <b>រឿង៖</b> {html.escape(drama_title)}")
    if req_text:
        lines.append(f"💬 <b>សំណើរបស់អ្នក៖</b> {html.escape(req_text)}")
    if not lines:
        return ""
    return "<blockquote>" + "\n".join(lines) + "</blockquote>\n\n"


def build_candidate_admin_targets() -> list[int]:
    candidates = []

    def _add(val):
        if val is None:
            return
        try:
            v = int(val)
            if v != 0 and v != CHANNEL_ID and v not in candidates:
                candidates.append(v)
        except Exception:
            pass

    saved_grp = load_saved_admin_group_id()
    _add(saved_grp)
    _add(ADMIN_GROUP_ID)

    for raw_id in [ADMIN_GROUP_ID, saved_grp, ADMIN_ID]:
        if raw_id and int(raw_id) > 0:
            pos = int(raw_id)
            _add(pos)
            _add(-pos)
            if not str(pos).startswith("100"):
                _add(int(f"-100{pos}"))

    for aid in sorted(list(dynamic_admin_ids)):
        _add(aid)
    _add(ADMIN_ID)
    return candidates


# --------------------------------------------------
# Telegram commands & handlers
# --------------------------------------------------

def complete_pending_watch_after_join(user_id: int, pending: Optional[dict], source: str = "membership_update") -> None:
    """Notify the user that joining succeeded, then deliver the pending episode exactly once."""
    if not pending:
        return
    uid = int(user_id)
    drama_title = str(pending.get("drama_title") or "").strip()
    ep_number = int(pending.get("ep_number") or 1)
    if not drama_title:
        return

    # Remove the old join/notification prompt so the success message and video are clear.
    old_msg_id = pending.get("notif_msg_id")
    if old_msg_id:
        try:
            bot.delete_message(chat_id=uid, message_id=int(old_msg_id))
        except Exception:
            pass

    success_notice_message_id = None
    try:
        success_notice = bot.send_message(
            uid,
            "✅ <b>ចូល Channel/Group បានជោគជ័យ!</b>\n\n🎬 កំពុងផ្ញើវីដេអូជូនអ្នក...",
            parse_mode="HTML",
        )
        success_notice_message_id = success_notice.message_id
    except Exception:
        log.exception("Could not send join success message to user %s", uid)

    log.info("Join completed for user %s via %s; delivering %s episode %s", uid, source, drama_title, ep_number)
    send_drama_episode(
        uid,
        drama_title,
        ep_number,
        success_notice_message_id=success_notice_message_id,
    )


@bot.chat_join_request_handler()
def handle_user_chat_join_request(join_req):
    """Approve the required Channel/Group join request, then notify and send the pending episode."""
    try:
        uid = int(join_req.from_user.id)
        cid = int(join_req.chat.id)
        required_id, required_link = get_required_channel_config(force_refresh=True)
        allowed_channel_ids, _ = _build_strict_channel_id_candidates(required_id, required_link)
        if cid not in {int(x) for x in allowed_channel_ids}:
            return
        PENDING_JOIN_REQUEST_USERS.add((cid, uid))
        try:
            bot.approve_chat_join_request(cid, uid)
        except Exception:
            log.exception("Could not approve channel/group join request for user %s", uid)
            return
        pending = PENDING_WATCH_AFTER_JOIN.pop(uid, None)
        complete_pending_watch_after_join(uid, pending, "join_request")
    except Exception:
        log.exception("Error in handle_user_chat_join_request")


@bot.chat_member_handler()
def handle_user_chat_member_update(update):
    """When a user joins the required Channel/Group, notify them and send their pending episode."""
    try:
        new_member = getattr(update, "new_chat_member", None)
        if not new_member or not getattr(new_member, "user", None):
            return
        uid = int(new_member.user.id)
        cid = int(update.chat.id) if getattr(update, "chat", None) else 0
        required_id, required_link = get_required_channel_config(force_refresh=True)
        allowed_channel_ids, _ = _build_strict_channel_id_candidates(required_id, required_link)
        if cid not in {int(x) for x in allowed_channel_ids}:
            return
        status = str(getattr(new_member, "status", "") or "").lower()
        if status in {"member", "administrator", "creator", "owner"} or (
            status == "restricted" and bool(getattr(new_member, "is_member", False))
        ):
            pending = PENDING_WATCH_AFTER_JOIN.pop(uid, None)
            complete_pending_watch_after_join(uid, pending, "chat_member")
        elif status in {"left", "kicked"}:
            PENDING_JOIN_REQUEST_USERS.discard((cid, uid))
    except Exception:
        log.exception("Error in handle_user_chat_member_update")


@bot.my_chat_member_handler()
def handle_bot_added_to_group(update):
    try:
        chat = update.chat
        if chat and chat.type in ["group", "supergroup"] and int(chat.id) != int(CHANNEL_ID):
            save_detected_admin_group_id(chat.id)
            log.info("Auto-registered Admin Group ID: %s", chat.id)
    except Exception:
        pass


@bot.message_handler(commands=["id", "chatid"])
def show_chat_id(message):
    if message.chat.type in ["group", "supergroup"] and int(message.chat.id) != int(CHANNEL_ID):
        save_detected_admin_group_id(message.chat.id)
    bot.reply_to(
        message,
        f"✅ <b>បានភ្ជាប់គ្រុបនេះជា Admin Group រួចរាល់!</b>\n"
        f"🆔 <b>Chat ID៖</b> <code>{message.chat.id}</code>\n"
        f"👤 <b>Your ID៖</b> <code>{message.from_user.id if message.from_user else ''}</code>",
        parse_mode="HTML",
    )


@bot.message_handler(commands=["stats"])
def show_admin_stats_command(message):
    if not message.from_user or not is_admin_user(message.from_user.id):
        if message.chat.type == "private":
            try:
                bot.delete_message(message.chat.id, message.message_id)
            except Exception:
                pass
        return

    stats = compute_user_analytics_payload()
    ns = stats["notif_stats"]
    text = (
        "📊 <b>ស្ថិតិអ្នកប្រើប្រាស់ និងប្រព័ន្ធ (Admin Only)</b>\n"
        "━━━━━━━━━━━━━━━━━━\n"
        f"👥 <b>អ្នកប្រើសរុប (ធ្លាប់ចុច /start)៖</b> {stats['total_users']} នាក់\n"
        f"🆕 <b>អ្នកប្រើថ្មីថ្ងៃនេះ៖</b> {stats['new_users_today']} នាក់\n"
        f"📅 <b>អ្នកប្រើថ្មី (៧ ថ្ងៃចុងក្រោយ)៖</b> {stats['new_users_7d']} នាក់\n"
        f"🔥 <b>អ្នកប្រើសកម្មថ្ងៃនេះ៖</b> {stats['active_users_today']} នាក់\n"
        "━━━━━━━━━━━━━━━━━━\n"
        f"🎬 <b>រឿងសរុប៖</b> {stats['total_dramas']} រឿង ({stats['total_episodes']} ភាគ)\n"
        "━━━━━━━━━━━━━━━━━━\n"
        "🔔 <b>ស្ថិតិការផ្ញើ Notification៖</b>\n"
        f"• ចំនួនដងផ្ញើសរុប៖ {ns.get('total_broadcasts', 0)} ដង\n"
        f"• សារផ្ញើជោគជ័យសរុប៖ {ns.get('total_delivered', 0)} សារ\n"
        f"• រឿងចុងក្រោយ៖ {html.escape(str(ns.get('last_drama', 'មិនទាន់មាន')))}\n"
        f"• លទ្ធផលចុងក្រោយ៖ ជោគជ័យ {ns.get('last_sent_ok', 0)} / បរាជ័យ {ns.get('last_failed', 0)}"
    )
    sent = bot.send_message(message.chat.id, text, parse_mode="HTML")
    if message.chat.type == "private":
        track_trash_message(message.chat.id, sent.message_id)


@bot.message_handler(
    func=lambda m: (
        m.reply_to_message is not None
        and (
            m.chat.type in ["group", "supergroup"]
            or (m.from_user and is_admin_user(m.from_user.id))
        )
        and extract_target_user_id_from_reply(m.reply_to_message) is not None
    ),
    content_types=["text", "photo", "video", "document", "audio", "voice", "sticker"],
)
def handle_admin_reply_to_user_in_bot(message):
    if message.chat.type in ["group", "supergroup"] and int(message.chat.id) != int(CHANNEL_ID):
        save_detected_admin_group_id(message.chat.id)

    target_uid = extract_target_user_id_from_reply(message.reply_to_message)
    if not target_uid:
        return

    orig_drama, orig_req_text = extract_original_request_context_from_reply(
        message.reply_to_message
    )
    quote_html = build_user_request_blockquote_html(orig_drama, orig_req_text)

    try:
        if message.content_type == "text":
            reply_body = html.escape(message.text or "")
            msg_to_user = (
                "📩 <b>សារឆ្លើយតបពី Admin ៖</b>\n"
                "━━━━━━━━━━━━━━━━━━\n"
                f"{quote_html}"
                f"✍️ <b>ចម្លើយ Admin៖</b>\n{reply_body}"
            )
            sent_to_user = bot.send_message(
                chat_id=target_uid,
                text=msg_to_user,
                parse_mode="HTML",
                reply_markup=get_bottom_menu(target_uid),
            )
            track_trash_message(target_uid, sent_to_user.message_id)
        else:
            raw_cap = html.escape(message.caption or "")
            reply_part = f"✍️ <b>ចម្លើយ Admin៖</b>\n{raw_cap}" if raw_cap else ""
            cap_text = (
                "📩 <b>សារឆ្លើយតបពី Admin ៖</b>\n"
                "━━━━━━━━━━━━━━━━━━\n"
                f"{quote_html}"
                f"{reply_part}"
            ).strip()
            sent_to_user = bot.copy_message(
                chat_id=target_uid,
                from_chat_id=message.chat.id,
                message_id=message.message_id,
                caption=cap_text if message.content_type != "sticker" else None,
                parse_mode="HTML" if message.content_type != "sticker" else None,
                reply_markup=get_bottom_menu(target_uid),
            )
            track_trash_message(target_uid, sent_to_user.message_id)

        bot.reply_to(
            message,
            f"✅ បានបញ្ជូនសារឆ្លើយតបទៅកាន់ Chat Bot របស់អ្នកប្រើប្រាស់ (<code>#UID{target_uid}</code>) រួចរាល់!",
            parse_mode="HTML",
        )
    except Exception as exc:
        log.exception("Failed to send admin reply to user %s", target_uid)
        bot.reply_to(
            message,
            f"❌ មិនអាចផ្ញើទៅកាន់ User (<code>#UID{target_uid}</code>) បានទេ៖ {exc}",
            parse_mode="HTML",
        )


@bot.message_handler(commands=["start"])
@bot.message_handler(
    func=lambda message: message.text in [BTN_REFRESH, "↺ refresh", "🔄 refresh"]
)
def start_or_refresh_bot(message):
    if message.chat.type != "private":
        save_detected_admin_group_id(message.chat.id)
        return

    chat_id = message.chat.id
    msg_id = message.message_id

    record_user_start(chat_id)

    try:
        main_msg = bot.send_message(
            chat_id,
            "សូមជ្រើសរើសម៉ឺនុយខាងក្រោម",
            reply_markup=get_bottom_menu(
                chat_id,
                has_active_drama=False
            ),
        )

        new_main_id = main_msg.message_id

        sweep_and_clean_chat(
            chat_id,
            up_to_msg_id=msg_id,
            keep_msg_id=new_main_id
        )

        user_main_menu_id[chat_id] = new_main_id

    except Exception:
        log.exception("Start / Refresh command failed")

@bot.message_handler(
    func=lambda message: message.text in [BTN_CLEAR, "សម្អាត🗑️", "សម្អាត 🗑️"]
)
def handle_clear_chat(message):
    if message.chat.type != "private":
        return

    chat_id = message.chat.id
    msg_id = message.message_id

    try:
        user_current_drama.pop(chat_id, None)
        user_current_ep.pop(chat_id, None)
        user_drama_multi_ep.pop(chat_id, None)

        main_msg = bot.send_message(
            chat_id,
            "សូមជ្រើសរើសម៉ឺនុយខាងក្រោម",
            reply_markup=get_bottom_menu(chat_id, has_active_drama=False),
        )
        new_main_id = main_msg.message_id

        sweep_and_clean_chat(chat_id, up_to_msg_id=msg_id, keep_msg_id=new_main_id)
        user_main_menu_id[chat_id] = new_main_id
    except Exception:
        log.exception("Failed to clear chat")


@bot.message_handler(
    commands=["upload"],
    content_types=["video", "document"],
)
def admin_upload_video(message):
    if not message.from_user or not is_admin_user(message.from_user.id):
        bot.reply_to(message, "⛔ អ្នកមិនមានសិទ្ធិ Upload ទេ។")
        return

    try:
        parts = (message.caption or "").strip().split(maxsplit=3)

        if len(parts) < 3 or parts[0].lower() != "/upload":
            bot.reply_to(
                message,
                "ទម្រង់៖ /upload Drama_Title 1 https://cover-url.jpg\n"
                "ប្រើ _ ជំនួសដកឃ្លាក្នុងឈ្មោះរឿង។",
            )
            return

        drama_title = parts[1].replace("_", " ").strip()
        ep_number = int(parts[2])
        cover_url = parts[3].strip() if len(parts) > 3 else ""

        video_duration = 0
        if message.content_type == "video" and getattr(message, "video", None):
            video_duration = int(message.video.duration or 0)

        sent_to_channel = bot.copy_message(
            chat_id=CHANNEL_ID,
            from_chat_id=message.chat.id,
            message_id=message.message_id,
            caption=f"🎬 {drama_title} - ភាគ {ep_number}",
        )

        supabase.table("episodes").upsert(
            {
                "drama_title": drama_title,
                "ep_number": ep_number,
                "channel_msg_id": sent_to_channel.message_id,
                "cover_url": cover_url or None,
            },
            on_conflict="drama_title,ep_number",
        ).execute()

        if video_duration > 0:
            try:
                supabase.table("drama_covers").upsert(
                    {
                        "drama_title": f"{DUR_STORE_PREFIX}{drama_title}",
                        "cover_file_id": str(video_duration),
                        "cover_channel_msg_id": 0,
                    },
                    on_conflict="drama_title",
                ).execute()
            except Exception:
                pass

        save_drama_upload_date_in_db(drama_title, get_cambodia_date_ddmmyyyy())
        invalidate_dramas_cache()

        bot.reply_to(
            message,
            f"✅ Upload រឿង {drama_title} ភាគ {ep_number} ជោគជ័យ!",
        )

    except Exception as exc:
        log.exception("Admin upload failed")
        bot.reply_to(
            message,
            f"❌ Upload បរាជ័យ៖ {exc}\n"
            "ពិនិត្យ table episodes និង unique constraint (drama_title, ep_number)។",
        )


@bot.message_handler(
    func=lambda message: message.text in [
        BTN_PREV_EP, "⮜ ភាគមុន", "⏮ ភាគមុន", "ភាគមុន",
        BTN_NEXT_EP, "ភាគបន្ត ⮞", "ភាគបន្ត ⏭", "ភាគបន្ត",
    ]
)
def handle_prev_next(message):
    if message.chat.type != "private":
        return

    user_id = message.chat.id
    try:
        bot.delete_message(user_id, message.message_id)
    except Exception:
        track_trash_message(user_id, message.message_id)

    title, current_ep = ensure_user_state(user_id)

    if not title or not current_ep:
        send_temp_notice(user_id, "សូមជ្រើសរើសរឿងពីកម្មវិធី Mini App ជាមុនសិន។", delay=2.5)
        return

    if not should_show_episode_buttons(user_id):
        send_temp_notice(user_id, "រឿងនេះជារឿងមួយភាគចប់។", delay=2.0)
        return

    if message.text in [BTN_PREV_EP, "⮜ ភាគមុន", "⏮ ភាគមុន", "ភាគមុន"]:
        target_ep = current_ep - 1
        if target_ep < 1:
            send_temp_notice(user_id, "នេះជាភាគដំបូងហើយ!", delay=2.0)
            return
    else:
        target_ep = current_ep + 1

    send_drama_episode(user_id, title, target_ep)


@bot.message_handler(
    func=lambda message: message.text in [
        BTN_ALL_EPS,
        "🗁 បញ្ជីភាគ",
        "បញ្ជីភាគ",
        "🗁 បញ្ជីភាគទាំងអស់",
        "📑 បញ្ជីភាគទាំងអស់",
    ]
)
def handle_all_eps_bottom(message):
    if message.chat.type != "private":
        return

    user_id = message.chat.id
    try:
        bot.delete_message(user_id, message.message_id)
    except Exception:
        track_trash_message(user_id, message.message_id)

    title, current_ep = ensure_user_state(user_id)

    if not title:
        send_temp_notice(
            user_id,
            f"⚠️ សូមចុចប៊ូតុង «{BTN_MINI_APP}» ដើម្បីជ្រើសរើសរឿងជាមុនសិន។",
            delay=2.5,
        )
        return

    total = len(get_episode_rows(title))
    if total == 0:
        send_temp_notice(user_id, "មិនទាន់មានភាគសម្រាប់រឿងនេះទេ។", delay=2.5)
        return

    if total <= 1:
        send_temp_notice(user_id, "រឿងនេះជារឿងមួយភាគចប់។", delay=2.0)
        return

    ep_num = current_ep or 1
    per_page = 6
    initial_page = max(0, (ep_num - 1) // per_page)

    sent_list = bot.send_message(
        user_id,
        f"បញ្ជីភាគ (អ្នកមើលដល់ភាគ {ep_num})",
        reply_markup=get_episodes_list_keyboard(initial_page, total),
    )
    track_trash_message(user_id, sent_list.message_id)


@bot.callback_query_handler(
    func=lambda call: bool(call.data) and call.data.startswith("notif_watch_")
)
def handle_notification_watch(call):
    """
    Point 3.3: When User clicks '▶️ មើលឥឡូវនេះ' in User Bot Chat and hasn't joined the Channel/Group yet,
    do NOT require opening the Mini App. Instead, display '📢 ចូល Channel χƒαη 18 קɭυѕ' + '▶️ មើលឥឡូវនេះ'
    directly on the message in the Bot Chat!
    """
    try:
        token = call.data.removeprefix("notif_watch_")
        target = get_notification_watch_target(token)
        if not target:
            bot.answer_callback_query(
                call.id,
                "❌ តំណមើលនេះមិនត្រឹមត្រូវ ឬផុតកំណត់។",
                show_alert=True,
            )
            return

        drama_title, ep_number = target
        user_id = int(call.from_user.id)
        if not is_required_channel_member(user_id):
            PENDING_WATCH_AFTER_JOIN[user_id] = {
                "drama_title": drama_title,
                "ep_number": int(ep_number),
                "notif_msg_id": call.message.message_id if call.message else None,
            }
            PENDING_WATCH_AFTER_JOIN[user_id]["watch_token"] = token
            join_markup = build_bot_chat_join_channel_markup()
            try:
                bot.edit_message_reply_markup(
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    reply_markup=join_markup,
                )
            except Exception:
                pass
            bot.answer_callback_query(
                call.id,
                f"⚠️ សូមចូល Channel {CHANNEL_DISPLAY_NAME} ជាមុនសិន។ បន្ទាប់ពីចូលរួច ប៊ូតុង «មើលឥឡូវនេះ» នឹងបង្ហាញដោយស្វ័យប្រវត្តិ។",
                show_alert=True,
            )
            return

        bot.answer_callback_query(call.id, "កំពុងផ្ញើវីដេអូ...")
        try:
            bot.delete_message(call.message.chat.id, call.message.message_id)
        except Exception:
            pass
        send_drama_episode(user_id, drama_title, ep_number)

    except Exception:
        log.exception("Notification watch callback failed")
        try:
            bot.answer_callback_query(
                call.id,
                "មានបញ្ហាក្នុងការផ្ញើវីដេអូ។ សូមសាកល្បងម្ដងទៀត។",
                show_alert=True,
            )
        except Exception:
            pass


@bot.callback_query_handler(
    func=lambda call: call.data.startswith("show_list_page_")
)
def handle_show_list(call):
    try:
        page = int(call.data.rsplit("_", 1)[1])
        user_id = call.message.chat.id
        title, _ = ensure_user_state(user_id)

        if not title:
            bot.answer_callback_query(call.id, "សូមចាប់ផ្តើមមើលរឿងមុន។")
            return

        total = len(get_episode_rows(title))

        bot.edit_message_reply_markup(
            user_id,
            call.message.message_id,
            reply_markup=get_episodes_list_keyboard(page, total),
        )
        bot.answer_callback_query(call.id)

    except Exception:
        log.exception("Episode list failed")
        bot.answer_callback_query(call.id, "មានបញ្ហាក្នុងការបង្ហាញបញ្ជីភាគ។")


@bot.callback_query_handler(
    func=lambda call: call.data == "close_list_view"
)
def handle_close_list(call):
    user_id = call.message.chat.id
    msg_id = call.message.message_id
    try:
        bot.delete_message(user_id, msg_id)
    except Exception:
        pass
    if user_id in user_trash_messages:
        user_trash_messages[user_id].discard(msg_id)
    bot.answer_callback_query(call.id)


@bot.callback_query_handler(
    func=lambda call: call.data.startswith("watch_ep_")
)
def handle_watch_ep(call):
    try:
        target_ep = int(call.data.rsplit("_", 1)[1])
        user_id = call.message.chat.id
        msg_id = call.message.message_id
        title, _ = ensure_user_state(user_id)

        if not title:
            bot.answer_callback_query(call.id, "សូមចាប់ផ្តើមមើលរឿងមុន។")
            return

        bot.answer_callback_query(call.id)

        try:
            bot.delete_message(user_id, msg_id)
        except Exception:
            pass
        if user_id in user_trash_messages:
            user_trash_messages[user_id].discard(msg_id)

        send_drama_episode(user_id, title, target_ep)

    except Exception:
        log.exception("Watch episode callback failed")
        bot.answer_callback_query(call.id, "មានបញ្ហាក្នុងការបើកភាគ។")


@bot.message_handler(content_types=["web_app_data"])
def handle_web_app_data(message):
    user_id = message.chat.id
    try:
        bot.delete_message(user_id, message.message_id)
    except Exception:
        track_trash_message(user_id, message.message_id)

    try:
        payload = json.loads(message.web_app_data.data or "{}")

        if (
            payload.get("action") == "select_drama"
            and payload.get("title")
        ):
            ep_num = int(payload.get("ep_number") or 1)
            send_drama_episode(
                user_id,
                str(payload["title"]),
                ep_num,
            )

    except Exception:
        log.exception("Mini App data handler failed")


@bot.message_handler(
    func=lambda message: True,
    content_types=[
        "text", "photo", "video", "document", "sticker", "audio"
    ],
)
def auto_delete_user_typing(message):
    if message.chat.type in ["group", "supergroup"]:
        if int(message.chat.id) != int(CHANNEL_ID):
            save_detected_admin_group_id(message.chat.id)
        return

    if message.chat.type != "private":
        return

    if message.from_user and is_admin_user(message.from_user.id) and message.content_type in ["video", "document"]:
        return

    user_id = message.chat.id
    try:
        bot.delete_message(
            chat_id=user_id,
            message_id=message.message_id,
        )
    except Exception:
        track_trash_message(user_id, message.message_id)


# --------------------------------------------------
# FastAPI application
# --------------------------------------------------

bot_thread = None
notif_queue_thread = None


def run_bot():
    log.info("Preparing Telegram bot polling...")
    try:
        bot.remove_webhook()
        time.sleep(0.5)
    except Exception:
        log.warning("Could not remove webhook before polling")

    while True:
        try:
            log.info("Telegram bot infinity_polling started")
            bot.infinity_polling(
                skip_pending=False,
                timeout=60,
                long_polling_timeout=30,
                allowed_updates=["message", "callback_query", "my_chat_member", "chat_member", "chat_join_request"],
            )
        except Exception:
            log.exception("Polling crashed, restarting in 5 seconds...")
            time.sleep(5)


def run_notif_queue_poller():
    while True:
        try:
            check_and_process_notif_queue()
        except Exception:
            pass
        time.sleep(8)


@asynccontextmanager
async def lifespan(app: FastAPI):
    global bot_thread, notif_queue_thread

    log.info("Starting XFANSTER Telegram Drama WebApp")
    try:
        supabase.table("user_history").delete().like("drama_title", "__JOINED_CH__%").execute()
    except Exception:
        pass
    load_saved_admin_group_id()
    get_required_channel_config(force_refresh=True)
    register_user_bot_config_in_db()
    sync_all_covers_for_bot_token_async()

    bot_thread = threading.Thread(
        target=run_bot,
        name="telegram-bot-polling",
        daemon=True,
    )
    bot_thread.start()

    notif_queue_thread = threading.Thread(
        target=run_notif_queue_poller,
        name="notif-queue-poller",
        daemon=True,
    )
    notif_queue_thread.start()

    yield

    log.info("Stopping XFANSTER Telegram Drama WebApp")
    try:
        bot.stop_polling()
    except Exception:
        log.exception("Error while stopping Telegram bot")


app = FastAPI(
    title="χƒαη 18 קɭυѕ Drama API",
    version="4.1.0",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-Cover-Fallback"],
)


class SelectDramaRequest(BaseModel):
    user_id: int
    title: str
    ep_number: int = 1


class FavoriteDramaRequest(BaseModel):
    user_id: int
    title: str
    is_favorite: bool = True


class RateDramaRequest(BaseModel):
    user_id: int = 0
    title: str
    stars: int


class UserFeedbackRequest(BaseModel):
    user_id: Optional[int] = None
    first_name: Optional[str] = ""
    last_name: Optional[str] = ""
    username: Optional[str] = ""
    drama_title: Optional[str] = ""
    message: str


class InternalBroadcastRequest(BaseModel):
    drama_title: str
    admin_chat_id: Optional[int] = None
    custom_cover_msg_id: Optional[int] = None
    custom_cover_file_id: Optional[str] = None


class InternalChannelConfigRequest(BaseModel):
    channel_id: Optional[int] = None
    channel_link: Optional[str] = None


class AdminToggleTrendingRequest(BaseModel):
    user_id: int
    title: str
    is_trending: bool


class AdminUploadEpCoverRequest(BaseModel):
    user_id: int
    title: str
    ep_number: int
    image_base64: str


class AdminUploadAlbumRequest(BaseModel):
    user_id: int
    title: str
    slot: int
    image_base64: str


class AdminDeleteAlbumRequest(BaseModel):
    user_id: int
    title: str
    slot: Optional[int] = None


def decode_base64_image_bytes(raw_b64: str) -> bytes:
    cleaned = str(raw_b64 or "").strip()
    if "," in cleaned and cleaned.startswith("data:"):
        cleaned = cleaned.split(",", 1)[1]
    img_bytes = base64.b64decode(cleaned)
    with Image.open(io.BytesIO(img_bytes)) as im:
        rgb = im.convert("RGB")
        out = io.BytesIO()
        rgb.save(out, format="JPEG", quality=88, optimize=True)
        return out.getvalue()


@app.api_route("/", methods=["GET", "HEAD"])
def home(request: Request):
    if request.method == "HEAD":
        return Response(status_code=200)

    index_file = Path(__file__).resolve().parent / "index.html"

    if not index_file.is_file():
        return JSONResponse(
            status_code=500,
            content={"error": "index.html not found"},
        )

    return FileResponse(
        index_file,
        media_type="text/html",
        headers={
            "Cache-Control": "no-store, no-cache, must-revalidate, max-age=0",
            "Pragma": "no-cache",
            "Expires": "0",
        },
    )


@app.api_route("/health", methods=["GET", "HEAD"])
def health():
    return {
        "status": "ok",
        "service": "χƒαη 18 קɭυѕ Drama API",
        "bot_thread_started": bool(
            bot_thread and bot_thread.is_alive()
        ),
    }


@app.post("/api/internal/broadcast-notification")
def trigger_user_bot_notification(req: InternalBroadcastRequest):
    clean_title = (req.drama_title or "").strip()
    if not clean_title:
        return JSONResponse(status_code=400, content={"detail": "Missing drama_title"})
    execute_user_bot_broadcast_async(
        drama_title=clean_title,
        admin_chat_id=req.admin_chat_id,
        custom_cover_msg_id=req.custom_cover_msg_id,
        custom_cover_file_id=req.custom_cover_file_id,
    )
    return {"status": "ok", "drama_title": clean_title}


@app.post("/api/internal/channel-config")
def update_required_channel_config_api(req: InternalChannelConfigRequest):
    cid, clink = save_required_channel_config_in_db(req.channel_id, req.channel_link)
    return {
        "status": "ok",
        "channel_id": cid,
        "channel_link": clink,
        "channel_name": CHANNEL_DISPLAY_NAME,
    }


@app.get("/api/admin/analytics")
def get_admin_analytics_api(user_id: Optional[int] = Query(default=None)):
    if not user_id or not is_admin_user(user_id):
        return JSONResponse(
            status_code=403,
            content={"is_admin": False, "detail": "Forbidden: Admin access only"},
        )

    stats = compute_user_analytics_payload()
    stats["is_admin"] = True
    return JSONResponse(
        content=stats,
        headers={"Cache-Control": "no-store, no-cache, must-revalidate, max-age=0"},
    )


@app.post("/api/admin/toggle-trending")
def admin_toggle_trending_api(req: AdminToggleTrendingRequest):
    if not is_admin_user(req.user_id):
        return JSONResponse(status_code=403, content={"detail": "Admin only"})

    clean_title = (req.title or "").strip()
    if not clean_title:
        return JSONResponse(status_code=400, content={"detail": "Invalid title"})

    try:
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id")
            .eq("drama_title", CFG_TRENDING_KEY)
            .limit(1)
            .execute()
        )
        trending_list = []
        if res.data and res.data[0].get("cover_file_id"):
            try:
                parsed = json.loads(res.data[0]["cover_file_id"])
                if isinstance(parsed, list):
                    trending_list = [str(x).strip() for x in parsed if str(x).strip()]
            except Exception:
                trending_list = []

        if req.is_trending:
            if clean_title not in trending_list:
                trending_list.insert(0, clean_title)
        else:
            trending_list = [t for t in trending_list if t != clean_title]

        serialized = json.dumps(trending_list, ensure_ascii=False)
        payload = {
            "drama_title": CFG_TRENDING_KEY,
            "cover_file_id": serialized,
            "cover_channel_msg_id": 0,
        }
        if res.data:
            supabase.table("drama_covers").update(payload).eq("drama_title", CFG_TRENDING_KEY).execute()
        else:
            supabase.table("drama_covers").insert(payload).execute()

        invalidate_dramas_cache()
        return {
            "status": "ok",
            "is_trending": clean_title in trending_list,
            "top_trending_titles": trending_list,
        }
    except Exception as exc:
        log.exception("Failed to toggle top trending")
        return JSONResponse(status_code=500, content={"detail": str(exc)})


@app.post("/api/admin/upload-ep-cover")
def admin_upload_episode_cover_api(req: AdminUploadEpCoverRequest):
    if not is_admin_user(req.user_id):
        return JSONResponse(status_code=403, content={"detail": "Admin only"})

    clean_title = (req.title or "").strip()
    ep_num = int(req.ep_number or 0)
    if not clean_title or ep_num < 1 or not req.image_base64:
        return JSONResponse(status_code=400, content={"detail": "Invalid parameters"})

    try:
        jpeg_bytes = decode_base64_image_bytes(req.image_base64)
        bio = io.BytesIO(jpeg_bytes)
        bio.name = f"ep_{ep_num}_cover.jpg"

        sent = bot.send_photo(
            chat_id=CHANNEL_ID,
            photo=bio,
            caption=f"🖼️ Cover ភាគទី {ep_num}៖ {clean_title}",
        )
        if not getattr(sent, "photo", None):
            return JSONResponse(status_code=500, content={"detail": "Failed to upload photo to Telegram Channel"})

        file_id = sent.photo[-1].file_id
        key = f"{EP_COV_STORE_PREFIX}{clean_title}__{ep_num}"
        res = (
            supabase.table("drama_covers")
            .select("drama_title, cover_channel_msg_id")
            .eq("drama_title", key)
            .limit(1)
            .execute()
        )
        payload = {
            "drama_title": key,
            "cover_file_id": file_id,
            "cover_channel_msg_id": sent.message_id,
        }
        if res.data:
            old_mid = res.data[0].get("cover_channel_msg_id")
            if old_mid and int(old_mid) != int(sent.message_id):
                safe_delete_one(CHANNEL_ID, int(old_mid))
            supabase.table("drama_covers").update(payload).eq("drama_title", key).execute()
        else:
            supabase.table("drama_covers").insert(payload).execute()

        ver = make_cover_version(file_id, sent.message_id)
        return {
            "status": "ok",
            "ep_number": ep_num,
            "cover_version": ver,
        }
    except Exception as exc:
        log.exception("Failed to upload episode cover via API")
        return JSONResponse(status_code=500, content={"detail": str(exc)})


@app.post("/api/admin/upload-album")
def admin_upload_album_photo_api(req: AdminUploadAlbumRequest):
    if not is_admin_user(req.user_id):
        return JSONResponse(status_code=403, content={"detail": "Admin only"})

    clean_title = (req.title or "").strip()
    slot = int(req.slot or 0)
    if not clean_title or slot not in (1, 2, 3) or not req.image_base64:
        return JSONResponse(status_code=400, content={"detail": "Slot must be 1, 2, or 3"})

    try:
        jpeg_bytes = decode_base64_image_bytes(req.image_base64)
        bio = io.BytesIO(jpeg_bytes)
        bio.name = f"album_{slot}.jpg"

        sent = bot.send_photo(
            chat_id=CHANNEL_ID,
            photo=bio,
            caption=f"📸 រូប Album ({slot}/3)៖ {clean_title}",
        )
        if not getattr(sent, "photo", None):
            return JSONResponse(status_code=500, content={"detail": "Failed to upload album photo"})

        file_id = sent.photo[-1].file_id
        key = f"{ALBUM_STORE_PREFIX}{clean_title}__{slot}"
        res = (
            supabase.table("drama_covers")
            .select("drama_title, cover_channel_msg_id")
            .eq("drama_title", key)
            .limit(1)
            .execute()
        )
        payload = {
            "drama_title": key,
            "cover_file_id": file_id,
            "cover_channel_msg_id": sent.message_id,
        }
        if res.data:
            old_mid = res.data[0].get("cover_channel_msg_id")
            if old_mid and int(old_mid) != int(sent.message_id):
                safe_delete_one(CHANNEL_ID, int(old_mid))
            supabase.table("drama_covers").update(payload).eq("drama_title", key).execute()
        else:
            supabase.table("drama_covers").insert(payload).execute()

        ver = make_cover_version(file_id, sent.message_id)
        return {
            "status": "ok",
            "slot": slot,
            "version": ver,
        }
    except Exception as exc:
        log.exception("Failed to upload album photo via API")
        return JSONResponse(status_code=500, content={"detail": str(exc)})


@app.post("/api/admin/delete-album")
def admin_delete_album_photo_api(req: AdminDeleteAlbumRequest):
    if not is_admin_user(req.user_id):
        return JSONResponse(status_code=403, content={"detail": "Admin only"})

    clean_title = (req.title or "").strip()
    if not clean_title:
        return JSONResponse(status_code=400, content={"detail": "Missing title"})

    slots = [int(req.slot)] if req.slot in (1, 2, 3) else [1, 2, 3]
    keys = [f"{ALBUM_STORE_PREFIX}{clean_title}__{s}" for s in slots]
    try:
        res = (
            supabase.table("drama_covers")
            .select("drama_title, cover_channel_msg_id")
            .in_("drama_title", keys)
            .execute()
        )
        for row in (res.data or []):
            mid = row.get("cover_channel_msg_id")
            if mid:
                safe_delete_one(CHANNEL_ID, int(mid))
        if res.data:
            supabase.table("drama_covers").delete().in_("drama_title", keys).execute()
        return {"status": "ok"}
    except Exception as exc:
        log.exception("Failed to delete album photo")
        return JSONResponse(status_code=500, content={"detail": str(exc)})


@app.get("/api/dramas")
def list_dramas(user_id: Optional[int] = Query(default=None)):
    if user_id:
        record_user_start(user_id)

    now = time.time()
    with dramas_cache_lock:
        if (
            DRAMAS_LIST_CACHE["payload"] is not None
            and (now - DRAMAS_LIST_CACHE["timestamp"]) < DRAMAS_CACHE_TTL
        ):
            cached_copy = dict(DRAMAS_LIST_CACHE["payload"])
            # Keep the newest-first order already stored in the cache.
            cached_copy["items"] = list(cached_copy.get("items") or [])
            cached_copy["is_admin"] = is_admin_user(user_id) if user_id else False
            cached_copy["required_channel_link"] = get_required_channel_link()
            cached_copy["channel_name"] = CHANNEL_DISPLAY_NAME
            return JSONResponse(
                content=cached_copy,
                headers={
                    "Cache-Control": "no-store, no-cache, must-revalidate, max-age=0",
                },
            )

    try:
        response = (
            supabase.table("episodes")
            .select("drama_title,cover_url,category,channel_msg_id")
            .order("channel_msg_id", desc=True)
            .execute()
        )

        meta = fetch_all_drama_metadata_maps()
        cover_versions = meta["versions"]
        synopses_map = meta["synopses"]
        langs_map = meta["langs"]
        codes_map = meta["codes"]
        upload_dates_map = meta["upload_dates"]
        durations_map = meta["durations"]
        tags_map = meta["tags"]
        gifs_map = meta["gifs"]
        extra_cats = meta["categories"]
        top_trending_titles = meta["top_trending_titles"]
        trending_set = set(top_trending_titles)

        views_map = fetch_drama_views_map()
        dramas = OrderedDict()
        categories_set = set(extra_cats)

        for row in response.data or []:
            title = (row.get("drama_title") or "").strip()
            if not title or is_internal_key(title):
                continue

            cat = (row.get("category") or "").strip()
            if cat:
                categories_set.add(cat)

            raw_cover_url = row.get("cover_url")
            ver = cover_versions.get(title, "v1")
            if raw_cover_url:
                ver = make_cover_version(raw_cover_url, ver)

            msg_id = int(row.get("channel_msg_id") or 0)
            audio_type = determine_audio_type(title, langs_map.get(title))

            if title not in dramas:
                dramas[title] = {
                    "title": title,
                    "cover_url": raw_cover_url,
                    "cover_version": ver,
                    "category": cat,
                    "audio_type": audio_type,
                    "views": views_map.get(title, 0),
                    "latest_id": msg_id,
                    "synopsis": synopses_map.get(title, ""),
                    "tags": tags_map.get(title, ""),
                    "is_trending": title in trending_set,
                }
            else:
                if msg_id > dramas[title]["latest_id"]:
                    dramas[title]["latest_id"] = msg_id
                if not dramas[title]["cover_url"] and raw_cover_url:
                    dramas[title]["cover_url"] = raw_cover_url
                    dramas[title]["cover_version"] = make_cover_version(raw_cover_url, ver)
                if not dramas[title]["category"] and cat:
                    dramas[title]["category"] = cat

        ensure_codes_and_dates_for_dramas(dramas, meta)

        for title, item in dramas.items():
            d_code = codes_map.get(title, "")
            u_date = upload_dates_map.get(title, get_cambodia_date_ddmmyyyy())
            dur_sec = durations_map.get(title, 0)
            gif_ver = gifs_map.get(title)
            item["drama_code"] = d_code
            item["upload_date"] = u_date
            item["duration_seconds"] = dur_sec
            item["duration_text"] = format_video_duration(dur_sec) if dur_sec > 0 else ""
            item["has_gif"] = bool(gif_ver)
            item["gif_version"] = gif_ver or "v1"
            item["search_tokens"] = build_smart_search_tokens(
                title=title,
                drama_code=d_code,
                category=item.get("category", ""),
                synopsis=item.get("synopsis", ""),
                tags=item.get("tags", ""),
            )

        sorted_items = sorted(
            dramas.values(),
            key=lambda x: x.get("latest_id", 0),
            reverse=True,
        )

        top_trending_items = [
            dramas[t] for t in top_trending_titles if t in dramas
        ]

        base_payload = {
            "items": sorted_items,
            "top_trending": top_trending_items,
            "top_trending_titles": top_trending_titles,
            "categories": sorted(list(categories_set)),
            "count": len(sorted_items),
            "is_admin": is_admin_user(user_id) if user_id else False,
            "required_channel_link": get_required_channel_link(),
            "channel_name": CHANNEL_DISPLAY_NAME,
        }

        with dramas_cache_lock:
            DRAMAS_LIST_CACHE["timestamp"] = time.time()
            DRAMAS_LIST_CACHE["payload"] = base_payload

        response_payload = dict(base_payload)
        # sorted_items is already ordered by latest_id descending.
        response_payload["items"] = list(sorted_items)

        return JSONResponse(
            content=response_payload,
            headers={
                "Cache-Control": "no-store, no-cache, must-revalidate, max-age=0",
            },
        )

    except Exception:
        log.exception("Drama list API failed")
        return JSONResponse(
            status_code=500,
            content={"detail": "Could not load dramas from Supabase"},
        )


@app.get("/api/dramas/{drama_title}/episodes")
def list_drama_episodes(
    drama_title: str,
    user_id: Optional[int] = Query(default=None),
):
    try:
        response = (
            supabase.table("episodes")
            .select("ep_number,cover_url,channel_msg_id")
            .eq("drama_title", drama_title)
            .order("ep_number")
            .execute()
        )

        last_ep = None
        is_fav = False
        can_request_today = True

        if user_id is not None:
            record_user_start(user_id)
            try:
                hist = (
                    supabase.table("user_history")
                    .select("last_ep")
                    .eq("user_id", user_id)
                    .eq("drama_title", drama_title)
                    .limit(1)
                    .execute()
                )
                if hist.data and hist.data[0].get("last_ep") is not None:
                    last_ep = int(hist.data[0]["last_ep"])
            except Exception:
                log.warning("Could not query user_history for detail page")

            is_fav = check_user_favorite(user_id, drama_title)
            can_request_today = not has_user_requested_today(user_id)

        single_meta = fetch_single_drama_metadata(drama_title)
        ep_cov_versions = fetch_episode_cover_versions_map(drama_title)
        avg_rating, rating_count, user_rating = fetch_drama_rating_stats(drama_title, user_id)
        views_count = 0
        likes_count = 0
        try:
            v_res = (
                supabase.table("user_history")
                .select("user_id", count="exact")
                .eq("drama_title", drama_title)
                .execute()
            )
            views_count = int(v_res.count if v_res.count is not None else len(v_res.data or []))
        except Exception:
            pass
        try:
            l_res = (
                supabase.table("user_history")
                .select("user_id", count="exact")
                .eq("drama_title", f"{FAV_STORE_PREFIX}{drama_title}")
                .execute()
            )
            likes_count = int(l_res.count if l_res.count is not None else len(l_res.data or []))
        except Exception:
            pass

        enriched_episodes = []
        for r in (response.data or []):
            ep_num = int(r.get("ep_number") or 1)
            raw_cov_url = r.get("cover_url")
            ch_msg_id = r.get("channel_msg_id") or 0
            has_custom_ep_cov = ep_num in ep_cov_versions
            # ប្រើ Version ដាច់ដោយឡែកសម្រាប់ភាគនីមួយៗ (ep_num + channel_msg_id)
            ep_ver = ep_cov_versions.get(ep_num) or make_cover_version(
                str(raw_cov_url or f"ep_{ep_num}"),
                f"{ep_num}_{ch_msg_id}",
            )
            enriched_episodes.append({
                "ep_number": ep_num,
                "cover_url": raw_cov_url,
                "cover_version": ep_ver,
                "has_custom_cover": has_custom_ep_cov,
            })

        return {
            "title": drama_title,
            "drama_code": single_meta["drama_code"],
            "upload_date": single_meta["upload_date"],
            "duration_seconds": single_meta["duration_seconds"],
            "duration_text": single_meta["duration_text"],
            "has_gif": single_meta["has_gif"],
            "gif_version": single_meta["gif_version"],
            "tags": single_meta["tags"],
            "album": single_meta["album"],
            "is_trending": single_meta["is_trending"],
            "episodes": enriched_episodes,
            "last_ep": last_ep,
            "synopsis": single_meta["synopsis"],
            "is_favorite": is_fav,
            "views": views_count,
            "likes_count": likes_count,
            "avg_rating": avg_rating,
            "rating_count": rating_count,
            "user_rating": user_rating,
            "can_request_today": can_request_today,
            "cambodia_date": get_cambodia_date_str(),
            "is_admin": is_admin_user(user_id) if user_id else False,
        }

    except Exception:
        log.exception("Episode list API failed")
        return JSONResponse(
            status_code=500,
            content={"detail": "Could not load episodes"},
        )


@app.post("/api/rate")
def rate_drama_api(req: RateDramaRequest):
    try:
        stars = max(1, min(5, int(req.stars)))
        uid = int(req.user_id) if req.user_id else int(time.time() % 100000000)
        rate_key = f"{RATE_STORE_PREFIX}{req.title}"

        supabase.table("user_history").upsert(
            {
                "user_id": uid,
                "drama_title": rate_key,
                "last_ep": stars,
            },
            on_conflict="user_id,drama_title",
        ).execute()

        avg_rating, rating_count, user_rating = fetch_drama_rating_stats(req.title, uid)
        return {
            "status": "ok",
            "stars": stars,
            "avg_rating": avg_rating,
            "rating_count": rating_count,
            "user_rating": user_rating,
        }
    except Exception as exc:
        log.exception("Rate drama API failed")
        return JSONResponse(
            status_code=500,
            content={"detail": str(exc)},
        )


@app.post("/api/user-request")
def submit_user_request_api(req: UserFeedbackRequest):
    try:
        clean_msg = (req.message or "").strip()
        if not clean_msg:
            return JSONResponse(status_code=400, content={"detail": "Empty message"})

        uid = req.user_id

        if uid and has_user_requested_today(uid):
            return JSONResponse(
                status_code=429,
                content={
                    "detail": "អ្នកបានផ្ញើសំណើ ១ដងរួចហើយក្នុងថ្ងៃនេះ។ សូមរង់ចាំថ្ងៃស្អែក!",
                    "can_request_today": False,
                    "cambodia_date": get_cambodia_date_str(),
                },
            )

        if uid:
            mark_user_requested_today(uid)

        first_name = (req.first_name or "").strip()
        last_name = (req.last_name or "").strip()
        username = (req.username or "").strip().lstrip("@")
        drama_title = (req.drama_title or "").strip()

        if uid and not (first_name or username):
            try:
                chat_info = bot.get_chat(int(uid))
                first_name = getattr(chat_info, "first_name", "") or ""
                last_name = getattr(chat_info, "last_name", "") or ""
                username = getattr(chat_info, "username", "") or ""
            except Exception:
                pass

        full_name = f"{first_name} {last_name}".strip() or "អ្នកប្រើប្រាស់ (User)"
        escaped_name = html.escape(full_name)
        escaped_drama = html.escape(drama_title) if drama_title else "ទូទៅ"
        escaped_msg = html.escape(clean_msg)

        if uid:
            account_link = f'<a href="tg://user?id={uid}">{escaped_name}</a>'
            uid_tag = f"<code>#UID{uid}</code>"
        else:
            account_link = escaped_name
            uid_tag = "មិនស្គាល់"

        username_line = f"@{html.escape(username)}" if username else "មិនមាន (@username)"

        admin_text = (
            "📩 <b>សារស្នើសុំថ្មីពីអ្នកប្រើប្រាស់ χƒαη 18 קɭυѕ Mini App</b>\n"
            "━━━━━━━━━━━━━━━━━━\n"
            f"👤 <b>គណនី៖</b> {account_link}\n"
            f"🔗 <b>Username៖</b> {username_line}\n"
            f"🆔 <b>User ID៖</b> {uid_tag}\n"
            f"🎬 <b>ពីទំព័ររឿង៖</b> {escaped_drama}\n"
            "━━━━━━━━━━━━━━━━━━\n"
            f"💬 <b>សារស្នើសុំ៖</b>\n{escaped_msg}\n\n"
            "💡 <i>ចុច Reply លើសារនេះដើម្បីផ្ញើសារឆ្លើយតបចូលទៅក្នុង Chat Bot របស់គាត់ផ្ទាល់។</i>"
        )

        markup = InlineKeyboardMarkup()
        markup.add(
            InlineKeyboardButton(
                "👥 បើកគ្រុប Admin",
                url=ADMIN_GROUP_INVITE_LINK,
            )
        )

        def _send_to_admins():
            candidates = build_candidate_admin_targets()
            delivered_any = False

            for target_id in candidates:
                try:
                    sent_msg = bot.send_message(
                        chat_id=target_id,
                        text=admin_text,
                        parse_mode="HTML",
                        reply_markup=markup,
                        disable_web_page_preview=True,
                    )
                    delivered_any = True
                    if int(target_id) < 0:
                        save_detected_admin_group_id(int(target_id))
                    if uid and sent_msg:
                        remember_admin_request_msg(
                            target_id,
                            sent_msg.message_id,
                            int(uid),
                            drama_title=drama_title or "ទូទៅ",
                            request_text=clean_msg,
                        )
                    break
                except Exception as e:
                    log.warning("Target chat %s not reachable (%s)", target_id, e)

            if not delivered_any:
                log.error(
                    "Could not deliver user request to any Admin chat. "
                    "Please add the Bot to %s and send /id in the group!",
                    ADMIN_GROUP_INVITE_LINK,
                )

        threading.Thread(target=_send_to_admins, daemon=True).start()
        return {
            "status": "ok",
            "can_request_today": False,
            "cambodia_date": get_cambodia_date_str(),
        }

    except Exception as exc:
        log.exception("Submit user request API failed")
        return JSONResponse(
            status_code=500,
            content={"detail": str(exc)},
        )


@app.post("/api/favorite")
def toggle_favorite_api(req: FavoriteDramaRequest):
    try:
        fav_key = f"{FAV_STORE_PREFIX}{req.title}"
        if req.is_favorite:
            supabase.table("user_history").upsert(
                {
                    "user_id": req.user_id,
                    "drama_title": fav_key,
                    "last_ep": 1,
                },
                on_conflict="user_id,drama_title",
            ).execute()
        else:
            (
                supabase.table("user_history")
                .delete()
                .eq("user_id", req.user_id)
                .eq("drama_title", fav_key)
                .execute()
            )
        return {"status": "ok", "is_favorite": req.is_favorite}
    except Exception as exc:
        log.exception("Toggle favorite API failed")
        return JSONResponse(
            status_code=500,
            content={"detail": str(exc)},
        )


@app.get("/api/favorites/{user_id}")
def get_user_favorites_api(user_id: int):
    try:
        fav_res = (
            supabase.table("user_history")
            .select("drama_title")
            .eq("user_id", user_id)
            .like("drama_title", f"{FAV_STORE_PREFIX}%")
            .order("updated_at", desc=True)
            .execute()
        )

        if not fav_res.data:
            return JSONResponse(
                content={"items": [], "count": 0},
                headers={"Cache-Control": "no-store, no-cache, must-revalidate, max-age=0"},
            )

        ep_res = supabase.table("episodes").select("drama_title, cover_url, category, channel_msg_id").execute()
        active_dramas = OrderedDict()
        for r in (ep_res.data or []):
            t = (r.get("drama_title") or "").strip()
            if t and not is_internal_key(t):
                msg_id = int(r.get("channel_msg_id") or 0)
                if t not in active_dramas:
                    active_dramas[t] = {
                        "cover_url": r.get("cover_url"),
                        "category": (r.get("category") or "").strip(),
                        "latest_id": msg_id,
                    }
                elif msg_id > active_dramas[t]["latest_id"]:
                    active_dramas[t]["latest_id"] = msg_id

        meta = fetch_all_drama_metadata_maps()
        ensure_codes_and_dates_for_dramas(active_dramas, meta)
        cover_versions = meta["versions"]
        synopses_map = meta["synopses"]
        langs_map = meta["langs"]
        codes_map = meta["codes"]
        upload_dates_map = meta["upload_dates"]
        durations_map = meta["durations"]
        tags_map = meta["tags"]
        gifs_map = meta["gifs"]
        views_map = fetch_drama_views_map()

        items = []
        for row in fav_res.data:
            raw_key = row.get("drama_title") or ""
            if not raw_key.startswith(FAV_STORE_PREFIX):
                continue
            title = raw_key[len(FAV_STORE_PREFIX):].strip()
            if not title or title not in active_dramas:
                continue

            raw_cover = active_dramas[title]["cover_url"]
            ver = cover_versions.get(title, "v1")
            if raw_cover:
                ver = make_cover_version(raw_cover, ver)

            audio_type = determine_audio_type(title, langs_map.get(title))
            d_code = codes_map.get(title, "")
            u_date = upload_dates_map.get(title, get_cambodia_date_ddmmyyyy())
            dur_sec = durations_map.get(title, 0)
            gif_ver = gifs_map.get(title)
            cat_str = active_dramas[title]["category"]
            syn_str = synopses_map.get(title, "")
            t_str = tags_map.get(title, "")

            items.append({
                "title": title,
                "drama_code": d_code,
                "upload_date": u_date,
                "duration_seconds": dur_sec,
                "duration_text": format_video_duration(dur_sec) if dur_sec > 0 else "",
                "has_gif": bool(gif_ver),
                "gif_version": gif_ver or "v1",
                "cover_url": raw_cover,
                "cover_version": ver,
                "category": cat_str,
                "audio_type": audio_type,
                "views": views_map.get(title, 0),
                "latest_id": active_dramas[title]["latest_id"],
                "synopsis": syn_str,
                "tags": t_str,
                "search_tokens": build_smart_search_tokens(title, d_code, cat_str, syn_str, t_str),
                "is_favorite": True,
            })

        return JSONResponse(
            content={"items": items, "count": len(items)},
            headers={"Cache-Control": "no-store, no-cache, must-revalidate, max-age=0"},
        )

    except Exception:
        log.exception("Favorites API failed")
        return JSONResponse(
            status_code=500,
            content={"detail": "Could not load favorites"},
        )


@app.post("/api/select-drama")
def select_drama_api(req: SelectDramaRequest):
    try:
        uid = int(req.user_id)
        if not is_required_channel_member(uid):
            PENDING_WATCH_AFTER_JOIN[uid] = {
                "drama_title": req.title,
                "ep_number": int(req.ep_number or 1),
                "notif_msg_id": None,
            }
            return JSONResponse(
                status_code=403,
                content={
                    "status": "join_required",
                    "channel_link": get_required_channel_link(),
                    "channel_name": CHANNEL_DISPLAY_NAME,
                    "message": f"សូមចូល Channel {CHANNEL_DISPLAY_NAME} ជាមុនសិន។",
                },
                headers={"Cache-Control": "no-store"},
            )
        PENDING_WATCH_AFTER_JOIN.pop(uid, None)
        threading.Thread(
            target=send_drama_episode,
            args=(uid, req.title, req.ep_number),
            daemon=True,
        ).start()
        return {"status": "ok"}
    except Exception as exc:
        log.exception("Select drama API failed")
        return JSONResponse(
            status_code=500,
            content={"detail": str(exc)},
        )


@app.get("/api/gif/{drama_title}")
def get_drama_gif_preview(
    drama_title: str,
    request: Request,
    v: Optional[str] = Query(default=None),
):
    try:
        gif_key = f"{GIF_STORE_PREFIX}{drama_title}"
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id, cover_channel_msg_id")
            .eq("drama_title", gif_key)
            .limit(1)
            .execute()
        )

        gif_file_id = res.data[0].get("cover_file_id") if res.data else None
        gif_msg_id = res.data[0].get("cover_channel_msg_id") if res.data else None

        if not gif_file_id and not gif_msg_id:
            return Response(status_code=404)

        actual_version = make_cover_version(gif_file_id, gif_msg_id)
        etag_value = f'"{actual_version}"'

        if_none_match = request.headers.get("if-none-match")
        if if_none_match and if_none_match == etag_value:
            return Response(
                status_code=304,
                headers={
                    "ETag": etag_value,
                    "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                },
            )

        cache_key = f"gif::{drama_title}::{actual_version}"
        cached = get_cached_gif_bytes(cache_key)
        if cached:
            gif_bytes, media_type = cached
            return Response(
                content=gif_bytes,
                media_type=media_type,
                headers={
                    "ETag": etag_value,
                    "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                },
            )

        file_info = None
        if gif_file_id:
            try:
                file_info = bot.get_file(gif_file_id)
            except Exception:
                file_info = None

        if not file_info and gif_msg_id:
            refreshed_fid = refresh_cover_file_id_from_channel(gif_key, int(gif_msg_id))
            if refreshed_fid:
                try:
                    file_info = bot.get_file(refreshed_fid)
                except Exception:
                    file_info = None

        if file_info and getattr(file_info, "file_path", None):
            download_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{file_info.file_path}"
            gif_res = requests.get(download_url, timeout=25)
            if gif_res.status_code == 200:
                fpath = str(file_info.file_path).lower()
                if fpath.endswith(".mp4"):
                    media_type = "video/mp4"
                elif fpath.endswith(".gif"):
                    media_type = "image/gif"
                else:
                    media_type = gif_res.headers.get("Content-Type", "image/gif")
                set_cached_gif_bytes(cache_key, gif_res.content, media_type)
                return Response(
                    content=gif_res.content,
                    media_type=media_type,
                    headers={
                        "ETag": etag_value,
                        "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                    },
                )

        return Response(status_code=404)
    except Exception:
        log.exception("GIF fetch failed for %s", drama_title)
        return Response(status_code=404)


@app.get("/api/album/{drama_title}/{slot}")
def get_drama_album_photo(
    drama_title: str,
    slot: int,
    request: Request,
    v: Optional[str] = Query(default=None),
):
    if int(slot) not in (1, 2, 3):
        return Response(status_code=404)
    try:
        album_key = f"{ALBUM_STORE_PREFIX}{drama_title}__{int(slot)}"
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id, cover_channel_msg_id")
            .eq("drama_title", album_key)
            .limit(1)
            .execute()
        )
        file_id = res.data[0].get("cover_file_id") if res.data else None
        msg_id = res.data[0].get("cover_channel_msg_id") if res.data else None

        if not file_id and not msg_id:
            return Response(status_code=404)

        actual_version = make_cover_version(file_id, msg_id)
        etag_value = f'"{actual_version}"'

        if_none_match = request.headers.get("if-none-match")
        if if_none_match and if_none_match == etag_value:
            return Response(
                status_code=304,
                headers={
                    "ETag": etag_value,
                    "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                },
            )

        cache_key = f"album::{drama_title}::{slot}::{actual_version}"
        cached = get_cached_cover_bytes(cache_key)
        if cached:
            img_bytes, media_type = cached
            return Response(
                content=img_bytes,
                media_type=media_type,
                headers={
                    "ETag": etag_value,
                    "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                },
            )

        file_info = None
        if file_id:
            try:
                file_info = bot.get_file(file_id)
            except Exception:
                file_info = None

        if not file_info and msg_id:
            refreshed_fid = refresh_cover_file_id_from_channel(album_key, int(msg_id))
            if refreshed_fid:
                try:
                    file_info = bot.get_file(refreshed_fid)
                except Exception:
                    file_info = None

        if file_info and getattr(file_info, "file_path", None):
            download_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{file_info.file_path}"
            image_res = requests.get(download_url, timeout=25)
            if image_res.status_code == 200:
                media_type = image_res.headers.get("Content-Type", "image/jpeg")
                set_cached_cover_bytes(cache_key, image_res.content, media_type)
                return Response(
                    content=image_res.content,
                    media_type=media_type,
                    headers={
                        "ETag": etag_value,
                        "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                    },
                )

        return Response(status_code=404)
    except Exception:
        log.exception("Album photo fetch failed for %s slot %s", drama_title, slot)
        return Response(status_code=404)


@app.get("/api/ep-cover/{drama_title}/{ep_number}")
def get_episode_cover_image(
    drama_title: str,
    ep_number: int,
    request: Request,
    v: Optional[str] = Query(default=None),
):
    """
    ទាញយករូបភាព Cover នៃវីដេអូភាគនីមួយៗ៖
    1) ពិនិត្យ __EPCOV__{drama_title}__{ep_number}
    2) បើមិនទាន់មាន ទាញយក Thumbnail/Frame ផ្ទាល់ពីវីដេអូនៃភាគនោះ (channel_msg_id) ក្នុង Channel ដោយស្វ័យប្រវត្តិ
    """
    try:
        ep_key = f"{EP_COV_STORE_PREFIX}{drama_title}__{int(ep_number)}"
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id, cover_channel_msg_id")
            .eq("drama_title", ep_key)
            .limit(1)
            .execute()
        )

        ep_file_id = res.data[0].get("cover_file_id") if res.data else None
        ep_msg_id = res.data[0].get("cover_channel_msg_id") if res.data else None

        if ep_file_id or ep_msg_id:
            actual_version = make_cover_version(ep_file_id, ep_msg_id)
            etag_value = f'"{actual_version}"'

            if_none_match = request.headers.get("if-none-match")
            if if_none_match and if_none_match == etag_value:
                return Response(
                    status_code=304,
                    headers={
                        "ETag": etag_value,
                        "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                    },
                )

            cache_key = f"epcov::{drama_title}::{ep_number}::{actual_version}"
            cached = get_cached_cover_bytes(cache_key)
            if cached:
                img_bytes, media_type = cached
                return Response(
                    content=img_bytes,
                    media_type=media_type,
                    headers={
                        "ETag": etag_value,
                        "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                    },
                )

            file_info = None
            if ep_file_id:
                try:
                    file_info = bot.get_file(ep_file_id)
                except Exception:
                    file_info = None

            if not file_info and ep_msg_id:
                refreshed_fid = refresh_cover_file_id_from_channel(ep_key, int(ep_msg_id))
                if refreshed_fid:
                    try:
                        file_info = bot.get_file(refreshed_fid)
                    except Exception:
                        file_info = None

            if file_info and getattr(file_info, "file_path", None):
                download_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{file_info.file_path}"
                image_res = requests.get(download_url, timeout=25)
                if image_res.status_code == 200:
                    media_type = image_res.headers.get("Content-Type", "image/jpeg")
                    set_cached_cover_bytes(cache_key, image_res.content, media_type)
                    return Response(
                        content=image_res.content,
                        media_type=media_type,
                        headers={
                            "ETag": etag_value,
                            "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                        },
                    )

        # បើមិនទាន់មានក្នុង __EPCOV__ ទេ ទាញយកចេញពីវីដេអូភាគនោះផ្ទាល់នៅក្នុង table episodes
        try:
            ep_row = (
                supabase.table("episodes")
                .select("cover_url, channel_msg_id")
                .eq("drama_title", drama_title)
                .eq("ep_number", int(ep_number))
                .limit(1)
                .execute()
            )
            if ep_row.data:
                ext_url = ep_row.data[0].get("cover_url")
                ch_msg_id = ep_row.data[0].get("channel_msg_id")

                if ext_url and str(ext_url).startswith(("http://", "https://")):
                    ext_res = requests.get(str(ext_url), timeout=15)
                    if ext_res.status_code == 200:
                        media_type = ext_res.headers.get("Content-Type", "image/jpeg")
                        return Response(
                            content=ext_res.content,
                            media_type=media_type,
                            headers={"Cache-Control": "public, max-age=3600"},
                        )

                if ch_msg_id:
                    extracted_bytes = extract_and_cache_episode_cover_from_channel(
                        drama_title=drama_title,
                        ep_number=int(ep_number),
                        channel_msg_id=int(ch_msg_id),
                    )
                    if extracted_bytes:
                        cache_key = f"epcov::{drama_title}::{ep_number}::ch_{ch_msg_id}"
                        set_cached_cover_bytes(cache_key, extracted_bytes, "image/jpeg")
                        return Response(
                            content=extracted_bytes,
                            media_type="image/jpeg",
                            headers={"Cache-Control": "public, max-age=86400"},
                        )
        except Exception:
            log.warning("Fallback episode video cover extraction failed for %s ep %s", drama_title, ep_number)

        return get_drama_cover(drama_title=drama_title, request=request, v=v)
    except Exception:
        log.exception("Episode cover fetch failed")
        return get_drama_cover(drama_title=drama_title, request=request, v=v)


@app.get("/api/cover/{drama_title}")
def get_drama_cover(
    drama_title: str,
    request: Request,
    v: Optional[str] = Query(default=None),
):
    try:
        res = (
            supabase.table("drama_covers")
            .select("cover_file_id, cover_channel_msg_id")
            .eq("drama_title", drama_title)
            .limit(1)
            .execute()
        )

        cover_file_id = res.data[0].get("cover_file_id") if res.data else None
        cover_msg_id = res.data[0].get("cover_channel_msg_id") if res.data else None

        if not cover_file_id and not cover_msg_id:
            return Response(
                content=FALLBACK_SVG_BYTES,
                media_type="image/svg+xml",
                headers={
                    "Cache-Control": "no-store, no-cache, must-revalidate, max-age=0",
                    "Pragma": "no-cache",
                    "X-Cover-Fallback": "1",
                },
            )

        actual_version = make_cover_version(cover_file_id, cover_msg_id)
        etag_value = f'"{actual_version}"'

        if_none_match = request.headers.get("if-none-match")
        if if_none_match and if_none_match == etag_value:
            return Response(
                status_code=304,
                headers={
                    "ETag": etag_value,
                    "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                },
            )

        cache_key = f"{drama_title}::{actual_version}"
        cached = get_cached_cover_bytes(cache_key)
        if cached:
            img_bytes, media_type = cached
            return Response(
                content=img_bytes,
                media_type=media_type,
                headers={
                    "ETag": etag_value,
                    "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                },
            )

        file_info = None
        if cover_file_id:
            try:
                file_info = bot.get_file(cover_file_id)
            except Exception:
                log.warning("Invalid cover_file_id for %s, attempting channel recovery...", drama_title)
                file_info = None

        if not file_info and cover_msg_id:
            refreshed_fid = refresh_cover_file_id_from_channel(drama_title, int(cover_msg_id))
            if refreshed_fid:
                try:
                    file_info = bot.get_file(refreshed_fid)
                except Exception:
                    file_info = None

        if file_info and getattr(file_info, "file_path", None):
            download_url = f"https://api.telegram.org/file/bot{BOT_TOKEN}/{file_info.file_path}"
            image_res = requests.get(download_url, timeout=25)
            if image_res.status_code == 200:
                media_type = image_res.headers.get("Content-Type", "image/jpeg")
                set_cached_cover_bytes(cache_key, image_res.content, media_type)
                return Response(
                    content=image_res.content,
                    media_type=media_type,
                    headers={
                        "ETag": etag_value,
                        "Cache-Control": "public, max-age=31536000, immutable" if v else "public, max-age=3600, must-revalidate",
                    },
                )

        try:
            ep_cov = (
                supabase.table("episodes")
                .select("cover_url")
                .eq("drama_title", drama_title)
                .limit(1)
                .execute()
            )
            ext_url = ep_cov.data[0].get("cover_url") if ep_cov.data else None
            if ext_url and str(ext_url).startswith(("http://", "https://")):
                ext_res = requests.get(str(ext_url), timeout=15)
                if ext_res.status_code == 200:
                    media_type = ext_res.headers.get("Content-Type", "image/jpeg")
                    set_cached_cover_bytes(cache_key, ext_res.content, media_type)
                    return Response(
                        content=ext_res.content,
                        media_type=media_type,
                        headers={"Cache-Control": "public, max-age=3600"},
                    )
        except Exception:
            pass

        return Response(
            content=FALLBACK_SVG_BYTES,
            media_type="image/svg+xml",
            headers={
                "Cache-Control": "no-store, no-cache, must-revalidate, max-age=0",
                "Pragma": "no-cache",
                "X-Cover-Fallback": "1",
            },
        )

    except Exception:
        log.exception("Cover fetch failed")
        return Response(
            content=FALLBACK_SVG_BYTES,
            media_type="image/svg+xml",
            headers={
                "Cache-Control": "no-store, no-cache, must-revalidate, max-age=0",
                "Pragma": "no-cache",
                "X-Cover-Fallback": "1",
            },
        )


@app.get("/api/history/{user_id}")
def get_user_history_api(user_id: int):
    try:
        hist_res = (
            supabase.table("user_history")
            .select("drama_title, last_ep")
            .eq("user_id", user_id)
            .order("updated_at", desc=True)
            .execute()
        )

        if not hist_res.data:
            return JSONResponse(
                content={"items": [], "count": 0},
                headers={"Cache-Control": "no-store, no-cache, must-revalidate, max-age=0"},
            )

        ep_res = supabase.table("episodes").select("drama_title, cover_url, category, channel_msg_id").execute()
        active_dramas = OrderedDict()
        for r in (ep_res.data or []):
            t = (r.get("drama_title") or "").strip()
            if t and not is_internal_key(t):
                msg_id = int(r.get("channel_msg_id") or 0)
                if t not in active_dramas:
                    active_dramas[t] = {
                        "cover_url": r.get("cover_url"),
                        "category": (r.get("category") or "").strip(),
                        "latest_id": msg_id,
                    }
                elif msg_id > active_dramas[t]["latest_id"]:
                    active_dramas[t]["latest_id"] = msg_id

        meta = fetch_all_drama_metadata_maps()
        ensure_codes_and_dates_for_dramas(active_dramas, meta)
        cover_versions = meta["versions"]
        synopses_map = meta["synopses"]
        langs_map = meta["langs"]
        codes_map = meta["codes"]
        upload_dates_map = meta["upload_dates"]
        durations_map = meta["durations"]
        tags_map = meta["tags"]
        gifs_map = meta["gifs"]
        views_map = fetch_drama_views_map()

        items = []
        for row in hist_res.data:
            title = (row.get("drama_title") or "").strip()
            if not title or is_internal_key(title) or title not in active_dramas:
                continue

            raw_cover = active_dramas[title]["cover_url"]
            ver = cover_versions.get(title, "v1")
            if raw_cover:
                ver = make_cover_version(raw_cover, ver)

            audio_type = determine_audio_type(title, langs_map.get(title))
            d_code = codes_map.get(title, "")
            u_date = upload_dates_map.get(title, get_cambodia_date_ddmmyyyy())
            dur_sec = durations_map.get(title, 0)
            gif_ver = gifs_map.get(title)
            cat_str = active_dramas[title]["category"]
            syn_str = synopses_map.get(title, "")
            t_str = tags_map.get(title, "")

            items.append({
                "title": title,
                "drama_code": d_code,
                "upload_date": u_date,
                "duration_seconds": dur_sec,
                "duration_text": format_video_duration(dur_sec) if dur_sec > 0 else "",
                "has_gif": bool(gif_ver),
                "gif_version": gif_ver or "v1",
                "last_ep": row.get("last_ep"),
                "cover_url": raw_cover,
                "cover_version": ver,
                "category": cat_str,
                "audio_type": audio_type,
                "views": views_map.get(title, 0),
                "latest_id": active_dramas[title]["latest_id"],
                "synopsis": syn_str,
                "tags": t_str,
                "search_tokens": build_smart_search_tokens(title, d_code, cat_str, syn_str, t_str),
            })

        return JSONResponse(
            content={"items": items, "count": len(items)},
            headers={"Cache-Control": "no-store, no-cache, must-revalidate, max-age=0"},
        )

    except Exception:
        log.exception("History API failed")
        return JSONResponse(
            status_code=500,
            content={"detail": "Could not load history"},
        )
