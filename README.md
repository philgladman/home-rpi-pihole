# home-rpi-pihole
Raspberry pi pihole DNS add blocker for home environment

# Prereqs
- K3s cluster on Raspberry Pi (without traefik) 
- MetalLB 
- Nginx Ingress
- Systemd-resolver turned off (ubuntu)
- Kustomize installed ([how-to](https://kubectl.docs.kubernetes.io/installation/kustomize/))

## Prereq Steps
### Raspberry Pi Setup
- Please follow this link for instructions on [How to install and setup K3s Cluster on raspberry pi](https://github.com/philgladman/home-rpi-k3s-cluster.git)
- This link will show you how to;
  - Prepare your raspberry pi for kubernetes
  - Spin up K3s
  - Install MetalLb
  - Install Nginx Ingress
- If you already have a Raspberry pi configured with K3s, MetalLb, and Nginx ingress, please move on to [Step 1.)](README.md#step-1---turn-off-systemd-resolver-ubuntu)

## Step 1.) - Turn off systemd-resolver (Ubuntu)
- If using ubuntu, check if systemd-resolver is running and using port 53 `sudo lsof -i :53`.
- If output of command shows systemd-resolver is using 53, example below;
```bash
COMMAND   PID            USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd-r 674 systemd-resolve   12u  IPv4  21062      0t0  UDP localhost:domain 
systemd-r 674 systemd-resolve   13u  IPv4  21063      0t0  TCP localhost:domain (LISTEN)
```
then run the following command to turn this off to free up port 53 for pihole `sudo sed -i "s/#DNS=/DNS=192.168.1.x/g; s/#DNSStubListener=yes/DNSStubListener=no/g" /etc/systemd/resolved.conf`. Change the IP Address with the IP of your Router.
- create sysmbolic link `sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf`
- reboot to apply changes `sudo systemctl reboot`
- confirm systemd-resolver is no longer using port 53 `sudo lsof -i :53`
- may have to update your /etc/hosts with `127.0.0.1 <your-hostname>`

## Step 2.) - Configure and install Pihole
- Create the directories below on the master node in order to persist our pihole data.
- `sudo mkdir -p /mnt/pihole/pihole`
- `sudo mkdir -p /mnt/pihole/dnsmasq.d`
- label master node so pihole container will only run on master node since that is where we want the persistent data to be stored `kubectl label nodes $(hostname) disk=disk1`
- clone this repo `git clone https://github.com/philgladman/home-rpi-pihole.git`
- cd into repo `cd home-rpi-pihole`
- Create password for pihole (replace `testpassword` with a custom password of your choice) `echo -n "testpassword" > pihole/pihole-pass`
- Lets create a local DNS name for our pihole server, so we can acces the pihole UI by a DNS name vs ip and port number. In my example, I use `pihole.phils-home.com`. You can use anything as this will only be accessible locally.
- change the `custom.list` file to have your custom DNS name for pihole `sed -i "s|pihole.phils-home.com|<your-local-dns-name-for-pihole>|g" pihole/custom.list`
- change the `custom.list` file to have the ip address of your nginx ingress controller `sed -i "s|192.168.1.x|$(kubectl get svc -n nginx-ingress nginx-ingress-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[].ip}')|g" pihole/custom.list`
- change pihole ingress name to your pihole custom DNS name `sed -i "s|pihole.phils-home.com|<your-local-dns-name-for-pihole>|g" pihole/pihole-ingress.yaml`
- change metallb ipaddress that will be used for by pihole `sed -i "s|192.168.1.x|<your-available-metallb-ip>|g" pihole/pihole-svc.yaml`
- change timezone in pihole to your local timezone `sed -i "s|America/New_York|<your-local-timezone>|g" pihole/pihole-cm.yaml`
- copy the `custom.list` file with your pihole custom dns entry over to the `/mnt/pihole/pihole/` dir with this `sudo cp pihole/custom.list /mnt/pihole/pihole/`
- Deploy pihole `kubectl apply -k .` 
- Wait for pod to spin up
- Now you just need to configure your router or your host to use the ip address of the udp/tcp pihole service as its DNS Server. You can get this ip address by running this command `kubectl get svc -n pihole pihole-dns-udp -o yaml -o jsonpath='{.status.loadBalancer.ingress[].ip}'`
- Once you have updated your DNS on your host to point to pihole, you should be able to access your pihole by this custom DNS name, without having to port forward
- In your browser, connect to your pihole custome DNS name (ex = `pihole.phils-home.com/admin`) and login.
- Pihole is up and running! Dont forget to configure your router or host to use pihole as its DNS Server.
