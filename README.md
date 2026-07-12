# WSL_config
commands and notes to customize WSL ubuntu on win 11

## PowerShell

#### Check current version
```powershell
$PSVersionTable.PSVersion
```

#### Check latest version
```powershell
winget show --id Microsoft.PowerShell --exact
```

#### Install latest version
```powershell
winget install --id Microsoft.PowerShell --exact
```

---

## WSL

#### Check current version in PowerShell
```powershell
wsl --version
```

#### Update WSL
```powershell
wsl --update
```

#### Set WSL 2 as the default for new distros
```powershell
wsl --set-default-version 2
```

```powershell
wsl --shutdown
```

## Ubuntu

### Check distro version from PowerShell
```powershell
wsl --list --verbose
```
or

```powershell
wsl -l -v
```

The displayed distro name may not include its Ubuntu release version.

### Check version from inside Ubuntu
```bash
lsb_release -a
```


### Upgrade the Ubuntu release
Back up important files first. Update installed packages before starting the release upgrade:

```bash
sudo apt update
sudo apt full-upgrade
sudo do-release-upgrade
```


### Uninstall a distro
This permanently deletes the distro and all of its files. Confirm the exact name first:

```powershell
wsl --list
```

```powershell
wsl --unregister <DistroName>
```

### Install a distro

#### Directly

Requires WSL 2.4.10 or later.

##### Download the WSL image
```powershell
$url = "https://releases.ubuntu.com/resolute/ubuntu-26.04-wsl-amd64.wsl"
$image = "$env:USERPROFILE\Downloads\ubuntu-26.04-wsl-amd64.wsl"
Invoke-WebRequest -Uri $url -OutFile $image
```

##### Install without the Store
```powershell
wsl --install --from-file $image
```

##### Check the installed distro name and set it as default
```powershell
wsl --list --verbose
```

```powershell
wsl --set-default <DistroName>
```

#### MS Store
```powershell
wsl --install -d Ubuntu-26.04
```

## Utilities

### Audio test

Install the PulseAudio client and a test sound, then play it through WSLg:

```bash
sudo apt install -y pulseaudio-utils sound-theme-freedesktop
paplay /usr/share/sounds/freedesktop/stereo/complete.oga
```

Hearing the sound confirms that WSLg audio is working.

### Graphics diagnostic

```bash
sudo apt install -y mesa-utils
```

## Graphics renderer

WSLg uses the Windows host GPU driver. Do not install a Linux NVIDIA, AMD, or Intel GPU driver inside Ubuntu for WSLg graphics.

### 1. Update and restart WSL

Run these commands from PowerShell or Command Prompt:

```powershell
wsl --update
wsl --shutdown
```

`wsl --shutdown` has two hyphens. `wsl shutdown` instead tries to run Linux's `shutdown` command inside the distro.

### 2. Check the default OpenGL renderer

Open a new Ubuntu shell and run:

```bash
glxinfo -B | grep -E 'Device:|Accelerated:|OpenGL renderer'
```

A hardware-backed WSLg result contains `D3D12`, the GPU name, and `Accelerated: yes`. `llvmpipe` means Mesa is using its CPU fallback.

### 3. Make Mesa prefer D3D12 when it falls back to llvmpipe

If the previous command reports `llvmpipe`, test the D3D12 driver explicitly:

```bash
GALLIUM_DRIVER=d3d12 glxinfo -B | grep -E 'Device:|Accelerated:|OpenGL renderer'
```

If this shows `D3D12` and `Accelerated: yes`, make that preference persistent for applications launched from new Ubuntu login shells:

```bash
printf '%s\n' 'export GALLIUM_DRIVER=d3d12' | sudo tee /etc/profile.d/wslg-d3d12.sh > /dev/null
```

Restart WSL from PowerShell, open Ubuntu again, and repeat the default renderer check:

```powershell
wsl --shutdown
```

This preference applies to Mesa/Gallium applications. It cannot force GPU acceleration for applications that intentionally use CPU rendering or have no GPU backend.

If explicitly selecting D3D12 also fails, update the Windows GPU driver, restart Windows, and repeat the WSL update and renderer check. Do not override `DISPLAY` or `WAYLAND_DISPLAY`, and do not use a separate X server. See Microsoft's [WSLg guidance](https://learn.microsoft.com/en-us/windows/wsl/tutorials/gui-apps).

