# CachyOS Smart Media Downloader (`vdl`)

> A professional, lightning-fast, and intelligent CLI media downloader wrapper built for CachyOS and Fish shell. It automatically routes downloads between `yt-dlp` (with native HLS/DASH fragment concurrency) and `aria2c` (multi-threaded direct file engine).

---

## Table of Contents
- [Overview](#overview)
- [Key Features](#key-features)
- [Prerequisites](#prerequisites)
- [Installation & Setup](#installation--setup)
  - [1. yt-dlp Configuration](#1-yt-dlp-configuration)
  - [2. vdl Fish Shell Function](#2-vdl-fish-shell-function)
  - [3. Tab Auto-Completion](#3-tab-auto-completion)
- [Usage Guide & Command Reference](#usage-guide--command-reference)
- [Browser Integration (The Stream Detector)](#browser-integration-the-stream-detector)
- [Troubleshooting & Corner Cases](#troubleshooting--corner-cases)

---

## Overview

The `vdl` utility is designed to streamline media acquisition on Linux. Instead of manually juggling between terminal flags, extracting browser cookies, or dealing with throttled network streams, `vdl` intelligently inspects the target URL or leverages clipboard integration to execute the optimal download strategy.

### Key Features
* **Smart Auto-Routing:** Automatically uses `yt-dlp` native fragment downloader for streaming manifests (`.m3u8` / `.mpd`), switches to `yt-dlp + aria2c` for web video pages/playlists, and routes direct file downloads (`.zip`, `.iso`, `.mp4`) straight to `aria2c` with 16-thread multi-segment downloading.
* **Clipboard Auto-Grab:** If invoked without arguments (`vdl`), it automatically reads the system clipboard (Wayland/X11) for valid URLs, eliminating manual pasting and shell wildcard errors with special characters (`?`, `&`).
* **Zero-Clutter Playlist Management:** Single videos save directly to `~/Downloads`, while playlists automatically create clean subfolders with sequential numbering (`Playlist Title/01 - Video Title.mp4`).
* **Browser Cookie Support:** Seamlessly integrates with Vivaldi (default), Firefox, Brave, Chrome, or Chromium via simple command-line flags.
* **Professional Terminal UI:** Features a clean, non-breaking minimalist tree header (`╭─`, `├─`, `╰─`) displaying active engines, modes, and destination directories.

---

## Prerequisites

Ensure the following core dependencies are installed on your CachyOS system:

```bash
sudo pacman -S yt-dlp aria2 ffmpeg
```

---

## Installation & Setup

### 1. yt-dlp Configuration
Create or overwrite your global `yt-dlp` configuration file to establish robust fallback timeouts, retry rules, and default formatting:

```bash
mkdir -p ~/.config/yt-dlp

printf "%s\n" \
"# Default resolution cap: Prefer up to 1080p" \
"-S \"res:1080\"" \
"" \
"# Impersonate Chrome to bypass Cloudflare 403" \
"--impersonate chrome" \
"" \
"# Use yt-dlp native engine for HLS (.m3u8) and DASH (.mpd) with 8 concurrent fragments" \
"--downloader m3u8:native" \
"--downloader mpd:native" \
"-N 8" \
"--socket-timeout 15" \
"--retries 10" \
"--fragment-retries 10" \
"--continue" \
"--no-overwrites" \
"" \
"# Use aria2c (10 threads max per server to avoid SSL handshake drops) for HTTP/HTTPS" \
"--downloader http:aria2c" \
"--downloader https:aria2c" \
"--downloader-args \"aria2c:-x 10 -s 10 -k 1M --file-allocation=none --summary-interval=0\"" \
"" \
"# Clean progress bar and merge video streams into MP4" \
"--progress" \
"--console-title" \
"--merge-output-format mp4" \
"" \
"# Save inside ~/Downloads/vdl (Creates real subfolder ONLY for playlists/albums)" \
"-P ~/Downloads/vdl" \
"-o \"%(playlist_title,album,NA)s/%(playlist_index&{} - |)s%(title).80B [%(id)s].%(ext)s\"" \
"--replace-in-metadata \"playlist_title,album\" \"^NA\$\" \".\"" > ~/.config/yt-dlp/config
```

### 2. vdl Fish Shell Function
Define the core `vdl` function in your Fish shell configuration. Paste the following block into your terminal:

```fish
functions -e vdl 2>/dev/null

function vdl --wraps=yt-dlp --description 'All-in-One Smart Downloader (yt-dlp + aria2c)'
    set -l browser "vivaldi"
    set -l res "1080"
    set -l res_custom 0
    set -l audio_only 0
    set -l show_help 0
    set -l user_agent ""
    set -l referer ""
    set -l args
    set -l urls
    
    # 1. Parse all arguments & corner cases safely
    set -l i 1
    while test $i -le (count $argv)
        switch $argv[$i]
            case -h --help help
                set show_help 1
            case -b --browser --cookies-from-browser
                set i (math $i + 1)
                if test $i -le (count $argv)
                    set browser (string lower -- $argv[$i])
                end
            case -n --no-cookies
                set browser "none"
            case -a --audio
                set audio_only 1
            case -an -na
                set audio_only 1
                set browser "none"
            case -q --quality
                set i (math $i + 1)
                if test $i -le (count $argv)
                    set res (string replace -i "p" "" -- $argv[$i])
                    if test "$res" = "4k" -o "$res" = "4K"
                        set res "2160"
                    else if test "$res" = "2k" -o "$res" = "2K"
                        set res "1440"
                    end
                    set res_custom 1
                end
            case --user-agent
                set i (math $i + 1)
                if test $i -le (count $argv)
                    set user_agent $argv[$i]
                    set -a args "--user-agent" $argv[$i]
                end
            case --referer
                set i (math $i + 1)
                if test $i -le (count $argv)
                    set referer $argv[$i]
                    set -a args "--referer" $argv[$i]
                end
            case 'http://*' 'https://*'
                set -a urls $argv[$i]
                set -a args $argv[$i]
            case '*'
                set -a args $argv[$i]
        end
        set i (math $i + 1)
    end
    
    # 2. Display Custom Help Menu if -h / --help / help is passed
    if test $show_help -eq 1
        echo ""
        set_color -o cyan;   printf "╭─[::] "; set_color -o white; echo "vdl // Help & Command Reference"
        set_color -o cyan;   printf "├─ >>  "; set_color -o yellow; printf "Usage:\n"
        set_color -o cyan;   printf "│      "; set_color green;  printf "vdl "; set_color normal; echo "[options] \"<url>\""
        set_color -o cyan;   printf "│      "; set_color green;  printf "vdl "; set_color brblack; echo "(no URL = auto-grabs link from clipboard)"
        set_color -o cyan;   printf "│\n"
        set_color -o cyan;   printf "├─ >>  "; set_color -o yellow; printf "Custom Flags & Values:\n"
        set_color -o cyan;   printf "│      "; set_color -o green; printf "-b, --browser <name>  "; set_color normal; echo "Cookie source (Default: vivaldi)"
        set_color -o cyan;   printf "│      "; set_color brblack;  echo "                      Values: vivaldi, firefox, brave, chromium, chrome, none"
        set_color -o cyan;   printf "│      "; set_color -o green; printf "-n, --no-cookies      "; set_color normal; echo "Disable browser cookies completely"
        set_color -o cyan;   printf "│      "; set_color -o green; printf "-q, --quality <res>   "; set_color normal; echo "Max video resolution (Default: 1080)"
        set_color -o cyan;   printf "│      "; set_color brblack;  echo "                      Values: 480, 720, 1080, 1440 (2k), 2160 (4k)"
        set_color -o cyan;   printf "│      "; set_color -o green; printf "-a, --audio           "; set_color normal; echo "Extract best audio only as MP3 (320kbps)"
        set_color -o cyan;   printf "│      "; set_color -o green; printf "-h, --help            "; set_color normal; echo "Show this help screen"
        set_color -o cyan;   printf "│\n"
        set_color -o cyan;   printf "├─ >>  "; set_color -o yellow; printf "Auto-Routing Engines:\n"
        set_color -o cyan;   printf "│      "; set_color magenta; printf "Streams (.m3u8/.mpd)  "; set_color normal; echo "-> yt-dlp native (8x concurrent fragments)"
        set_color -o cyan;   printf "│      "; set_color magenta; printf "Web Video / Playlists "; set_color normal; echo "-> yt-dlp + aria2c (16x threads)"
        set_color -o cyan;   printf "│      "; set_color magenta; printf "Direct Files & .mp4   "; set_color normal; echo "-> aria2c direct engine (16x threads)"
        set_color -o cyan;   printf "│\n"
        set_color -o cyan;   printf "╰─ >>  "; set_color -o yellow; printf "Examples:\n"
        set_color -o cyan;   printf "       "; set_color white;  echo "vdl                                  # Download link from clipboard"
        set_color -o cyan;   printf "       "; set_color white;  echo "vdl \"https://x.com/...\"              # 1080p MP4 with Vivaldi cookies"
        set_color -o cyan;   printf "       "; set_color white;  echo "vdl -q 720 -b firefox \"<url>\"        # 720p MP4 with Firefox cookies"
        set_color -o cyan;   printf "       "; set_color white;  echo "vdl -a \"<url>\"                       # Download as MP3 audio"
        set_color -o cyan;   printf "       "; set_color white;  echo "vdl -n \"https://site.com/file.iso\"   # Fast direct file via aria2c"
        set_color normal
        echo ""
        return 0
    end
    
    # 3. Corner Case: If no URL given, auto-grab from Clipboard (Wayland/X11)
    if test (count $urls) -eq 0
        set -l clip ""
        if command -q wl-paste
            set clip (wl-paste -n 2>/dev/null | string trim)
        else if command -q xclip
            set clip (xclip -o -selection clipboard 2>/dev/null | string trim)
        end
        
        if string match -q -r '^https?://' -- "$clip"
            set -a urls "$clip"
            set -a args "$clip"
            set_color -o brblack; echo "[::] Auto-grabbed URL from clipboard"; set_color normal
        else
            set_color -o red; echo "[!] Error: No URL provided and no valid link found in clipboard."; set_color normal
            set_color brblack; echo "    Run 'vdl -h' to see all available commands and options."; set_color normal
            return 1
        end
    end
    
    set -l primary_url $urls[1]
    
    # 4. Detect direct files (.zip, .iso, etc., plus direct .mp4/.mkv when not extracting audio)
    set -l is_direct_file 0
    if string match -q -i -r '\.(zip|rar|7z|iso|exe|msi|dmg|pkg|deb|rpm|appimage|flatpak|tar|gz|xz|zst|bz2|pdf|epub|apk|bin|img)(\?.*)?$' -- "$primary_url"
        set is_direct_file 1
    else if test $audio_only -eq 0; and string match -q -i -r '\.(mp4|mkv|webm|avi|mov|flv)(\?.*)?$' -- "$primary_url"
        set is_direct_file 1
    end
    
    # 5. Set dynamic display strings for Konsole header
    set -l engine_str "yt-dlp (auto: hlsnative 8x / aria2c 16x)"
    set -l mode_str "Video (Max "$res"p MP4)"
    
    if test $is_direct_file -eq 1
        set engine_str "aria2c Direct Engine (16 threads)"
        set mode_str "Direct File / Video Download"
    else if test $audio_only -eq 1
        set mode_str "Audio Only (Best MP3 320kbps)"
    else if string match -q -i -r '\.(m3u8|mpd)(\?.*)?$' -- "$primary_url"
        set engine_str "yt-dlp Native Stream Engine (8x fragments)"
    end
    
    # 6. Print Minimalist, Non-Breaking Konsole Header
    echo ""
    set_color -o cyan;   printf "╭─[::] "; set_color -o white; echo "vdl // CachyOS Smart Downloader"
    set_color -o cyan;   printf "├─ >>  "; set_color normal;   printf "Engine   : "; set_color -o green;   echo "$engine_str"
    set_color -o cyan;   printf "├─ >>  "; set_color normal;   printf "Mode     : "; set_color -o blue;    echo "$mode_str"
    if test $is_direct_file -eq 0
        set_color -o cyan; printf "├─ >>  "; set_color normal; printf "Cookies  : "; set_color -o yellow;  echo "$browser"
    end
    set_color -o cyan;   printf "╰─ >>  "; set_color normal;   printf "Save Dir : "; set_color -o magenta; echo "~/Downloads/vdl"
    set_color normal
    echo ""
    
    # 7. Execute appropriate engine
    if test $is_direct_file -eq 1
        # Fix: Direct files will now also save inside ~/Downloads/vdl
        set -l aria_args -x 16 -s 16 -k 1M --continue=true --file-allocation=none -d ~/Downloads/vdl
        if test -n "$user_agent"
            set -a aria_args --user-agent="$user_agent"
        end
        if test -n "$referer"
            set -a aria_args --referer="$referer"
        end
        aria2c $aria_args $urls
    else
        set -l ytdlp_opts
        if test "$browser" != "none"
            set -a ytdlp_opts --cookies-from-browser $browser
        end
        if test $audio_only -eq 1
            # Override video config rules cleanly for pure MP3 extraction
            set -a ytdlp_opts -f "bestaudio/best" -x --audio-format mp3 --audio-quality 0
        else if test $res_custom -eq 1
            set -a ytdlp_opts -S "res:$res"
        end
        yt-dlp $ytdlp_opts $args
    end
end

funcsave vdl
```

### 3. Tab Auto-Completion
Set up auto-completion so pressing Tab displays available flags and parameters:

```fish
mkdir -p ~/.config/fish/completions

printf "%s\n" \
"complete -c vdl -f" \
"complete -c vdl -s h -l help -d 'Show help and command reference'" \
"complete -c vdl -s b -l browser -d 'Select browser for cookies (Default: vivaldi)' -xa 'vivaldi firefox brave chromium chrome edge opera none'" \
"complete -c vdl -s n -l no-cookies -d 'Disable browser cookies'" \
"complete -c vdl -s q -l quality -d 'Set max video resolution (Default: 1080)' -xa '360 480 720 1080 1440 2160 2k 4k'" \
"complete -c vdl -s a -l audio -d 'Download audio only as MP3 (320kbps)'" \
"complete -c vdl -s F -d 'List all available formats/resolutions of URL'" > ~/.config/fish/completions/vdl.fish
```

---

## Usage Guide & Command Reference

| Action | Command Example | Description |
|---|---|---|
| Auto-Grab Clipboard | `vdl` | Reads clipboard URL automatically; no quoting needed. |
| Standard Download | `vdl "https://url.com/vid"` | Downloads up to 1080p MP4 using Vivaldi cookies. |
| Custom Resolution | `vdl -q 720 "https://url.com/vid"` | Caps max resolution to 720p (supports 480, 720, 1080, 2k, 4k). |
| Alternative Browser | `vdl -b firefox "https://url.com"` | Uses Firefox cookies instead of the default Vivaldi. |
| No Cookies (Public) | `vdl -n "https://url.com"` | Disables browser cookie extraction entirely. |
| Audio Extraction | `vdl -a "https://url.com"` | Downloads best audio source converted directly to MP3. |
| Direct File Download | `vdl "https://site.com/file.iso"` | Bypasses stream parsing and uses 16-thread aria2c. |
| Help Reference | `vdl -h` | Displays the built-in minimalist reference screen. |

---

## Browser Integration (The Stream Detector)

For complex third-party streaming sites utilizing obfuscated manifests (`.m3u8` / `.mpd`):

1. Install **The Stream Detector** extension in your browser.
2. Navigate to extension settings and configure "User-defined commands:" to:
   ```bash
   vdl --user-agent "%useragent%" --referer "%origin%" "%url%"
   ```
3. Set "Copy stream URL as:" to **User-defined command 1**. Clicking the extension icon will instantly pipe authenticated streaming headers into your local `vdl` function.

---

## Troubleshooting & Corner Cases

* **Fish Shell Wildcards:** URLs containing `?` or `&` characters do not require quotation marks when using auto-clipboard grab (`vdl`), preventing shell globbing errors.
* **Direct File Detection:** Standard extensions (`.zip`, `.iso`, `.tar.zst`, `.mp4`) skip extractor overhead, immediately handing off execution to `aria2c`.
* **Playlist Hierarchies:** Playlists automatically generate designated subfolders inside `~/Downloads`, keeping your root download directory organized.
