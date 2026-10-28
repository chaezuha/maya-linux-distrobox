# Maya 2027 on Linux via Distrobox (Fedora Silverblue, Arch, Ubuntu, and others)

How to run Autodesk Maya 2027 inside a Rocky Linux 9 distrobox, with Maya kept in its own home folder. Tested on Fedora Silverblue (GNOME, Wayland) with an AMD Radeon iGPU (Mesa/radeonsi), Maya 2027.2 and a Rocky 9.8 container, Oct 2026. Everything inside the container works the same on any host distro; [Other host distros](#other-host-distros) covers what differs on the host.

The commands use `$HOME`, `$USER` and `$(id -u)`, so they work for any user. Values you might want to change:

| Value | Used in this guide | When to change it |
|---|---|---|
| Container name | `maya` | Pick any name; use it consistently |
| Maya home folder | `~/distrobox/maya-home` | Any folder you like |
| Maya install folder | `/usr/autodesk/maya2027` | Other Maya versions (`ls /usr/autodesk`) |
| Product key / version | `657S1` / `2027.0.0.F` | Other versions (read from `AdskLicensingInstHelper list`) |
| NVIDIA GPU | not covered by testing | Add `--nvidia` when creating the container |

> Maya officially supports only RHEL/Rocky 8.10 and 9.7. Other distros are unsupported, hence the container. Autodesk won't support this setup.

---

## Before you start

Each code block starts with `# On the host` (your normal terminal) or `# Inside the container` (run `distrobox enter maya` first; `exit` gets you out).

Every block can be pasted whole unless the line above it says otherwise. Paste into a terminal with Ctrl+Shift+V. When `sudo` asks for your password, nothing shows as you type. Running `sudo -v` before pasting a block keeps the password prompt from eating the pasted lines.

---

## Why it's set up this way

Terms used below:
- **Container / distrobox:** a separate Linux system (here Rocky) running on top of yours, sharing your display, user and files.
- **XWayland:** runs older X11 apps on a Wayland desktop. Maya runs through it.
- **X cookie:** a key a program needs to draw on your display. A new one is made at every login.
- **systemd:** starts background services, such as Maya's licensing service.

| Problem | Cause | Fix used here |
|---|---|---|
| Maya won't run natively on your distro | Only RHEL/Rocky supported | Rocky 9 distrobox |
| Licensing service won't run | `adsklicensing` is a systemd service | `--init` container (systemd inside) |
| Black viewport (Maya 2025+) | Broken on native Wayland | Force XWayland: `QT_QPA_PLATFORM=xcb` |
| `IDSDK 3029: display server not supported` | Identity Manager refuses Wayland | `unset WAYLAND_DISPLAY`, `XDG_SESSION_TYPE=x11` |
| `Authorization required ... unable to open display :0` | The container can't find your X cookie, and the host's copy is filed under the host's hostname | Add the cookie inside the container + `xhost +si:localuser` (step 4) |
| Dotfiles cluttering real home | Distrobox shares `$HOME` by default | `--home` custom home folder |

`--home` keeps configs out of your real home, but your real home is **still mounted** in the container. It's tidiness, not isolation.

---

## 1. Host prerequisites

You need distrobox plus podman (or docker). Some distros, such as Bazzite, already include it.

**Run only the line for your distro.**
```bash
# On the host
rpm-ostree install distrobox           # Fedora Silverblue / Atomic desktops (then reboot)
sudo dnf install distrobox podman      # Fedora Workstation
sudo pacman -S distrobox podman        # Arch / CachyOS
sudo apt install distrobox podman      # Debian / Ubuntu / Mint
```

---

## Other host distros

From step 2 on, everything happens inside the Rocky container and is the same on any host. These host-side parts can differ:

| Part | Fedora Silverblue (tested) | Other distros |
|---|---|---|
| Installing distrobox | `rpm-ostree install distrobox` | The distro's package manager (step 1) |
| Your real home path | `/var/home/$USER` | Usually `/home/$USER`. Only matters for the `distrobox-export --bin` path in step 10 |
| X cookie file | GNOME: `/run/user/<uid>/.mutter-Xwaylandauth.*` | Other desktops (e.g. KDE) name it differently. Use whatever `echo $XAUTHORITY` prints on the host in step 4. `maya-launch` also checks KDE's `xauth_*` and the host's `~/.Xauthority`, or you can point it at the file (step 9) |
| X11 session hosts | n/a (Wayland only) | Hiding Wayland is unnecessary but harmless. The cookie file is often `~/.Xauthority` in your real home |
| NVIDIA | not tested | Use `--nvidia`. A forum tester on Arch/CachyOS also had to install the NVIDIA container toolkit and regenerate its CDI config before the container could see the GPU. Bazzite ships this preconfigured. On Btrfs, they also needed `fuse-overlayfs` for podman |
| Debian / Ubuntu / Mint | n/a | The same tester reported Mint blocking the container's X/Wayland access. They needed `xhost` and `QT_QPA_PLATFORM=xcb` (both already in `maya-launch`); expect more friction |

If you get it working on another distro or desktop, please open an issue with what you changed.

---

## 2. Create the container

Creates the Rocky container with systemd and a separate home folder for Maya. The last line puts you inside it.

```bash
# On the host
distrobox create \
  --name maya \
  --image docker.io/rockylinux/rockylinux:9-ubi-init \
  --init \
  --home ~/distrobox/maya-home

distrobox enter maya
```

- `9-ubi-init` includes systemd. Plain `rockylinux:9` doesn't.
- The `--home` folder is created if it doesn't exist.
- Inside the container, `~` is the Maya home (on Silverblue, `/var/home/<you>/distrobox/maya-home`), not your real home. Use full paths when you mean your real home.
- NVIDIA users: add `--nvidia` to share the host's driver. AMD and Intel need nothing extra.

---

## 3. Dependencies

Maya's required libraries plus a few extras that turned out to be missing. This takes a few minutes.

```bash
# Inside the container
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb
sudo dnf upgrade -y

# Autodesk's documented list + extras that turned out to be needed
sudo dnf install -y \
  audiofile compat-openssl11 gtk2 libffi libmng libglvnd-opengl libpng15 libpq libXp \
  mesa-libGLU mesa-dri-drivers glx-utils \
  xcb-util-cursor xcb-util-wm xcb-util-image xcb-util-keysyms xcb-util-renderutil \
  xorg-x11-fonts-ISO8859-1-75dpi xorg-x11-fonts-ISO8859-1-100dpi \
  firefox glibc-langpack-en \
  webkit2gtk3 libXpm libxkbfile libxkbcommon-x11 \
  xdg-utils xorg-x11-xauth

# xhost (package name varies; this finds it)
sudo dnf install -y "$(dnf provides -q '*/bin/xhost' | awk 'NR==1{print $1}')"
```

Why the extras:
- `webkit2gtk3`: Identity Manager login fails without it.
- `libXpm`, `libxkbfile`, `libxkbcommon-x11`: missing at first launch (`error while loading shared libraries`).
- `firefox`: the sign-in needs a browser inside the container.
- `xdg-utils`: needed to set a default browser.

Once Maya is installed (step 5), you can check for missing libraries. Maya bundles its own Qt, Python and so on, so its lib folder has to be on the path or you get false alarms:

```bash
# Inside the container
ldd /usr/autodesk/maya2027/bin/maya.bin | grep "not found"
LD_LIBRARY_PATH=/usr/autodesk/maya2027/lib find /usr/autodesk/maya2027/lib -name '*.so*' -exec ldd {} \; 2>/dev/null | grep "not found" | sort -u
```

Leftovers like Graphviz (`libcgraph`, `libgvc`), `libpng12` and `libgs.so.8` are only used by optional plugins. Maya starts without them.

---

## 4. Display access (X cookie)

Programs in the container need your desktop's X cookie to open windows, and the container can't find it on its own. You copy its key into a Maya-only file, `~/.maya-xauthority`.

First find the cookie file:

```bash
# On the host
echo $XAUTHORITY      # GNOME: /run/user/1000/.mutter-Xwaylandauth.AB12CD
hostname
```

**Edit first:** set `COOKIE` to the path the host printed.
```bash
# Inside the container
COOKIE=/run/user/$(id -u)/.mutter-Xwaylandauth.AB12CD    # your path; if missing, prefix /run/host
[ -r "$COOKIE" ] && xauth -i -f "$COOKIE" list || echo "Can't read $COOKIE - check the path"
echo $DISPLAY
```

`xauth list` prints one line per record, `<host>/unix:<display>  <protocol>  <key>`, for example:

```
myhost/unix:0  MIT-MAGIC-COOKIE-1  3f2a9c...
```

Pick the line whose display number matches `echo $DISPLAY` (usually `:0`). If several match, use the one with your host's name from `hostname`. GNOME often lists the same key twice, once with your hostname and once as `#ffff#...`. Either works.

Then add that key and check it works before saving it:

**Edit first:** set `KEY` to the key you picked.
```bash
# Inside the container
KEY=3f2a9c...                                   # third column of the line you chose
NEW=$(mktemp ~/.maya-xauthority.XXXXXX)
xauth -f "$NEW" add "$DISPLAY" MIT-MAGIC-COOKIE-1 "$KEY"
if XAUTHORITY="$NEW" xhost >/dev/null; then
  mv "$NEW" ~/.maya-xauthority && echo "Cookie works"
else
  rm -f "$NEW"; echo "That key didn't work - try another line"
fi

export XAUTHORITY=~/.maya-xauthority
xhost +si:localuser:$USER        # prints "...being added"
glxinfo -B | grep -i renderer
```

`glxinfo` should show your GPU, e.g. `AMD ... (radeonsi ...)` on the tested machine. Intel and NVIDIA show their own names. `llvmpipe` means software rendering, so something is wrong.

The cookie changes every login and the `xhost` permission resets on logout. Redo this step after logging in, or let `maya-launch` (step 9) do it for you.

---

## 5. Install Maya 2027 from the RPMs

Download the Linux installer from your Autodesk account, extract it somewhere under the Maya home (e.g. `~/installer/`), and `cd` into its `Packages` folder. Run everything in this step from there, in the same terminal.

The 2027 layout differs from older docs: the licensing packages are in subfolders, and there's no `adlmapps` RPM. Skip `FlowMaya/`, `FlowService/` and `flowretopology/` unless you use Autodesk Flow.

Licensing, sign-in (Identity Manager) and the crash reporter. Two Identity Manager versions ship. Use the newer one in `AdskIdentityManager/` (1.18 in 2027.2), not the copy in `Licensing/`.

```bash
# Inside the container, in the installer's Packages folder
sudo dnf install -y ./Licensing/adsklicensing*.rpm \
                    ./Licensing/adskflexnetserverIPV6*.rpm \
                    ./AdskIdentityManager/adskidentitymanager*.rpm
sudo systemctl enable --now adsklicensing
systemctl status adsklicensing --no-pager     # should be active (running)
sudo dnf install -y ./CER/autodesk.cer.service-*.rpm
```

The ADP SDK is a zip, not an RPM. Its install folder is listed in `AdpSdk/pkg.adp-desktop-sdk.xml`:

```bash
# Inside the container, in the installer's Packages folder
ADP_DIR=$(grep -o 'installLocation="[^"]*"' AdpSdk/pkg.adp-desktop-sdk.xml | cut -d'"' -f2)
echo "$ADP_DIR"    # e.g. /opt/Autodesk/AdpDesktopSDK/6.4.14.18
```

That should print exactly one path. If it prints nothing or several lines, open the XML, find the SDK's `installLocation`, and set it yourself (`ADP_DIR=/opt/Autodesk/...`). Then install it:

```bash
# Inside the container, in the installer's Packages folder
sudo chmod +x AdpSdk/InstallCustomAction
sudo ./AdpSdk/InstallCustomAction action=pre
sudo mkdir -p "$ADP_DIR"
sudo unzip -o AdpSdk/adp-desktop-sdk.zip -d "$ADP_DIR"
sudo ./AdpSdk/InstallCustomAction action=post
```

Maya itself:

```bash
# Inside the container, in the installer's Packages folder
sudo dnf install -y ./Maya2027_64-*.rpm
```

Optional plugins:

**Edit first:** remove any you don't want.
```bash
# Inside the container, in the installer's Packages folder
sudo dnf install -y ./Bifrost2027-*.rpm ./MayaUSD2027-*.rpm ./LookdevX-*.rpm ./AdobeSubstance3DforMaya-*.rpm
```

Arnold (MtoA) isn't installed automatically with the RPMs, unlike on macOS/Windows. Its installer asks a few questions:

```bash
# Inside the container, in the installer's Packages folder
sudo ./unix_installer.sh
```

### Fonts the licensing installer couldn't install

The licensing RPM tries to write to `/usr/local/share/fonts`, which is read-only in the container. Put the fonts in the Maya home instead:

```bash
# Inside the container
mkdir -p ~/.local/share/fonts/ArtifaktElement
cp /opt/Autodesk/AdskLicensing/*/AdskLicensingService/res/fonts/* ~/.local/share/fonts/ArtifaktElement/
fc-cache -f ~/.local/share/fonts
```

---

## 6. Set the license method (required with the RPM install)

The RPM install registers Maya without a license method, which causes `ADLSDK_STATUS_LICENSE_CHECKOUT_ERROR`. This sets it to sign-in (user) licensing. `657S1` and `2027.0.0.F` are Maya 2027's product key and version. For another version, use the `def_prod_key` and `def_prod_ver` from the `MAYA` entry in the `list` output.

```bash
# Inside the container
# Use Current/ explicitly; a * glob matches two folders and breaks the command
/opt/Autodesk/AdskLicensing/Current/helper/AdskLicensingInstHelper list

sudo /opt/Autodesk/AdskLicensing/Current/helper/AdskLicensingInstHelper change \
  -pk 657S1 -pv 2027.0.0.F -lm USER

/opt/Autodesk/AdskLicensing/Current/helper/AdskLicensingInstHelper list   # should now show "lic_method": 4
sudo systemctl restart adsklicensing
```

---

## 7. Default browser + Identity Manager handler

Signing in opens a browser inside the container. Without a default browser you get `IDSDK 3074: No default web browser is set`. These settings only affect the Maya home, not your host browser.

```bash
# Inside the container
mkdir -p ~/.config
cat > ~/.config/mimeapps.list <<'EOF'
[Default Applications]
x-scheme-handler/http=firefox.desktop
x-scheme-handler/https=firefox.desktop
text/html=firefox.desktop
EOF
xdg-settings set default-web-browser firefox.desktop 2>/dev/null
xdg-mime query default x-scheme-handler/https    # firefox.desktop

/opt/Autodesk/AdskIdentityManager/Current/AdskIdentityManager --register
```

---

## 8. (Optional) Legacy X font warnings

Errors like `// Error: Failed trying to load font : -*-helvetica-bold-r-normal-...` are cosmetic and safe to ignore. They happen because XWayland on the host looks up old-style X fonts and can't see the container's. To silence them, copy the fonts into the Maya home, which the host can see:

```bash
# Inside the container
mkdir -p ~/x11fonts
cp -r /usr/share/X11/fonts/75dpi /usr/share/X11/fonts/100dpi ~/x11fonts/
for d in ~/x11fonts/*; do mkfontdir "$d"; done
xset +fp "$HOME/x11fonts/75dpi,$HOME/x11fonts/100dpi"
xset fp rehash
xset q | grep -A2 "Font Path"    # your x11fonts folders should be listed
```

- Both folders go in one `xset +fp` argument, separated by a comma. Separating them with a space fails.
- `bad font path element` means a folder is missing its `fonts.dir` index. Rerun the `mkfontdir` loop.
- `mkfontdir` and `xset` come from `xorg-x11-font-utils` and `xorg-x11-server-utils`. If the font folders aren't at that path, run `rpm -ql xorg-x11-fonts-ISO8859-1-75dpi | head`.

The font path resets on logout. `maya-launch` re-applies it whenever `~/x11fonts` exists.

---

## 9. The launch script: `maya-launch`

Every launch needs the step 4 cookie refresh plus a few settings. [`maya-launch`](maya-launch) does all of it in one command, and it's what the step 10 launchers run.

To launch by hand instead, do step 4, then:

```bash
# Inside the container, after doing step 4 (once per login)
unset WAYLAND_DISPLAY
export XDG_SESSION_TYPE=x11 QT_QPA_PLATFORM=xcb LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
export XAUTHORITY=~/.maya-xauthority
/usr/autodesk/maya2027/bin/maya
```

- `unset WAYLAND_DISPLAY` and `XDG_SESSION_TYPE=x11` hide Wayland from Maya and the Identity Manager, which avoids `IDSDK 3029`.
- `QT_QPA_PLATFORM=xcb` makes Maya's interface draw through X11/XWayland ("xcb" is Qt's X11 backend). This avoids the black viewport.
- `LANG` and `LC_ALL` force an English locale regardless of your host's settings.
- If you did step 8, rerun its two `xset` lines after each login too.

