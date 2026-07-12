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


## Firefox
not installed by default
Use Mozilla's APT repository to install Firefox as a DEB package instead of Snap. (snap firefox isnt meant for wsl)

### Check Firefox from inside Ubuntu
```bash
which firefox
```

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

## Graphics renderer

Assumes the Windows host GPU driver is current. WSLg uses that Windows driver, so do not install a Linux GPU driver inside Ubuntu for WSLg graphics.

### Restart WSL after updating it

```powershell
wsl --update
wsl --shutdown
```

### Verify hardware rendering from Ubuntu
```bash
sudo apt install -y mesa-utils
glxinfo -B | grep "OpenGL renderer"
```

A hardware-backed WSLg result commonly contains `D3D12` and the GPU name. A result containing `llvmpipe` indicates a software-rendering fallback.

If `llvmpipe` remains, confirm that Windows is fully updated, restart Windows, rerun the WSL restart commands above, and do not manually override `DISPLAY` or `WAYLAND_DISPLAY` or use a separate X server. See Microsoft's [WSLg guidance](https://learn.microsoft.com/en-us/windows/wsl/tutorials/gui-apps).

### Check an NVIDIA GPU (NVIDIA systems only)
```bash
nvidia-smi
```

This checks NVIDIA GPU and compute visibility; use the OpenGL-renderer check above for Intel, AMD, or general GUI-rendering verification.
