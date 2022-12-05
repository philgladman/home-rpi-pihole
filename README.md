# home-rpi-pihole
Raspberry pi pihole DNS add blocker for home environment
- `sudo mkdir -p /mnt/pihole/pihole`
- `sudo mkdir -p /mnt/pihole/dnsmasq.d`
- check if systemd-resolver is running and using port 53 `sudo lsof -i :53`
- ubuntu, turn of systemd-resolver `sudo sed -i "s/#DNS=/DNS=1.1.1.1/g; s/#DNSStubListener=yes/DNSStubListener=no/g" /etc/systemd/resolved.conf`
- create sysmbolic link `sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf`
- reboot to apply changes `sudo systemctl reboot`
- confirm systemd-resolver is no longer using port 53 `sudo lsof -i :53`

## Configure
- Create password for pihole
```bash
cat >"pihole-https/pihole-pass" <<EOF
testpassword
EOF
```
- change pihole ingress name to your custom DNS name `sed -i "s|pihole.phils-home.com|<your-local-dns-namme-for-pihole>|g" pihole-https/pihole-ingress.yaml`
- change metallb ipaddress that will be used for by pihole `sed -i "s|192.168.1.202|<your-available-metallb-ip>|g" pihole-https/pihole-svc.yaml`
- change timezone in pihole to your local timezone `sed -i "s|America/New_York|<your-local-timezone>|g" pihole-https/pihole-cm.yaml`

- `kubectl apply -k .` so far creates everything, and can login, and point macbook dns to here, and queires go thru