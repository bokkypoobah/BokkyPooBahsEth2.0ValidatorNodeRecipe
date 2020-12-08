# BokkyPooBahs eth 2.0 Validator Node Recipe

**Work in progress**

**NOTE**: This is my recipe. Use at your own risk. Do your own research.

Basic recipe to build an eth2.0 validator node on an Intel NUC running Ubuntu, using go-ethereum as the eth1 node and Sigma Prime's [https://github.com/sigp/lighthouse](https://github.com/sigp/lighthouse) as the eth2 validator node.

Other references, including different eth1 and eth2 clients:
* https://github.com/SomerEsat/ethereum-staking-guide

<br />

<hr />

## Overview

* Deposit 32 ETH
* Hardware
* Operating System
* Install Go-Ethereum
* Install Lighthouse

<br />

<hr />

## Deposit 32 ETH

See https://launchpad.ethereum.org/ . It's now after genesis (01 Dec 2020), so your 32 ETH deposit go into a queue for a period of time before your eth2 validator become active.

Do this only if you are confident you can run a validator node and keep it online (my aim is for 90+% uptime).

<br />

<hr />

## Hardware

### Specifications

* Intel NUC 10th generation i5 or i7
* 16 GB RAM minimum
* 1 TB SSH minimum. eth1 chaindata starts at ~ 300GB and grows about 2GB per day

### Installation

* Open cover on the bottom by unscrewing the 4 screws
* Insert RAM chip(s)
* Insert SSD drive
* Close cover

### Hardware Settings

* Change BIOS setting so the computer restarts when the power supply is (re)connected - https://www.intel.com/content/www/us/en/support/articles/000054773/intel-nuc.html . Instructions are incorrect - select Power -> Secondary Power Settings -> After Power Failure -> **Power On**. Test by removing and re-inserting the power supply plug.

You can set up a password to bootup the NUC.

### Warning

* I had some DMA failures with the Samsung 870 QVO. Some of the Samsung SSDs have had the same issue, described in https://bugzilla.kernel.org/show_bug.cgi?id=201693 . I've replaced my SSDs with the Crucial MX500 series

<br />

<hr />

## Operating System

Install Ubuntu 20.04 LTS from https://ubuntu.com/ .

See https://www.makeuseof.com/tag/how-to-boot-a-linux-live-usb-stick-on-your-mac/ for instructions on creating a USB installation disk, using either Etcher or command line utilities.

### Install Ubuntu Server

* Select language
* Use updated installer if given the choice
* Configure network connection. Wired preferred, but wireless a bit more to do - http://smalldatum.blogspot.com/2020/01/setting-up-ubuntu-1804-server-on-intel.html
* Storage - use entire disk
  * Set up as an LVM group
    * Encrypt with LUKS ideally (you will have to enter a password to boot up)
* Enter your name; your server name; your username and password
* Install OpenSSH server
* Don't need to install Snaps

### Resize Disk

The Ubuntu installation will only configure part of your disk for the LVM group. See https://manjaro.site/how-to-extend-lvm-disk-on-ubuntu-20-04/ for some details on expanding your logical disk to use up the whole physical disk.

* `df -h` will show the current disk usage
* `sudo vgs` will show the volume group
* `sudo pvs` and `sudo pvdisplay` will show the physical volume
* `sudo lvs` will show the logical volumes
* `sudo lvm lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv` will extend the logical volume
* `sudo resize2fs -p /dev/mapper/ubuntu--vg-ubuntu--lv`
* `df -h` will show the current disk usage

### Update

* Restart
* `sudo apt-get update`
* `sudo apt-get upgrade`

And sometimes:

* `sudo apt-get dist-upgrade`

### New Distribution?

If there is a new release upgrade:

* `do-release-upgrade`

### Some Useful Commands

#### Network Info

* `ifconfig -a` shows the network interfaces

#### Connect via SSH from Mac

* `ssh user@ipaddress` to initially log in
* `ssh-copy-id user@ipaddress` to copy your public key to the new server
* `ssh user@ipaddress` should allow you to log in without having to specify a password
* `sudo vi /etc/ssh/sshd_config` - https://www.cyberciti.biz/faq/how-to-disable-ssh-password-login-on-linux/
  * Uncomment `PasswordAuthentication no`
  * Add `PermitRootLogin no`
  * Save and close the file
* `sudo systemctl restart sshd`

#### Install Firewall

Enable only the following incoming ports:

* 22 Secure shell (ssh)
* 30303 Go Ethereum communications port