### 4. Check an NVIDIA GPU (NVIDIA systems only)

```bash
nvidia-smi
```

This checks NVIDIA GPU and compute visibility. Use the OpenGL renderer check above for Intel, AMD, and general GUI-rendering verification.

## Firefox
not installed by default
Use Mozilla's APT repository to install Firefox as a DEB package instead of Snap. (snap firefox isnt meant for wsl)

### Check Firefox from inside Ubuntu
```bash
which firefox
```

### Why `sudo apt install firefox` alone pulls in Snap

On stock Ubuntu, the `firefox` APT package is a thin transitional package that depends on `snapd` and installs the real browser as a snap on first run. This happens even if Firefox was never manually installed, because `snapd` ships pre-installed on Ubuntu images. Snap Firefox does not work well under WSLg. Adding Mozilla's PPA below with a pinning rule (`Pin-Priority: 1000`) makes `apt install firefox` resolve to Mozilla's real DEB package instead, so the trap is avoided entirely rather than fixed after the fact.

### Optionally remove an existing Snap installation
```bash
sudo snap remove firefox
```

### Add Mozilla's official APT repository
```bash
sudo apt update
sudo apt install -y wget gpg
sudo install -d -m 0755 /etc/apt/keyrings
wget -q https://packages.mozilla.org/apt/repo-signing-key.gpg -O- | sudo tee /etc/apt/keyrings/packages.mozilla.org.asc > /dev/null
gpg -n -q --import --import-options import-show /etc/apt/keyrings/packages.mozilla.org.asc | awk '/pub/{getline; gsub(/^ +| +$/, "", $0); if($0 == "35BAA0B33E9EB396F59CA838C0BA5CE6DC6315A3") print "Key fingerprint matches (" $0 ")."; else print "Key fingerprint verification failed (" $0 ")."}'
```

For Ubuntu 26.04 (Resolute) and later:

```bash
sudo tee /etc/apt/sources.list.d/mozilla.sources > /dev/null << 'EOF'
Types: deb
URIs: https://packages.mozilla.org/apt
Suites: mozilla
Components: main
Signed-By: /etc/apt/keyrings/packages.mozilla.org.asc
EOF

sudo tee /etc/apt/preferences.d/mozilla > /dev/null << 'EOF'
Package: *
Pin: origin packages.mozilla.org
Pin-Priority: 1000
EOF

sudo apt update
sudo apt install -y firefox
```

Mozilla's [Linux installation guide](https://support.mozilla.org/en-US/kb/install-firefox-linux) has the repository instructions for older Ubuntu releases.

### Enable Firefox hardware compositing on WSLg

After the default OpenGL renderer check reports accelerated D3D12, Firefox may still incorrectly report `WebRender (Software)` in `about:support`. Enable its WSLg workaround:

1. Open `about:config` in Firefox and accept the warning.
2. Set the Boolean preference `gfx.webrender.all` to `true`.
3. Fully quit and reopen Firefox normally.

### Verify Firefox hardware compositing

Open `about:support`, then find **Compositing** under **Graphics**:

- `WebRender` confirms Firefox hardware compositing is enabled.
- `WebRender (Software)` means Firefox is using the CPU fallback.

Firefox can still print repeated `libEGL` warnings and `Couldn't sanitize GL_RENDERER "D3D12 (...)"` at startup. On affected WSLg systems, these messages are Firefox's D3D12 detection issue; use the **Compositing** value as the authoritative check.

### Understanding the `libEGL warning` / `MESA-LOADER` startup messages

Firefox (and other Mesa-based GUI apps) can print these two lines on every launch, once per process:

```
libEGL warning: failed to get driver name for fd -1
libEGL warning: MESA-LOADER: failed to retrieve device information
```

**Root cause:** Mesa's EGL loader always probes `/dev/dri/renderD*` (the standard Linux DRM render node) before falling back to another backend. WSLg has no such device — GPU access is exposed only through `/dev/dxg`, a Windows-specific WDDM device that Mesa reaches through its separate `d3d12`/dxcore Gallium driver. The DRM probe fails as designed, Mesa logs it, then correctly falls back to `d3d12`. `glxinfo -B` and Firefox's `about:support` → Compositing (`WebRender`, not `WebRender (Software)`) both confirm hardware acceleration is active despite the warning.

