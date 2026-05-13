# Enable Clipboard Pasting (Ctrl+V) in MPV

A clean and simple guide to enable clipboard pasting in the `mpv` media player. This setup allows you to open `mpv`, press `Ctrl+V`, and instantly stream a copied URL (e.g., a YouTube or direct video link) from your clipboard.

## 📦 Prerequisites

Depending on your operating system and display server, ensure you have the required clipboard utility installed:

- **Wayland (Linux):** `wl-clipboard`
- **X11 (Linux):** `xclip`
- **macOS / Windows:** No extra dependencies required (uses built-in `pbpaste` or PowerShell).

## 🚀 Installation Guide

### Step 1: Keep MPV Open (Optional but Recommended)
By default, if you launch `mpv` from an application menu without a file, it will immediately close. To keep it open and waiting for your input, add the following to your `mpv.conf` file (located at `~/.config/mpv/mpv.conf` on Linux):

```ini
# Keep the player open when there is no audio/video to play
idle=yes
profile=pseudo-gui

```

### Step 2: Add the Clipboard Engine Script

We will use a highly efficient script by [CogentRedTester](https://github.com/CogentRedTester/mpv-clipboard) that acts as the backend engine for clipboard operations.

1. Navigate to your `mpv` scripts directory: `~/.config/mpv/scripts/`
2. Create a new file named `clipboard.lua`.
3. Paste the following code into `clipboard.lua` and save it:

```lua
--[[
    A simple script that provides extremely low-level clipboard commands.
    Original Author: CogentRedTester
    Source: [https://github.com/CogentRedTester/mpv-clipboard](https://github.com/CogentRedTester/mpv-clipboard)
]]

local mp = require 'mp'
local msg = require 'mp.msg'

local function detect_platform()
    local o = {}
    if mp.get_property_native('options/vo-mmcss-profile', o) ~= o then
        return 'windows'
    elseif mp.get_property_native('options/macos-force-dedicated-gpu', o) ~= o then
        return 'macos'
    elseif os.getenv('WAYLAND_DISPLAY') then
        return 'wayland'
    end
    return 'x11'
end

local platform = detect_platform()

local function get_command()
    if platform == 'x11' then return 'xclip -silent -selection clipboard -in' end
    if platform == 'wayland' then return 'wl-copy' end
    if platform == 'macos' then return 'pbcopy' end
end

local function error_handler(err)
    msg.warn(debug.traceback("", 2))
    msg.error(err)
end

local function co_resume_err(...)
    local success, err = coroutine.resume(...)
    if not success then
        msg.warn(debug.traceback( (select(1, ...)) ))
        msg.error(err)
    end
    return success
end

local function co_run(fn, ...)
    local co = coroutine.create(fn)
    co_resume_err(co, ...)
end

local function escape_powershell(str)
    return '"'..string.gsub(str, '[$"`]', '`%1')..'"'
end

local function subprocess(args)
    local cmd = {
        name = 'subprocess',
        args = args,
        playback_only = false,
        capture_stdout = true
    }

    local success, res, err
    local co, main = coroutine.running()

    if main or not co then
        res, err = mp.command_native(cmd)
        success = res
    else
        mp.command_native_async(cmd, function(...) return co_resume_err(co, ...) end)
        success, res, err = coroutine.yield()
    end

    if not success then error(err) end
    res.error = res.error_string ~= '' and res.error_string or nil
    return res
end

local function get_clipboard()
    if platform == 'x11' then
        local res = subprocess({ 'xclip', '-selection', 'clipboard', '-out' })
        if not res.error then return res.stdout end
    elseif platform == 'wayland' then
        local res = subprocess({ 'wl-paste', '-n' })
        if not res.error then return res.stdout end
    elseif platform == 'windows' then
        local res = subprocess({ 'powershell', '-NoProfile', '-Command', [[& {
                Trap {
                    Write-Error -ErrorRecord $_
                    Exit 1
                }
                $clip = ""
                if (Get-Command "Get-Clipboard" -errorAction SilentlyContinue) {
                    $clip = Get-Clipboard -Raw -Format Text -TextFormatType UnicodeText
                } else {
                    Add-Type -AssemblyName PresentationCore
                    $clip = [Windows.Clipboard]::GetText()
                }
                $clip = $clip -Replace "`r",""
                $u8clip = [System.Text.Encoding]::UTF8.GetBytes($clip)
                [Console]::OpenStandardOutput().Write($u8clip, 0, $u8clip.Length)
            }]]
        })
        if not res.error then return res.stdout end
    elseif platform == 'macos' then
        local res = subprocess({ 'pbpaste' })
        if not res.error then return res.stdout end
    end
    return ''
end

local function substitute(str, clip)
    return string.gsub(str, '%b%%', function(text)
        if text == '%clip%' then return clip end
        if text == '%%' then return '%' end
    end)
end

local function set_clipboard(text)
    msg.verbose('setting clipboard text:', text)
    if platform == 'windows' then
        mp.commandv('run', 'powershell', '-NoProfile', '-command', 'set-clipboard', escape_powershell(text))
    else
        local pipe = io.popen(get_command(), 'w')
        if not pipe then return msg.error('could not open unix pipe') end
        pipe:write(text)
        pipe:close()
    end
end

local function clipboard_command(...)
    msg.verbose('received clipboard command:', ...)
    local args = {'osd-auto', ...}
    local function command()
        local clip = get_clipboard()
        for i, str in ipairs(args) do
            args[i] = substitute(str, clip)
        end
        mp.command_native(args)
    end

    if select(1, ...) == 'sync' then xpcall(command, error_handler)
    else co_run(command) end
end

local function clipboard_request(response)
    msg.verbose('received clipboard request - sending response to:', response)
    co_run(function()
        mp.commandv('script-message', response, get_clipboard())
    end)
end

mp.register_script_message('set-clipboard', set_clipboard)
mp.register_script_message('get-clipboard', clipboard_request)
mp.register_script_message('clipboard-command', clipboard_command)

```

### Step 3: Map `Ctrl+V` in `input.conf`

Now, you need to tell `mpv` what to do when you press `Ctrl+V`.

1. Open or create the `input.conf` file in your `mpv` configuration directory (`~/.config/mpv/input.conf`).
2. Add the following line and save the file:

```text
ctrl+v script-message clipboard-command loadfile "%clip%"

```

## 🎉 Usage

1. Copy a video URL (e.g., `https://youtube.com/...` or a direct `.mp4` link).
2. Open `mpv`.
3. Press `Ctrl+V`.
4. The player will instantly load and play your copied link!