https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-14-04

* `sudo apt-get install ufw`
* `sudo vi /etc/default/ufw`. Check `IPV6=yes`
* `sudo ufw reset`
* `sudo ufw status verbose`. Should show inactive
* `sudo ufw default deny incoming`
* `sudo ufw default allow outgoing`
* `sudo ufw allow ssh`
* `sudo ufw allow 30303`
* `sudo ufw enable`
* `sudo ufw status verbose`. Should show active

#### Set Timezone

`sudo timedatectl set-timezone Australia/Sydney`

#### Install Further Software

* `sudo apt-get install lm-sensors` to install temperature utilities, e.g., `sensors`
* `sudo apt-get install net-tools` to install tools for commands like `ifconfig -a` (network interface) and `netstat`

<br />

<hr />

## Install  Go Ethereum

This is installed for a user `geth` with a home directory `/home/geth`. Go Ethereum data will be located in `/home/geth/.ethereum`.

<br />

### Install Software

Installation from https://geth.ethereum.org/docs/install-and-build/installing-geth#install-on-ubuntu-via-ppas :

```bash
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install ethereum
```

`geth` is now installed in `/usr/local/bin/geth`.

<br />

### Create `geth` User

```
sudo useradd -m -s /bin/bash geth
```

To log in as the **geth** user (not now, but if you want to check the data):

```
sudo su - geth
```

### Install Go Ethereum As A Systemd Service

```
sudo vi /etc/systemd/system/geth.service
```

Add the following:

```
[Unit]
Description=Geth

[Service]
Type=simple
User=geth
ExecStart=/usr/bin/geth --http
KillMode=process
TimeoutStopSec=300
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=default.target
```

Reload the sysmtemd configuration files

```
sudo systemctl daemon-reload
```

Enable The Go Ethereum Service

```
sudo systemctl enable geth
```

Start The Go Ethereum Service

```
sudo systemctl start geth
```

Check the logs for the Go Ethereum Service

```
journalctl -f -u geth
```

Hit ^C to exit

<br />

<hr />

# TODO BELOW

## lighthouse

sudo useradd -m -s /bin/bash lighthouse

sudo su - lighthouse

mkdir install
cd install
wget https://github.com/sigp/lighthouse/releases/download/v1.0.1/lighthouse-v1.0.1-x86_64-unknown-linux-gnu.tar.gz
tar xvzf lighthouse-v1.0.1-x86_64-unknown-linux-gnu.tar.gz
mkdir ~/bin
mv lighthouse ~/bin
~/bin/lighthouse --version


## Prometheus

https://prometheus.io/

    sudo useradd -m -s /bin/bash prometheus
    sudo su - prometheus

    mkdir ~/install
    cd ~/install
    # Check website for latest version
    wget https://github.com/prometheus/prometheus/releases/download/v2.23.0/prometheus-2.23.0.linux-amd64.tar.gz
    # https://prometheus.io/docs/introduction/first_steps/
    tar xvfz prometheus-*.tar.gz
    cd prometheus-*
    ./prometheus --help
    # Configuration
    vi prometheus.yml
    # set scrape_interval: 12s and evaluation_interval: 12s to match slot
    ./prometheus --config.file=prometheus.yml
    ssh -nNT -L 9090:192.168.1.133:9090 username@remotehost
    http://localhost:9090
    http://localhost:9090/metrics

    cd ~/prometheus
    wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
    tar xvfz node_exporter-*.*-amd64.tar.gz
    cd node_exporter-*.*-amd64
    ./node_exporter
    curl http://localhost:9100/metrics


https://www.digitalocean.com/community/tutorials/how-to-install-prometheus-on-ubuntu-16-04

sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

Download from https://prometheus.io/download/
prometheus-2.23.0.linux-amd64.tar.gz
node_exporter-1.0.1.linux-amd64.tar.gz

tar xvf prometheus-2.23.0.linux-amd64.tar.gz

sudo cp prometheus-2.23.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.23.0.linux-amd64/promtool /usr/local/bin/

sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

sudo cp -r prometheus-2.23.0.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.23.0.linux-amd64/console_libraries /etc/prometheus

sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries

# Customise /etc/prometheus/prometheus.yml


sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml

sudo -u prometheus /usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

create /etc/systemd/system/prometheus.service

sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl status prometheus
sudo systemctl enable prometheus


sudo cp node_exporter-1.0.1.linux-amd64/node_exporter /usr/local/bin
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl status node_exporter
sudo systemctl enable node_exporter


