# home-rpi-pihole
Raspberry pi pihole DNS add blocker for home environment

# Prereqs
- K3s cluster (Withou Traefik) on Raspberry Pi
- MetalLB installed on cluster
- Nginx Ingress installed on cluster
- Systemd-resolver turned off (ubuntu)

# Prereqs Steps
## Step 1.) - Raspberry Pi Setup
- If your Raspberry Pi is already configured, and K3s is running, move on to step [## Step 3.) - Install MetalLB on K3s Cluster]
- Use Raspberry Pi Imager (https://www.raspberrypi.com/software/) to write Ubuntu Server 20.04 LTS (64-BIT) to sd card
- After image has been succesfully written, Eject SD Card from computer, and then reinsert card into computer.
- click on SD Card, and open up the newly created `boot` or `system-boot` folder
- Open up the `network-config.txt` file in a txt editor, create this file if it does not exisit.
- Add the contents below to the end of the file, this will configure your static ip. Replace the ip address with the ip address you want your rpi to have. Also replace the gateway ip, and dns ip with your home networks values.

```bash
version: 2
ethernets:
  eth0:
    addresses:
      - 192.168.1.x/24
    gateway4: 192.168.1.x
    nameservers:
      addresses: [192.168.1.x]
    optional: true
```

- In the `boot`  or `system-boot` folder, open up the `cmdline.txt` file in a txt editor
- Add the following `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` to the end of the top line, this will enable cgroups
- add a blank file named `ssh` into the `boot` or `system-boot` folder by running the following command. `touch ssh` to create the blank `ssh` file. Find newly created `ssh` file and move it into the sd cards `boot` or `system-boot` folder.
- Eject SD card and plug into rpi.
- Turn rpi on and let rpi boot up.
- ssh into the pi with `ssh ubuntu@<rpi-ip-address>`. The password will be `ubuntu`
- change password

## Step 2.) - Deploy and configure k3s single node cluster
- create k3s cluster without install teaefik (we will use nginx ingress instead later) `curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable traefik" sh`
- copy newly created kubeconfig to home dir `mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown ubuntu:ubuntu ~/.kube/config`
- export kubeconfig `echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc && source ~/.bashrc`
- install helm `curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

## Step 3.) - Install MetalLB on K3s Cluster
- If your K3s cluster already has MetalLB installed, move onto step [## Step 4.) - Install Nginx Ingress on K3s Cluster]

## Step 4.) - Install Nginx Ingress on K3s Cluster
- If your K3s cluster already has Nginx Ingress installed, move onto step [## Step 5.) - Turn off systemd-resolver (Ubuntu)]

## Step 5.) - Turn off systemd-resolver (Ubuntu)
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

## Step 6.) - Configure and install Pihole
- Create the directories below on the master node inorder to persist our pihole data.
- `sudo mkdir -p /mnt/pihole/pihole`
- `sudo mkdir -p /mnt/pihole/dnsmasq.d`
- label master node so pihole container will only run on master node since that is where we want the persistent data to be stored `kubectl label nodes $(hostname) disk=disk1`
- clone this repo `git clone https://github.com/philgladman/home-rpi-pihole.git`
- cd into repo `cd home-rpi-pihole`
- Create password for pihole (replace `testpassword` below with a custom password of your choice) `echo -n "testpassword" > pihole/pihole-pass`
- after we deploy pihole, we will create a local DNS name for our pihole server, inorder to acces the UI by a DNS name vs ip. For now we need to update the value in our ingress file to the DNS name that you come up with. In my example I use `pihole.phils-home.com`. You can use anything as this will only be accessible locally.
- change pihole ingress name to your custom DNS name for pihole server `sed -i "s|pihole.phils-home.com|<your-local-dns-namme-for-pihole>|g" pihole/pihole-ingress.yaml`
- change metallb ipaddress that will be used for by pihole `sed -i "s|192.168.1.202|<your-available-metallb-ip>|g" pihole/pihole-svc.yaml`
- change timezone in pihole to your local timezone `sed -i "s|America/New_York|<your-local-timezone>|g" pihole/pihole-cm.yaml`
- Deploy pihole `kubectl apply -k .` 
- Wait for pod to spin up
- For now we will need to port forward pihole on our local machine until we have our pihole local DNS name configured, `kubectl port-forward svc/pihole-ui-svc 8082:8082 -n pihole`.
- Access pihole in the browser `localhost:8082/admin`
- Login with the password you made earlier
- On the left bar, click on `Local DNS`, and then click on `DNS Records`.
- In the `Domain:` field, type in your custom DNS name for pihole server (ex = `pihole.phils-home.com`)
- For the `IP Address:` field, type in the output to the following command `kubectl get ing -n pihole -o jsonpath='{.items[].status.loadBalancer.ingress[].ip}'`, this should be the ip address of your metallb.
- Click `add`
- now you should be able to access your pihole by this custom DNS name, without having to port forward
- Enter `ctrl+c` on your terminal tab that is port forwarding to cancel the port forward.
- In your browser, connect to your pihole custome DNS name (ex = `pihole.phils-home.com/admin`) and login.
- Pihole is up and running! Now you just need to configure your router or your host to use this the ip address of the udp/tcp pihole service as its DNS Server. You can get this ip address by running this command `kubectl get svc -n pihole pihole-dns-udp -o yaml -o jsonpath='{.status.loadBalancer.ingress[].ip}'`
