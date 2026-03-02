# Fedora Silverblue — Personal Setup Notes

This README documents the personal steps I use to set up a fresh Fedora Silverblue installation. Use at your own risk — adapt to your hardware and preferences.

## Quick overview
- Install RPM Fusion repos, install codecs and personal overrides 
- Configure Flatpak remotes and apps
- Add automatic Flatpak update timer
- Add on auto-staged updates and hardware/kernel settings

## 1) Add RPM Fusion
```bash
sudo rpm-ostree install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

systemctl reboot
```

## 2) Flatpak setup
```bash
# Add Flathub
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Reinstall existing apps from Flathub
flatpak install --reinstall flathub $(flatpak list --app-runtime=org.fedoraproject.Platform --columns=application | tail -n +1 )

# Remove Fedora remote 
flatpak remote-delete fedora

# Install my flatpaks 
flatpak install com.github.tchx84.Flatseal \
  com.mattjakeman.ExtensionManager com.prusa3d.PrusaSlicer com.ranfdev.DistroShelf \
  io.github.flattool.Warehouse io.github.kolunmi.Bazaar io.github.shiftey.Desktop \
  io.missioncenter.MissionCenter io.podman_desktop.PodmanDesktop it.mijorus.gearlever \
  org.gimp.GIMP org.inkscape.Inkscape org.libreoffice.LibreOffice \
  org.mozilla.firefox org.telegram.desktop

# Uninstall the GNOME packaged Extensions app if present
flatpak uninstall org.gnome.Extensions
```

## 3) Automatic Flatpak updates (systemd service + timer)
```bash
# Create the service unit
sudo tee /etc/systemd/system/flatpak-update.service > /dev/null <<'EOF'
[Unit]
Description=Update Flatpak apps automatically

[Service]
Type=oneshot
ExecStart=/usr/bin/flatpak update -y --noninteractive
EOF

# Create the timer unit
sudo tee /etc/systemd/system/flatpak-update.timer > /dev/null <<'EOF'
[Unit]
Description=Run Flatpak update every 24 hours
Wants=network-online.target
Requires=network-online.target
After=network-online.target

[Timer]
OnBootSec=120
OnUnitActiveSec=24h

[Install]
WantedBy=timers.target
EOF

# Reload systemd and enable the timer
sudo systemctl daemon-reload
sudo systemctl enable --now flatpak-update.timer

# Check the status to verify everything is working
sudo systemctl status flatpak-update.timer
```

## 4) System and service tweaks
```bash
# Disable NetworkManager-wait-online (if you prefer faster boot)
sudo systemctl disable NetworkManager-wait-online.service

# Ensure system clock uses UTC
sudo timedatectl set-local-rtc 0 --adjust-system-clock

# Example GNOME setting (adjust PTYXIS_PROFILE first if used)
gsettings set org.gnome.Ptyxis.Profile:/org/gnome/Ptyxis/Profiles/$PTYXIS_PROFILE/ opacity .90
```

## 5) Codecs, drivers and layered packages

Why these choices:
- `discord`: I install the native package via rpm-ostree because the distro package is better packaged for desktop integration (audio/video handling and system integration) than the Flatpak.
- `firefox`: I remove the distro `firefox` here because I prefer to run the Flatpak `org.mozilla.firefox` for sandboxing and easier updates.
- `gnome-software`: I remove `gnome-software` and related `gnome-software-rpm-ostree` in favor of the Flatpak-based app store `Bazaar` (installed earlier as `io.github.kolunmi.Bazaar`).
- `distrobox`: I prefer `distrobox` together with `podman` over `toolbox` for a easier container workflow.

```bash
rpm-ostree install \
  intel-media-driver \
  discord \
  distrobox \
  gnome-tweaks \
  podman-compose \
  podman-docker \
  gstreamer1-plugin-libav \
  gstreamer1-plugins-bad-free-extras \
  gstreamer1-plugins-bad-freeworld \
  gstreamer1-plugins-ugly \
  gstreamer1-vaapi \
  --allow-inactive

rpm-ostree override remove \
  gnome-software \
  gnome-software-rpm-ostree \
  firefox \
  firefox-langpacks \
  fdk-aac-free \
  libavcodec-free \
  libavdevice-free \
  libavfilter-free \
  libavformat-free \
  libavutil-free \
  libpostproc-free \
  libswresample-free \
  libswscale-free \
  ffmpeg-free \
  --install ffmpeg

systemctl reboot

# Allow upgrades to later Fedora versions
sudo rpm-ostree update --uninstall rpmfusion-free-release --uninstall rpmfusion-nonfree-release --install rpmfusion-free-release --install rpmfusion-nonfree-release

systemctl reboot
```

## 6) Setup automatic system updates
Edit `/etc/rpm-ostreed.conf` and set:

```ini
[Daemon]
AutomaticUpdatePolicy=stage
```

Then start/enable the rpm-ostreed automatic timer:
```bash
sudo systemctl start rpm-ostreed    # or reload
sudo systemctl enable --now rpm-ostreed-automatic.timer
```

## 7) Intel i915 kernel/module options (if applicable)
Create or edit `/etc/modprobe.d/i915.conf` with:

```text
options i915 enable_guc=2
options i915 enable_fbc=1
```

Then regenerate initramfs enabling the modprobe file so it is included:
```bash
sudo rpm-ostree initramfs --enable --arg=-I --arg=/etc/modprobe.d/i915.conf
```

## 8) Disable RAOP in PipeWire so we don't find Airplay devices
```bash
mkdir -p ~/.config/pipewire/pipewire.conf.d/
cd ~/.config/pipewire/pipewire.conf.d/
touch disable-raop.conf
# Edit disable-raop.conf and add:
# context.properties = { module.raop = false }
```

## 9) Disable GNOME donation popups
Sometimes GNOME shows occasional donation/reminder popups. To check the current
setting run:

```bash
gsettings get org.gnome.settings-daemon.plugins.housekeeping donation-reminder-enabled
```

To disable the donation reminder/popups, run:

```bash
gsettings set org.gnome.settings-daemon.plugins.housekeeping donation-reminder-enabled false
```


