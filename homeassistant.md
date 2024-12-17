# Install HomeAssistant Core on Android phone with root.

'uv' package removed from original 'core' project. 'uv' is available to install in Termux now (pkg install uv), but 'pip install uv' doesn't work.

Xiaomi Mi11 rooted with Magisk
https://droidwin.com/how-to-root-xiaomi-eu-rom-via-magisk/

## Steps
- Install Termux, Termux:Boot via F-Droid
- Install Automate and set "turn wifi hotspot on boot" (allow run Automation on boot in Automation settings and allow all permissions for Automation)
- In Termux:
``` sh
pkg update
pkg upgrade
termux-wake-lock
termux-setup-storage
pkg install openssh
passwd
pkg install uv tsu ffmpeg python libxml2 libxslt pkg-config python-lxml libffi libjpeg-turbo python-numpy patchelf ninja screen git libpng tur-repo git python3 rust binutils-is-llvm
```

Hass core version: 2025.1.0.dev0
```
python3 -m venv venv
source venv/bin/activate
git clone -b without-uv --single-branch https://github.com/ondyn/hass-core.git
cd hass-core
python -m script.translations develop --all
pip install .
cd ..
sudo hass -v
```
Instead of 'pip install .' you can try 'uv pip install .' Will be much faster, but I faced some errors.

## Install Tailscale VPN into Termux
Update to latest version
```
mkdir vpn
cd vpn
curl -fsSL https://pkgs.tailscale.com/stable/tailscale_1.78.1_arm64.tgz | tar xzv
https://pkgs.tailscale.com/stable/tailscale_1.76.1_arm64.tgz
mv tailscale_1.78.1_arm64/{tailscale,tailscaled} . && chmod 700 {tailscale,tailscaled} && rm -r tailscale_1.78.1_arm64
```

## Setup .termux/boot scripts for VPN, SSH and HASS
vpn script
```
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
sudo ~/vpn/tailscaled -tun userspace-networking --state=$PREFIX/var/lib/tailscale/tailscaled.state -socket $PREFIX/var/run/tailscale/tailscaled.sock
```
ssh script
```
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
sshd
```
hass script
```
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
echo "Termux boot hass: starting Hass script in screen..."
screen -dmS hass sh /data/data/com.termux/files/home/scripts/hass.sh
echo "Termux boot hass: Done"
```
scripts/hass.sh

note: ESP is connected via mobiles Wifi hotspot, but ESP client IP is changing each switch on/off of hotspot, this script is changing IP address of ESP client in HASS config, to be always visible in HASS
```
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
IP=$(sudo ifconfig wlan1 | grep "inet " | awk -F'[: ]+' '{ print $3 }')
echo "${IP}"

sudo python /data/data/com.termux/files/home/scripts/update_ip.py ${IP} /data/data/com.termux/files/home/.suroot/.homeassistant/.storage/core.config_entries
. "/data/data/com.termux/files/home/.venv/bin/activate"
sudo hass
```
scripts/update_ip.py
```
import os
import json
import shutil
import sys

try:
    # Path to the file to modify
    # file_path = "/data/data/com.termux/files/home/.suroot/.homeassistant/.storage/core.config_entries"
    file_path = sys.argv[2]

    # Get the IP address of the wlan0 interface
    ip_address = sys.argv[1]
    # subprocess.call('tsu ifconfig wlan0 | grep "inet " | awk -F"[: ]+" "{ print $3 }"').decode("utf-8")

    parts = ip_address.split(".")
    parts[-1] = "7"
    ip_address = ".".join(parts)

    # Backup the original file
    backup_path = "core.config_entries.bak"
    if os.path.exists(file_path):
        shutil.copy(file_path, backup_path)

    # Open the file for reading and writing
    with open(file_path, "r+") as file:
        # Load the file contents as a JSON object
        contents = json.load(file)

        # Replace the old text with the new text
        for entry in contents['data']['entries']:
            if entry['entry_id'] == '39bb02abde1417f5f7a5efb2f474bb1c':
                entry['data']['host'] = ip_address
        file.seek(0)
        file.truncate()
        json.dump(contents, file, indent=2)

    print("File updated successfully!")
except Exception as e:
    print("Error updating IP!")
    print(e)
```
