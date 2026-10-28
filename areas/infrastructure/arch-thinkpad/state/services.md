## system enabled
NetworkManager-dispatcher.service
NetworkManager-wait-online.service
NetworkManager.service
apparmor.service
bluetooth.service
fstrim.timer
fwupd-refresh.timer
getty@.service
libvirtd-admin.socket
libvirtd-ro.socket
libvirtd.service
libvirtd.socket
mullvad-daemon.service
mullvad-early-boot-blocking.service
nftables.service
paccache.timer
plasmalogin.service
reflector.timer
remote-fs.target
restic-b2-check.timer
restic-b2.timer
snapper-cleanup.timer
snapper-timeline.timer
sshd.service
systemd-timesyncd.service
systemd-userdbd.socket
tailscaled.service
tlp.service
usbguard.service
virtlockd-admin.socket
virtlockd.socket
virtlogd-admin.socket
virtlogd.socket

## user enabled
inbox-sync.timer
p11-kit-server.socket
pipewire-pulse.service
pipewire-pulse.socket
pipewire.service
pipewire.socket
podman.socket
restic-b2-stale.timer
ssh-agent.service
syncthing.service
vault-push.timer
wireplumber.service
xdg-user-dirs.service

## graphical-session.target dependency graph (loaded units, collect-time snapshot)
graphical-session.target
* |-app-org.fcitx.Fcitx5@autostart.service
* |-app-org.kde.discover.notifier@autostart.service
* |-app-org.kde.plasma\x2dfallback\x2dsession\x2drestore@autostart.service
* |-at-spi-dbus-bus.service
* |-drkonqi-coredump-cleanup.service
* |-drkonqi-coredump-pickup.service
* |-drkonqi-sentry-postman.service
* |-gvfs-daemon.service
* |-gvfs-metadata.service
* |-kde-baloo.service
* |-plasma-baloorunner.service
* |-plasma-gmenudbusmenuproxy.service
* |-plasma-kaccess.service
* |-plasma-kactivitymanagerd.service
* |-plasma-kcminit-phase1.service
* |-plasma-kcminit.service
* |-plasma-kded6.service
* |-plasma-kglobalaccel.service
* |-plasma-knighttimed.service
* |-plasma-krunner.service
* |-plasma-ksmserver.service
* |-plasma-kwallet-pam.service
* |-plasma-kwin_wayland.service
* |-plasma-plasmashell.service
* |-plasma-polkit-agent.service
* |-plasma-powerdevil.service
* |-plasma-xdg-desktop-portal-kde.service
* |-plasma-xembedsniproxy.service
* |-xdg-desktop-portal-gtk.service
* |-xdg-desktop-portal.service
* |-xdg-document-portal.service
* |-xdg-permission-store.service
* `-basic.target
*   |-paths.target
*   |-sockets.target
*   | |-dbus.socket
*   | |-dirmngr.socket
*   | |-drkonqi-coredump-launcher.socket
*   | |-gpg-agent-browser.socket
*   | |-gpg-agent-extra.socket
*   | |-gpg-agent-ssh.socket
*   | |-gpg-agent.socket
*   | |-keyboxd.socket
*   | |-p11-kit-server.socket
*   | |-pipewire-pulse.socket
*   | |-pipewire.socket
*   | |-podman.socket
*   | |-systemd-ask-password.socket
*   | |-systemd-importd.socket
*   | |-systemd-journalctl.socket
*   | |-systemd-machined.socket
*   | `-systemd-storage-fs.socket
*   `-timers.target
*     |-drkonqi-coredump-cleanup.timer
*     |-drkonqi-sentry-postman.timer
*     |-inbox-sync.timer
*     |-restic-b2-stale.timer
*     `-vault-push.timer
