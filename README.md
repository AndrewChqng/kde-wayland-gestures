# KDE Plasma Wayland Touchpad Gestures (`InputActions`)

Custom multi-finger touchpad gestures on **KDE Plasma 6 (Wayland)** to control audio volume, internal display brightness, and media playback.

---

## Gestures Overview

| Gesture | Action | Behavior |
| :--- | :--- | :--- |
| **4 Fingers Swipe Up / Down** | **Volume Up / Down** | Continuous scroll adjustment; triggers native KDE volume OSD slider |
| **4 Fingers Swipe Left / Right** | **Brightness Down / Up** | Isolates **internal laptop panel only**; leaves external monitors unchanged; triggers native KDE brightness OSD slider |
| **4 Fingers Tap** | **Play / Pause** | Toggles global media playback via `playpause` |

---

## Why InputActions?

On KDE Plasma (Wayland), touchpad gestures are handled directly inside KWin. Native System Settings reserves 4-finger swipes for window management (*Overview* and *Desktop Grid*) without a GUI option to map custom actions.

**[InputActions](https://wiki.inputactions.org/)** (`inputactions-kwin`) is a native C++ KWin compositor effect plugin that intercepts gestures before default handlers and allows running arbitrary commands, keyboard emulation, and DBus calls.

---

## Setup & Installation

### 1. Install Packages
On Arch Linux / CachyOS:

```bash
sudo pacman -S inputactions-kwin inputactions-kwin-alpm-hook
```

> [!IMPORTANT]
> Always install `inputactions-kwin-alpm-hook`. Because KWin C++ effect plugins enforce an exact binary ABI match (`EffectPluginFactory6.x.y`), any system update that upgrades `kwin` will break `inputactions-kwin` unless it is updated or rebuilt (see [KWin Version Upgrades](#1-kwin-version-upgrades--boot-failures)).

### 2. Install Configuration
Copy [`config.yaml`](config.yaml) to `~/.config/inputactions/config.yaml`:

```bash
mkdir -p ~/.config/inputactions
cp config.yaml ~/.config/inputactions/config.yaml
```

### 3. Enable Plugin in KWin
Activate the effect in your running KWin session:

```bash
qdbus6 org.kde.KWin /Effects org.kde.kwin.Effects.loadEffect kwin_gestures
```

Persist the plugin across reboots:

```bash
kwriteconfig6 --file kwinrc --group Plugins --key kwin_gesturesEnabled true
```

### 4. Enable Tap Detection Permissions (`udev`)
Swipe gestures are handled natively by KWin, but multi-finger **tap** detection requires raw `libevdev` event access. Grant user access to touchpad input devices:

```bash
echo 'ENV{ID_INPUT_TOUCHPAD}=="1", TAG+="uaccess"' | sudo tee /etc/udev/rules.d/71-touchpad.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### 5. Reload & Verify
To reload changes or check for syntax issues:

```bash
inputactions config reload
inputactions config issues
```

> [!NOTE]
> `inputactions config reload` requires the KWin effect (`kwin_gestures`) to already be loaded. If you receive `Failed to connect to the InputActions DBus interface`, load the effect first via step 3.

---

## Nuances & Technical Details

### 1. KWin Version Upgrades & Boot Failures
KWin C++ effect plugins bind directly to the internal KWin factory symbol (e.g. `org.kde.kwin.EffectPluginFactory6.8.1`).
- **Symptom**: Gestures stop working after rebooting following a system upgrade. Running:
  ```bash
  journalctl --user -b -u plasma-kwin_wayland.service | grep kwin_gestures
  ```
  shows:
  ```text
  Effect "kwin_gestures" was built for a different version of KWin and needs to be rebuilt, expected "org.kde.kwin.EffectPluginFactory6.8.1" got "org.kde.kwin.EffectPluginFactory6.8.0"
  ```
- **Fix**: Update or rebuild `inputactions-kwin`:
  ```bash
  sudo pacman -Sy inputactions-kwin
  # or rebuild via AUR if the binary repository is lagging:
  yay -S --rebuild inputactions-kwin
  ```
- The `inputactions-kwin-alpm-hook` package triggers a notification upon any `kwin` update reminding you if a rebuild is necessary.

### 2. Multi-Monitor Brightness Isolation
- **The Problem**: Standard keyboard shortcuts (`keyboard: [ brightnessup ]` / `[ brightnessdown ]`) trigger PowerDevil's global brightness logic. In KDE Plasma 6, PowerDevil broadcasts brightness step commands to **all** connected displays (including external monitors supporting DDC/CI or software brightness).
- **The Solution**: The configuration in `config.yaml` intercepts horizontal swipes and executes a `busctl` command targeting KDE's PowerDevil DBus service directly:
  1. Inspects `org.kde.ScreenBrightness.DisplaysDBusNames` to find the display with `IsInternal == true` (typically `display0` / `eDP-1`).
  2. Queries current `Brightness` and `MaxBrightness`.
  3. Increments/decrements in 5% steps (`±500` out of `10000`).
  4. Calls `org.kde.ScreenBrightness.Display.SetBrightness iu <val> 0`. Passing `flags = 0` (`ShowIndicator`) triggers the native KDE Brightness OSD slider while leaving external displays untouched.

### 3. KWin Native Gesture Conflicts
KDE Plasma natively reserves 4-finger vertical swipes for *Overview* and *Desktop Grid*.
- `inputactions-kwin` consumes gestures when matched.
- If conflicts arise or you prefer preventing native handlers entirely, disable *Overview* / *Desktop Grid* in:
  **System Settings** ➔ **Window Management** ➔ **Desktop Effects** (you can still bind them to keyboard shortcuts).

---

## Credits & Disclaimers

- **AI Assistance**: Documented and implemented with assistance from Antigravity (Google DeepMind) by analyzing KWin Wayland gesture architectures and configuring InputActions.
- **Disclaimer**: System configurations, hardware udev permissions, and custom display timings interact directly with hardware drivers and compositor components. Experimental use only; use at your own discretion.