**Tested and ruled out** (do not attempt these as fixes):
- `MESA_LOADER_DRIVER_OVERRIDE=d3d12` — silences these two lines but replaces them with `libEGL warning: egl: failed to create dri2 screen`, i.e. trades one warning for another without fixing anything.
- `EGL_PLATFORM=surfaceless`, `MOZ_ENABLE_WAYLAND=1` — no effect; the DRM probe runs before these are consulted.
- Symlinking `/dev/dxg` to `/dev/dri/renderD128` — `/dev/dxg` speaks the WDDM/dxcore protocol, not the DRM ioctl protocol a render node is expected to support. A bare symlink makes `open()` succeed but does not make the device behave like a DRM node, and risks breaking the working `d3d12` fallback rather than fixing the probe.

There is currently no known way to suppress this specific probe-and-fallback message on stock WSLg without patching Mesa. It is safe to ignore as long as the **Compositing** check above reports `WebRender`.

If Firefox reports audio warnings, first verify WSLg audio using the audio test above. A successful `paplay` test confirms the WSLg audio path is working.

---

## Isolated dev environment (Hyper-V VM)

WSLg is good for lightweight GUI testing, but it shares the Windows kernel and isn't a real isolation boundary. For a dev environment with VS Code, a VPN client, and other utilities that need their own independent network stack, a Hyper-V VM is a better fit than WSL. Hyper-V is already enabled on this machine (it underlies WSL2), so no new hypervisor install is required.

### 1. Confirm Hyper-V is available

From an elevated PowerShell prompt:

```powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
Get-Service vmms
```

`vmms` (Hyper-V Virtual Machine Management) should show `Running`.

### 2. Get an Ubuntu ISO

Prefer **Ubuntu Server** over Desktop if a GUI inside the VM isn't required — VS Code can connect remotely instead. Download from Ubuntu's official releases page and note the local file path.

### 3. Create an isolated virtual switch

Creating a dedicated switch keeps the VM's network (and any VPN client running inside it) separate from the host's normal traffic:

```powershell
New-VMSwitch -Name "DevVM-Switch" -SwitchType Internal
```

Use `SwitchType External` instead if the VM needs direct LAN/internet access under its own identity; use `Internal` for host-only isolation with NAT added separately if needed.

### 4. Create the VM

```powershell
New-VM -Name "DevBox" -Generation 2 -MemoryStartupBytes 4GB -NewVHDPath "D:\VMs\DevBox\DevBox.vhdx" -NewVHDSizeBytes 60GB -SwitchName "DevVM-Switch"
Set-VMFirmware -VMName "DevBox" -EnableSecureBoot On -SecureBootTemplate "MicrosoftUEFICertificateAuthority"
Set-VMDvdDrive -VMName "DevBox" -Path "<path-to-ubuntu.iso>"
Set-VMProcessor -VMName "DevBox" -Count 4
```

Adjust memory/CPU/disk size to taste. `MicrosoftUEFICertificateAuthority` is required for Linux Secure Boot support on Generation 2 VMs.

### 5. Install Ubuntu

```powershell
Start-VM -Name "DevBox"
vmconnect.exe localhost "DevBox"
```

Walk through the Ubuntu installer in the VM Connect window as normal.

### 6. Snapshot immediately after base setup

Once Ubuntu, VS Code Server, VPN client, and other utilities are installed and working, take a checkpoint so this known-good state can be restored instantly:

```powershell
Checkpoint-VM -Name "DevBox" -SnapshotName "Base-Setup-Complete"
```

Restore with:

```powershell
Restore-VMSnapshot -VMName "DevBox" -Name "Base-Setup-Complete" -Confirm:$false
```

### 7. Connect VS Code to the VM

Install the **Remote - SSH** extension in Windows-side VS Code, then connect to the VM's IP over SSH. This keeps the editor UI on Windows while all code execution, VPN traffic, and tooling stay isolated inside the VM.

### Notes

- A VM is a genuine isolation boundary (separate kernel); WSL2 is not (shared kernel with Windows via a lightweight VM, but tightly integrated by design).
- Checkpoints are the fast undo button here — take one before installing anything risky or experimental.
- If the VPN client requires exclusive routing control, the `Internal` switch type avoids fighting with the host's own network adapters.