sudo nano /etc/prometheus/prometheus.yml
...
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']


sudo systemctl restart prometheus
sudo systemctl status prometheus


https://vinayakpandey-7997.medium.com/pushing-bash-script-result-to-prometheus-using-pushgateway-a0760cd261e

sudo useradd --no-create-home --shell /bin/false pushgateway

wget https://github.com/prometheus/pushgateway/releases/download/v1.3.0/pushgateway-1.3.0.linux-amd64.tar.gz
sudo cp pushgateway-1.3.0.linux-amd64/pushgateway /usr/local/bin
sudo chown pushgateway:pushgateway /usr/local/bin/pushgateway

sudo vi /etc/systemd/system/pushgateway.service
[Unit]
Description=Prometheus Pushgateway
Wants=network-online.target
After=network-online.target[Service]
User=pushgateway
Group=pushgateway
Type=simple
ExecStart=/usr/local/bin/pushgateway[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl start pushgateway
sudo systemctl status pushgateway
sudo systemctl enable pushgateway

sudo vi /etc/prometheus/prometheus.yml

- job_name: 'Pushgateway'
  honor_labels: true
  static_configs:
  - targets: ['localhost:9091']

https://grafana.com/docs/grafana/latest/installation/debian/

sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install grafana

sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable grafana-server
sudo /bin/systemctl start grafana-server

# does not work
# geth --metrics
# curl localhost:6060/metrics

https://github.com/sigp/lighthouse/blob/5a3b94cbb4a82f999c2deb5c45146eaf58146957/book/src/advanced_metrics.md
lighthouse bn --metrics
curl localhost:5054/metrics

lighthouse vc --metrics
curl localhost:5064/metrics

grafana

sudo apt-get update
sudo apt-get install ssmtp

sudo vi /etc/ssmtp/ssmtp.conf

root=eth2@bok.id.au
mailhub=mail.bok.id.au:465
FromLineOverride=YES
AuthUser=eth2@bok.id.au
AuthPass=<password>
UseTLS=YES
UseSTARTTLS=YES


echo "Test SSMTP" | ssmtp eth2@bok.id.au

sudo apt install mailutils

sudo chmod 640 /etc/ssmtp/ssmtp.conf
sudo chown root:mail /etc/ssmtp/ssmtp.conf

 echo "sample text" | mail -s "Subject" eth2@bok.id.au

sudo apt-get remove ssmtp
sudo vim /etc/apt/sources.list
%s/us.archive/au.archive/g




https://www.medo64.com/2020/06/sendmail-via-gmail-on-ubuntu-server/

debconf-set-selections <<< "postfix postfix/main_mailer_type string 'zozzfozzel'"
debconf-set-selections <<< "postfix postfix/mailname string ''"
apt-get install --assume-yes postfix libsasl2-modules


unset HISTFILE
echo "[mail.bok.id.au]:465 eth2@bok.id.au:password" > /etc/postfix/sasl/sasl_passwd
postmap /etc/postfix/sasl/sasl_passwd
chmod 0600 /etc/postfix/sasl/sasl_passwd /etc/postfix/sasl/sasl_passwd.db


sed -i 's/relayhost = /relayhost = [mail.bok.id.au]:465/' /etc/postfix/main.cf
cat <<EOF >> /etc/postfix/main.cf
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_wrappermode = yes
smtp_tls_security_level = encrypt
EOF

systemctl restart postfix

echo "Subject: Test via sendmail" | sendmail -v eth2@bok.id.au


Nov 30 16:30:50 zozzfozzel postfix/smtp[30948]: SMTPS wrappermode (TCP port 465) requires setting "smtp_tls_wrappermode = yes", and "smtp_tls_security_level = encrypt" (or stronger)

/etc/grafana/grafana.ini

[smtp]
;enabled = false
enabled = true
;host = localhost:25
host = mail.bok.id.au:465
;user =
user = eth2@bok.id.au
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
;password =
password = 56nMlwr6LmuY
;cert_file =
cert_file =
;key_file =
key_file =
;skip_verify = false
skip_verify = false
;from_address = admin@grafana.localhost
from_address = eth2@bok.id.au
;from_name = Grafana
from_name = eth2_zozzfozzel
# EHLO identity in SMTP dialog (defaults to instance_name)
;ehlo_identity = dashboard.example.com
ehlo_identity =
# SMTP startTLS policy (defaults to 'OpportunisticStartTLS')
;startTLS_policy = NoStartTLS
