# Fedora Kinoite — Personal Setup Notes

This README documents the personal steps I use to set up a fresh Fedora Kinoite installation. Use at your own risk — adapt to your hardware and preferences.

## Quick overview
- Install RPM Fusion repos, install codecs and personal overrides 
- Configure Flatpak remotes and apps
- Add on auto-staged updates and hardware/kernel settings

## 1) Add RPM Fusion repos
```bash
sudo rpm-ostree install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

## 2) Flatpak setup
```bash
# Add Flathub
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

```bash
# Reinstall existing apps from Flathub
flatpak install --reinstall flathub $(flatpak list --app-runtime=org.fedoraproject.Platform --columns=application | tail -n +1 )
```

```bash
# Remove Fedora remote 
flatpak remote-delete fedora
```

```bash
# Remove Fedora testing
flatpak remote-delete fedora-testing
```

```bash
# Install my flatpaks 
flatpak install com.github.tchx84.Flatseal \
  com.prusa3d.PrusaSlicer \
  com.ranfdev.DistroShelf \
  it.mijorus.gearlever \
  org.gimp.GIMP \
  org.inkscape.Inkscape \
  org.libreoffice.LibreOffice \
  org.mozilla.firefox \
  org.telegram.desktop \
  com.discordapp.Discord 
```

## 3) System and service tweaks
```bash
# Disable NetworkManager-wait-online (if you prefer faster boot)
sudo systemctl disable NetworkManager-wait-online.service
```

## 4) Setup automatic system updates
Edit `/etc/rpm-ostreed.conf` and set:

```ini
[Daemon]
AutomaticUpdatePolicy=stage
```

Then start/enable the rpm-ostreed automatic timer:
```bash
sudo systemctl start rpm-ostreed    # or reload
```

```bash
sudo systemctl enable --now rpm-ostreed-automatic.timer
```

## 5) Disable RAOP in PipeWire so we don't find Airplay devices
```bash
mkdir -p ~/.config/pipewire/pipewire.conf.d/
```

```bash
cd ~/.config/pipewire/pipewire.conf.d/
```

```bash
# Add this to the conf file
# context.properties = { module.raop = false }
nano disable-raop.conf
```

## 6) Reboot here

```bash
systemctl reboot
```

## 7) Codecs, drivers and layered packages

Why these choices:
- `firefox`: I remove the distro `firefox` here because I prefer to run the Flatpak `org.mozilla.firefox` for sandboxing and easier updates.
- `distrobox`: I prefer `distrobox` over `toolbox` for a easier container workflow.

```bash
rpm-ostree install \
  intel-media-driver \
  distrobox \
  gstreamer1-plugin-libav \
  gstreamer1-plugins-bad-free-extras \
  gstreamer1-plugins-bad-freeworld \
  gstreamer1-plugins-ugly \
  gstreamer1-vaapi \
  --allow-inactive
```

```bash
rpm-ostree override remove \
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
```

```bash
# Allow upgrades to later Fedora versions
sudo rpm-ostree update --uninstall rpmfusion-free-release --uninstall rpmfusion-nonfree-release --install rpmfusion-free-release --install rpmfusion-nonfree-release
```

## 8) Intel i915 kernel/module options (if applicable)
Create or edit `/etc/modprobe.d/i915.conf` with:

```text
options i915 enable_guc=2
options i915 enable_fbc=1
```

Then regenerate initramfs enabling the modprobe file so it is included:
```bash
sudo rpm-ostree initramfs --enable --arg=-I --arg=/etc/modprobe.d/i915.conf
```

## 9) One final reboot

```bash
systemctl reboot
```