Install the script:

```bash
# Inside the container
curl -fLo ~/maya-launch https://raw.githubusercontent.com/chaezuha/maya-linux-distrobox/main/maya-launch
sudo install -m 755 ~/maya-launch /usr/local/bin/maya-launch
```

The script also finds the cookie file on its own, tries each record until one works, and tells you if none does. To point it at a specific cookie file, run `XAUTHORITY=/run/host<host path> maya-launch`. For another Maya version, edit the path on its last line.

---

## 10. Launching

You can start Maya from inside the container, or export a launcher to your host. Exporting (an app-grid icon or a `maya-launch` command) is the most convenient. If you'd rather keep your host system separate from the container, skip it and launch from inside the container. It's up to you.

### From inside the container

Nothing gets added to your host. Enter the container:

```bash
# On the host
distrobox enter maya
```

Then start Maya:

```bash
# Inside the container
maya-launch
```

From a host terminal, `distrobox enter maya -- maya-launch` does both in one go.

### App-grid icon

This puts a `.desktop` file in your host's `~/.local/share/applications`.

```bash
# Inside the container
sudo cp /usr/autodesk/maya2027/desktop/Autodesk-Maya.desktop /usr/share/applications/maya2027.desktop
sudo sed -i 's|^Exec=.*|Exec=/usr/local/bin/maya-launch %F|' /usr/share/applications/maya2027.desktop
distrobox-export --app /usr/share/applications/maya2027.desktop
# undo: distrobox-export --app /usr/share/applications/maya2027.desktop --delete
```

