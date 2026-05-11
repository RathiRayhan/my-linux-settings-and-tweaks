# MPV Auto-Resume Session Setup

This guide configures the `mpv` media player to automatically save the history of your last played file or URL, along with its exact playback position. The next time you launch `mpv` without any arguments, it will seamlessly resume from where you left off.

## Step 1: Create Necessary Directories

First, create the required directories for mpv scripts, script options, and the watch-later history. Run the following command in your terminal:

```bash
mkdir -p ~/.config/mpv/{scripts,script-opts} ~/.local/state/mpv/watch_later ~/.config/mpv/watch_later

```

*(Note: Depending on your mpv version and system, the `watch_later` directory might be read from different locations, so creating both ensures compatibility.)*

## Step 2: Install the `keep-session.lua` Script

This Lua script is responsible for remembering your playlist and URL history. Download it directly from GitHub into your scripts folder:

```bash
curl -L https://raw.githubusercontent.com/CogentRedTester/mpv-scripts/master/keep-session.lua -o ~/.config/mpv/scripts/keep-session.lua

```

## Step 3: Configure the Script (`keep_session.conf`)

To override the script's default settings and enable automatic saving and loading, create a configuration file in the `script-opts` directory:

```bash
nano ~/.config/mpv/script-opts/keep_session.conf

```

Paste the following lines into the file, then save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`):

```ini
auto_save=yes
auto_load=yes
maintain_pos=yes

```

## Step 4: Main MPV Configuration (`mpv.conf`)

To ensure `mpv` always remembers the exact timestamp (playback position) when you quit, you need to add a line to your main `mpv.conf` file:

```bash
nano ~/.config/mpv/mpv.conf

```

Add the following line at the bottom of the file, then save and exit:

```ini
save-position-on-quit=yes

```

---

## Testing / Usage

1. Play any local video or cloud URL from your terminal:

```bash
mpv "https://your-video-url-or-local-file.mkv"

```

2. Let the video play for a few seconds, then quit the player by pressing `Q` on your keyboard or closing the window.
3. Now, launch mpv from the terminal without any arguments:

```bash
mpv

```

**Result:** The last played video will automatically load and resume from the exact second you closed it!
