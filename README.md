# home-rpi-pihole
Raspberry pi pihole DNS add blocker for home environment
- `sudo mkdir -p /mnt/pihole/pihole`
- `sudo mkdir -p /mnt/pihole/dnsmasq.d`
- check if systemd-resolver is running and using port 53 `sudo lsof -i :53`
- ubuntu, turn of systemd-resolver `sudo sed -i "s/#DNS=/DNS=1.1.1.1/g; s/#DNSStubListener=yes/DNSStubListener=no/g" /etc/systemd/resolved.conf`
- create sysmbolic link `sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf`
- reboot to apply changes `sudo systemctl reboot`
- confirm systemd-resolver is no longer using port 53 `sudo lsof -i :53`