Maya now shows up in your host's app grid. If the desktop file isn't at that path, find it with `rpm -ql $(rpm -qa | grep -i '^Maya') | grep desktop`.

### Command on the host

This exports `maya-launch` to your real home's `~/.local/bin`. It needs the full path, because `~` inside the container is the Maya home. On Silverblue and other Fedora Atomic desktops your real home is `/var/home/$USER`; on most other distros it's `/home/$USER`.

**Edit first:** if you're not on Silverblue, change `/var/home` to `/home`.
```bash
# Inside the container
distrobox-export --bin /usr/local/bin/maya-launch --export-path /var/home/$USER/.local/bin
```

When you're done, stop the container (this also stops the licensing service):

```bash
# On the host
distrobox stop maya
```

---

## 11. Uninstalling everything

Removing the container deletes Maya and everything installed in it. The Maya home folder is a normal folder on the host and is kept unless you pass `--rm-home` or delete it yourself.

**Back up any projects in the Maya home first.** Maya's default project folder is `~/maya/projects` inside the container, which is `<maya-home>/maya/projects` on the host.

First remove the launchers from step 10, if you made any. These run on the host and undo the export inside the container.

**Edit first:** keep only the lines for launchers you made, and use the same `--export-path` as in step 10.
```bash
# On the host
distrobox enter maya -- distrobox-export --app /usr/share/applications/maya2027.desktop --delete
distrobox enter maya -- distrobox-export --bin /usr/local/bin/maya-launch --export-path /var/home/$USER/.local/bin --delete
```

