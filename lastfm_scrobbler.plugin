import hashlib
import json
import threading
import traceback
import time
from typing import Any, Dict, List, Optional, Tuple

import requests

from base_plugin import BasePlugin
from client_utils import get_media_controller, run_on_queue
from ui.bulletin import BulletinHelper
from ui.settings import Divider, Header, Input, Switch, Text

__id__ = "lastfm_scrobbler"
__name__ = "Last.fm Scrobbler"
__description__ = "Автоматически скробблит музыку, которую вы слушаете в Telegram, в Last.fm"
__author__ = "@Kruz_mm"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"

API_URL = "https://ws.audioscrobbler.com/2.0/"
POLL_INTERVAL = 2.0          
MIN_TRACK_LENGTH = 30        
MAX_SCROBBLE_AFTER = 240     
UNKNOWN_DURATION_AFTER = 120 
MAX_PENDING = 500
HTTP_TIMEOUT = 10


def _ui(fn):
    """Выполнить fn в UI-потоке (нужно для bulletin)."""
    try:
        from android_utils import run_on_ui_thread
        run_on_ui_thread(fn)
    except Exception:
        try:
            fn()
        except Exception:
            pass


class LastFmScrobblerPlugin(BasePlugin):
    def _log(self, msg: str):
        try:
            lg = getattr(self, "logger", None)
            if lg is not None:
                lg.info(msg)
                return
        except Exception:
            pass
        try:
            self.log(msg)
        except Exception:
            pass

    # ------------------------------------------------------------------ lifecycle
    def on_plugin_load(self):
        self._stop = threading.Event()
        self._cur: Optional[Dict[str, Any]] = None
        self._pending: List[Dict[str, Any]] = self._load_pending()
        self._next_flush = 0.0
        self._thread = threading.Thread(target=self._loop, name="lastfm-scrobbler", daemon=True)
        self._thread.start()
        self._log("Last.fm Scrobbler loaded")

    def on_plugin_unload(self):
        try:
            self._stop.set()
        except Exception:
            pass
        self._save_pending()
        self._log("Last.fm Scrobbler unloaded")

    # ------------------------------------------------------------------ settings
    def create_settings(self) -> List[Any]:
        session = self.get_setting("session_key", "")
        user = self.get_setting("lfm_user", "")
        status = f"Вы вошли как {user}" if session else "Не авторизован"

        items: List[Any] = [
            Header(text="Last.fm"),
            Switch(
                key="enabled",
                text="Скробблинг включён",
                default=True,
                icon="msg_music",
            ),
            Switch(
                key="now_playing",
                text="Отправлять «Сейчас играет»",
                default=True,
                icon="msg_info",
            ),
            Divider(),
            Header(text="API-ключи"),
            Input(
                key="api_key",
                text="API key",
                default="",
                subtext="Создайте api-ключ на last.fm/api/account/create",
                icon="msg_settings",
            ),
            Input(
                key="api_secret",
                text="Shared secret",
                default="",
                icon="msg_secret",
            ),
            Divider(),
            Header(text="Аккаунт"),
            Input(key="username", text="Логин Last.fm", default="", icon="msg_contacts"),
            Input(
                key="password",
                text="Пароль Last.fm",
                default="",
                subtext="Нужен только для входа",
                icon="msg_permissions",
            ),
            Text(
                text="Войти",
                subtext=status,
                icon="msg_arrow_forward",
                accent=True,
                on_click=lambda view: run_on_queue(self._login),
            ),
        ]
        if session:
            items.append(
                Text(
                    text="Выйти",
                    icon="msg_leave",
                    red=True,
                    on_click=lambda view: self._logout(),
                )
            )
        return items

    # ------------------------------------------------------------------ helpers
    def _toast(self, text: str):
        _ui(lambda: BulletinHelper.show_info(text))

    def _creds(self) -> Tuple[str, str, str]:
        return (
            (self.get_setting("api_key", "") or "").strip(),
            (self.get_setting("api_secret", "") or "").strip(),
            (self.get_setting("session_key", "") or "").strip(),
        )

    def _api_call(self, params: Dict[str, Any], need_session: bool = True) -> Tuple[bool, Dict[str, Any]]:
        api_key, secret, session = self._creds()
        if not api_key or not secret:
            return False, {"message": "Не заданы API key / secret"}
        if need_session and not session:
            return False, {"message": "Не выполнен вход"}

        p = {k: str(v) for k, v in params.items()}
        p["api_key"] = api_key
        if need_session:
            p["sk"] = session

        raw = "".join(k + p[k] for k in sorted(p)) + secret
        p["api_sig"] = hashlib.md5(raw.encode("utf-8")).hexdigest()
        p["format"] = "json"  # не входит в подпись

        try:
            r = requests.post(API_URL, data=p, timeout=HTTP_TIMEOUT)
            data = r.json()
        except Exception as e:
            return False, {"message": str(e), "network": True}

        if isinstance(data, dict) and "error" in data:
            if data.get("error") == 9:  # Invalid session key
                self.set_setting("session_key", "", reload_settings=True)
            return False, {"message": data.get("message", "error"), "code": data.get("error")}
        return True, data

    # ------------------------------------------------------------------ auth
    def _login(self):
        api_key, secret, _ = self._creds()
        username = (self.get_setting("username", "") or "").strip()
        password = self.get_setting("password", "") or ""
        if not (api_key and secret and username and password):
            self._toast("Last.fm: заполните API key, secret, логин и пароль")
            return

        ok, data = self._api_call(
            {"method": "auth.getMobileSession", "username": username, "password": password},
            need_session=False,
        )
        if not ok:
            self._toast(f"Last.fm: ошибка входа — {data.get('message')}")
            return

        sess = data.get("session", {})
        key = sess.get("key")
        if not key:
            self._toast("Last.fm: сервер не вернул session key")
            return

        self.set_setting("lfm_user", sess.get("name", username))
        self.set_setting("password", "")  # пароль больше не нужен
        self.set_setting("session_key", key, reload_settings=True)
        self._toast(f"Last.fm: вход выполнен ({sess.get('name', username)})")

    def _logout(self):
        self.set_setting("session_key", "")
        self.set_setting("lfm_user", "", reload_settings=True)
        self._toast("Last.fm: вы вышли из аккаунта")

    # ------------------------------------------------------------------ pending queue
    def _load_pending(self) -> List[Dict[str, Any]]:
        try:
            return json.loads(self.get_setting("pending", "[]") or "[]")
        except Exception:
            return []

    def _save_pending(self):
        try:
            self.set_setting("pending", json.dumps(self._pending[-MAX_PENDING:]))
        except Exception:
            pass

    def _flush_pending(self):
        if not self._pending or time.time() < self._next_flush:
            return
        batch = self._pending[:50]
        params: Dict[str, Any] = {"method": "track.scrobble"}
        for i, t in enumerate(batch):
            params[f"artist[{i}]"] = t["artist"]
            params[f"track[{i}]"] = t["title"]
            params[f"timestamp[{i}]"] = t["ts"]
            if t.get("duration"):
                params[f"duration[{i}]"] = t["duration"]
        ok, data = self._api_call(params)
        if ok:
            self._pending = self._pending[len(batch):]
            self._save_pending()
            self._next_flush = 0.0
        else:
            if data.get("code") in (4, 9, 10, 26):  
                self._next_flush = time.time() + 300
            else:
                self._next_flush = time.time() + 30
            self._log(f"Scrobble failed: {data}")

    # ------------------------------------------------------------------ now playing / scrobble
    def _send_now_playing(self, cur: Dict[str, Any]):
        params: Dict[str, Any] = {
            "method": "track.updateNowPlaying",
            "artist": cur["artist"],
            "track": cur["title"],
        }
        if cur["duration"] > 0:
            params["duration"] = cur["duration"]
        ok, data = self._api_call(params)
        if not ok:
            self._log(f"updateNowPlaying failed: {data}")

    def _scrobble(self, cur: Dict[str, Any]):
        self._pending.append(
            {
                "artist": cur["artist"],
                "title": cur["title"],
                "ts": int(cur["start_ts"]),
                "duration": cur["duration"],
            }
        )
        self._save_pending()
        self._log(f"Scrobble queued: {cur['artist']} - {cur['title']}")
        self._next_flush = 0.0
        self._flush_pending()

    # ------------------------------------------------------------------ player polling
    def _loop(self):
        while not self._stop.is_set():
            try:
                self._tick()
            except Exception:
                self._log("Scrobbler tick failed: " + traceback.format_exc())
            self._stop.wait(POLL_INTERVAL)

    @staticmethod
    def _read_track(msg) -> Optional[Dict[str, Any]]:
        """Достаёт artist/title/duration из MessageObject, если это музыка."""
        try:
            if msg.isVoice() or msg.isRoundVideo():
                return None
        except Exception:
            pass

        try:
            title = msg.getMusicTitle(False)
            artist = msg.getMusicAuthor(False)
        except Exception:
            title = msg.getMusicTitle()
            artist = msg.getMusicAuthor()

        title = str(title).strip() if title else ""
        artist = str(artist).strip() if artist else ""
        if not title or not artist:
            return None  

        try:
            duration = int(msg.getDuration())
        except Exception:
            duration = 0

        return {"artist": artist, "title": title, "duration": duration}

    def _tick(self):
        if not self.get_setting("enabled", True):
            self._cur = None
            return

        # Отправка накопившихся (офлайн) скробблов
        if self._pending and self.get_setting("session_key", ""):
            self._flush_pending()

        if not self.get_setting("session_key", ""):
            return

        mc = get_media_controller()
        msg = mc.getPlayingMessageObject()
        if msg is None:
            self._cur = None
            return

        info = self._read_track(msg)
        if info is None:
            self._cur = None
            return

        try:
            paused = bool(mc.isMessagePaused())
        except Exception:
            paused = False
        try:
            pos = int(msg.audioProgressSec)
        except Exception:
            pos = 0

        now = time.time()
        key = (int(msg.getDialogId()), int(msg.getId()))
        cur = self._cur

        # Новый трек, либо тот же трек пошёл по кругу после скробблинга
        restarted = (
            cur is not None
            and cur["key"] == key
            and cur["scrobbled"]
            and pos + 5 < cur["last_pos"]
            and pos < 5
        )
        if cur is None or cur["key"] != key or restarted:
            cur = {
                "key": key,
                **info,
                "start_ts": now - pos,
                "played": 0.0,
                "last_tick": now,
                "last_pos": pos,
                "scrobbled": False,
                "now_sent": False,
            }
            self._cur = cur

        # Копим реально проигранное время (паузы не считаются)
        delta = min(now - cur["last_tick"], POLL_INTERVAL * 3)
        cur["last_tick"] = now
        cur["last_pos"] = pos
        if not paused:
            cur["played"] += delta

            if not cur["now_sent"] and self.get_setting("now_playing", True):
                cur["now_sent"] = True
                self._send_now_playing(cur)

        if cur["scrobbled"]:
            return

        duration = cur["duration"]
        if duration and duration < MIN_TRACK_LENGTH:
            return
        threshold = min(duration / 2.0, MAX_SCROBBLE_AFTER) if duration else UNKNOWN_DURATION_AFTER

        if cur["played"] >= threshold:
            cur["scrobbled"] = True
            self._scrobble(cur)