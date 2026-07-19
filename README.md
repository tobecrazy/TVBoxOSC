# TVBox

An Android TV / mobile **video aggregator** app. TVBox hosts no content of its own — it loads
user-supplied **source configs** (JSON) that point to *spiders* (pluggable scrapers) which produce
the video lists, detail pages, and playable URLs. Streams play through a bundled media player
(IJK / ExoPlayer / system), and the app also supports IPTV live TV, DLNA casting, and an embedded
HTTP control server for pushing configs/URLs to the box.

- **Launcher activity:** `HomeActivity` (registered for both `LAUNCHER` and `LEANBACK_LAUNCHER`).
- **Application class:** `com.github.tvbox.osc.base.App` — wires up OkGo, the control server, the
  Room database, QuickJS, AutoSize, and player defaults on startup.
- **Application ID:** `com.github.tvbox.osc.jun` · **versionName** `1.0.2`.

## Features

- **Pluggable spiders** — JAR/DEX (Java), JS (QuickJS), and Python (Chaquopy) scrapers.
- **Multiple source types** — XML CMS, JSON CMS, spider, and remote (新接口) configs.
- **Players** — system / IJK / ExoPlayer, with resolution switching and third-party handoff
  (MX, VLC, Kodi, Reex).
- **IPTV live TV** with EPG, DoH, and channel grouping.
- **DLNA / UPnP** casting (Cling-based renderer + controller).
- **Embedded control server** (NanoHTTPD) for the "推送" cast-to-box feature and local file serving.
- **Subtitles**, danmaku (弹幕), and short-video vertical playback with swipe-to-switch episodes.

## Modules

| Module | Purpose |
| --- | --- |
| `:app` | Everything app-facing: UI, config/API loading, spider dispatch, control server, DLNA, players. |
| `:player` | Reusable player library (ExoPlayer 2.19.1 + IJK via dkplayer-ui). |
| `:quickjs` | QuickJS wrapper for running JS spiders (namespace `com.whl.quickjs.wrapper`). |
| `:pyramid` | Chaquopy Python host (`com.undcover.freedom.pyramid`); only linked into `python*` flavors. |

## Architecture

- **Config & spider loading** — `api/ApiConfig.java` is the central singleton. It fetches/parses the
  source config JSON, builds `SourceBean` entries, and lazily instantiates the right spider per site
  via `com.github.catvod.crawler`: `JarLoader` (DEX/JAR, `type 3`), `JsLoader` (JS), and
  `pyLoader`/`PythonLoader` (Python — flavor-specific: a no-op stub for java flavors, real impl for
  python flavors).
- **Source type dispatch** — `viewmodel/SourceViewModel.java` branches every fetch on the site's
  integer `type`: `0` XML CMS, `1` JSON CMS, `3` spider, `4` remote. Results are posted to the UI as
  `MutableLiveData`.
- **Playback** — `MyVideoView` wraps the dkplayer core; `ExoMediaPlayerFactory` / `IjkMediaPlayer`
  select the engine (Hawk key `PLAY_TYPE`: 0 system, 1 IJK, 2 EXO). `VodController` /
  `LiveController` drive the on-screen UI.
- **Embedded server** — `server/ControlManager` starts `RemoteServer` (NanoHTTPD); `RequestProcess`
  subclasses handle routes.
- **Other subsystems** — `dlna/` (Cling UPnP), `subtitle/`, `data/` (Room: `AppDataBase` /
  `AppDataManager`), Hawk-persisted settings (keys in `util/HawkConfig.java`), and a greenrobot
  `EventBus` in `event/`.

## Build & Run

Build is **Gradle 9.6.1 wrapper + AGP 9.3.0 + Chaquopy 17 on JDK 17**. Use `./gradlew`.

The app has a `mode` product-flavor dimension:

- `java`, `java32`, `java64` — no bundled Python; spiders are JAR + JS only. `minSdk 23`.
- `python`, `python32`, `python64` — bundle Chaquopy so Python spiders run. `minSdk 24`.
  **All Python flavors are `arm64-v8a` only** (Chaquopy 17 + Python 3.13 has no 32-bit support;
  `python32` is now a misnomer).

```bash
./gradlew :app:assembleJavaDebug          # fastest debug build (no Python)
./gradlew :app:assemblePythonDebug        # with Python spider support (arm64 only)
./gradlew :app:installJavaDebug           # install to connected device/emulator
./gradlew :app:lintJavaDebug              # lint (abortOnError is off)
./gradlew clean
```

> There are no unit/instrumentation tests in this repo — do not assume a `test` task exists.

### Requirements

- `local.properties` must provide `sdk.dir`.
- `buildPython` is pinned in `pyramid/build.gradle`
  (`/opt/homebrew/opt/python@3.13/bin/python3.13`); adjust for your machine when building a
  Python flavor.

### Build constraints worth knowing

- `android.nonTransitiveRClass=false` in `gradle.properties` is **required** — the app relies on
  transitive R classes from library modules (e.g. `R.id.loading` from dkplayer-ui).
- cronet is pinned to `143.7445.0`; older versions cause an AGP-9 `org.chromium.net` namespace
  collision (this is why java flavors are `minSdk 23`).
- Python deps are installed via Chaquopy `pip` in `pyramid/build.gradle`. `ujson` and `jsonpath`
  are intentionally absent (no wheels); code substitutes `json` and a vendored
  `pyramid/src/python/jsonpath.py`.
- AGP 9 dropped `outputFileName`, so APKs use the default `app-<flavor>-<type>.apk` naming.

## Source config reference

Field meanings used when editing default settings / building a config:

```text
searchable  : 搜索开关          0:关闭 1:启用
filterable  : 首页可选          0:否 1:是
playerType  : 播放器类型        0:系统 1:IJK 2:EXO
采集接口类型  :                  0:xml 1:json 3:jar 4:remote
parses 解析类型 :               0:嗅探,自带播放器 1:解析,返回直链
直播参数        :               ua:自定义UA epg:节目网址 logo:台标网址
```

Example config skeleton:

```json
{
    "spider": "./your.jar",
    "wallpaper": "./api/img",
    "sites": [],
    "parses": [],
    "hosts": [
        "cache.ott.ystenlive.itv.cmvideo.cn=base-v4-free-mghy.e.cdn.chinamobile.com",
        "cache.ott.bestlive.itv.cmvideo.cn=ip"
    ],
    "lives": [],
    "rules": [],
    "doh": [
        { "name": "騰訊", "url": "https://doh.pub/dns-query" },
        { "name": "阿里", "url": "https://dns.alidns.com/dns-query" },
        { "name": "360",  "url": "https://doh.360.cn/dns-query" }
    ]
}
```

### Live stream sources

- https://zilong7728.github.io/Collect-IPTV/

## Disclaimer

TVBox does not provide, host, or bundle any video content. All media is loaded from user-supplied
source configs. Use responsibly and in accordance with the laws of your jurisdiction.

## License

See [LICENSE](LICENSE).