Then remove the container. To delete the Maya home folder too:

```bash
# On the host
distrobox stop maya
distrobox rm --rm-home maya
```

Or, to keep the Maya home folder (you can delete `~/distrobox/maya-home` later):

```bash
# On the host
distrobox stop maya
distrobox rm maya
```

Optionally, free up disk space by removing the Rocky image:

```bash
# On the host
podman rmi docker.io/rockylinux/rockylinux:9-ubi-init
```

Apart from distrobox and podman, nothing was installed on the host.

---

## Troubleshooting quick reference

| Symptom | Fix |
|---|---|
| App icon does nothing | Run `maya-launch` in a terminal inside the container to see Maya's startup and licensing errors |
| `maya-launch: no X cookie found that opens :0` | Do step 4 by hand, or point the script at your cookie: `XAUTHORITY=/run/host<path> maya-launch` |
| `maya-launch: cannot create/write/replace ...` | The Maya home isn't writable; check its permissions |
| `Authorization required, but no authorization protocol specified` | Step 4 (cookie + `xhost`) |
| `xauth: timeout in locking authority file` | `rm -f ~/.Xauthority-c ~/.Xauthority-l` (use the file name from the message) |
| `Can not load RPM file: ./adsklicensing*.rpm` | Maya 2027 keeps them in `Licensing/` and `AdskIdentityManager/` subfolders |
| `mkdir: cannot create '/usr/local/share/fonts/...': Read-only` | Copy the fonts to `~/.local/share/fonts` (step 5) |
| `No help topic for ...AdskLicensingInstHelper` | Use `Current/` instead of `*` in the path |
| `error while loading shared libraries: libXxx.so` | `dnf provides '*/libXxx.so*'`, then install it |
| `ADLSDK_STATUS_LICENSE_CHECKOUT_ERROR` | Check `/usr/tmp/MayaCLM-<date>.log` for the IDSDK code |
| ...with no `lic_method` in `list` | Step 6 (`-lm USER`) |
| ...with `IDSDK 3029: display server not supported` | Launch via `maya-launch` (it hides Wayland); then `pkill -f AdskIdentityManager; pkill -f AdskLicensingAgent` |
| ...with `IDSDK 3074: No default web browser` | Step 7 |
| Black viewport | Make sure Maya runs via `maya-launch` (`QT_QPA_PLATFORM=xcb`) |
| Helvetica font errors in Script Editor | Cosmetic. Ignore, or step 8 |
| ...still there after step 8 | Check `xset q` lists `x11fonts`; the `xset +fp` folders must be comma-separated |
| Graph Editor not redrawing until mouse leaves | Known XWayland quirk |
| "systemd shutdown inhibition lock" warnings during dnf | Harmless |

Useful logs:
- `/usr/tmp/MayaCLM-<date>.log`: license checkout errors (IDSDK codes)
- `~/.local/share/Autodesk/Identity Services/Log/IdServices.log`: sign-in / Identity Manager
- `/var/opt/Autodesk/AdskLicensingService/Log/AdskLicensingService.log`: licensing service

To check what environment the Identity Manager actually got:

```bash
# Inside the container
for p in $(pgrep -f AdskIdentityManager); do echo "== $p"; tr '\0' '\n' < /proc/$p/environ | grep -E 'WAYLAND|XDG_SESSION_TYPE|^DISPLAY|XAUTHORITY'; done
```

---

## Hardware notes

- AMD / Intel: the viewport runs through Mesa with no extra setup. On integrated GPUs (tested on a Radeon Vega iGPU) it's fine for modeling and learning, but heavy scenes and XGen will be slow. The iGPU shares system RAM, so more RAM helps.
- NVIDIA: create the container with `--nvidia`. This guide wasn't tested on NVIDIA, so expect some extra troubleshooting.
- Arnold: without a supported NVIDIA GPU, Arnold renders on the CPU. It works, but it's slow on laptop CPUs.
