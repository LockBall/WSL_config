# WSL_config
commands and notes to customize WSL ubuntu on win 11

check WSL version
`lsb_release -a`

upgrade WSL release version
`sudo do-release-upgrade`

## install firefox
there can be issues with WSL snap firefox
# Remove & purge the snap version
sudo snap remove firefox
sudo apt purge firefox -y

# Add Mozilla's official APT repo
sudo apt install -y wget gpg
wget -q https://packages.mozilla.org/apt/repo-signing-key.gpg -O- | sudo tee /etc/apt/keyrings/packages.mozilla.org.asc > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/packages.mozilla.org.asc] https://packages.mozilla.org/apt mozilla main" | sudo tee /etc/apt/sources.list.d/mozilla.list > /dev/null

# Pin it so apt prefers Mozilla's build over any Ubuntu snap-stub package
``
echo '
Package: firefox*
Pin: origin packages.mozilla.org
Pin-Priority: 1000
' | sudo tee /etc/apt/preferences.d/mozilla
``

`sudo apt update`
`sudo apt install firefox -y`

check grpahics renderer
install mesa utils
`sudo apt install mesa-utils -y`

`glxinfo | grep "OpenGL renderer"`
result of llvmpipe indicates software fallback

