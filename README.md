# BokkyPooBahs eth 2.0 Validator Node Recipe

**Work in progress**

**NOTE**: This is my recipe. Use at your own risk. Do your own research.

Basic recipe to build an eth2.0 validator node on an Intel NUC running Ubuntu, using go-ethereum as the eth1 node and Sigma Prime's [https://github.com/sigp/lighthouse](https://github.com/sigp/lighthouse) as the eth2 validator node.

Other references, including different eth1 and eth2 clients:
* https://github.com/SomerEsat/ethereum-staking-guide

<br />

<hr />

## Overview

* [Deposit 32 ETH](#deposit-32-eth)
* [Hardware](#hardware)
* [Operating System](#operating-system)
* [Install Go Ethereum](#install-go-ethereum)
* [Install Lighthouse](#install-lighthouse)
* [Install Prometheus, Pushgateway, Node Exporter And Grafana](#install-prometheus-pushgateway-node-exporter-and-grafana)

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

* `irq/65-i2c-INT3` takes up CPU time. Workaround is to add `blacklist tps6598x` to `/etc/modprobe.d/blacklist.conf`

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

Reference https://help.dreamhost.com/hc/en-us/articles/216499537-How-to-configure-passwordless-login-in-Mac-OS-X-and-Linux

#### Install Firewall

Enable only the following incoming ports:

* 22 Secure shell (ssh)
* 30303 Go Ethereum communications port

Configuration:

* `sudo apt-get install ufw`
* `sudo vi /etc/default/ufw`. Check `IPV6=yes`
* `sudo ufw reset`
* `sudo ufw status verbose`. Should show inactive
* `sudo ufw default deny incoming`
* `sudo ufw default allow outgoing`
* `sudo ufw allow ssh`
* `sudo ufw allow 30303`. eth1 p2p communication port
* `sudo ufw allow 9000`. eth2 p2p communication port
* `sudo ufw enable`
* `sudo ufw status verbose`. Should show active

Reference https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-14-04

#### Set Timezone

`sudo timedatectl set-timezone Australia/Sydney`

#### Install Further Software

* `sudo apt-get install lm-sensors` to install temperature utilities, e.g., `sensors`
* `sudo apt-get install net-tools` to install tools for commands like `ifconfig -a` (network interface) and `netstat`

<br />

<hr />

## Install Go Ethereum

This is installed for a user `geth` with a home directory `/home/geth`. Go Ethereum data will be located in `/home/geth/.ethereum`.

<br />

### Installation via PPA

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

## Install Lighthouse

### Create `lighthouse` User

```
sudo useradd -m -s /bin/bash lighthouse
```

To log in as the **lighthouse** user:

```
sudo su - lighthouse
```

### Install From Tar File

Get latest version from https://github.com/sigp/lighthouse/releases

```
mkdir install
cd install
wget https://github.com/sigp/lighthouse/releases/download/v1.0.3/lighthouse-v1.0.3-x86_64-unknown-linux-gnu.tar.gz
tar xvf lighthouse-v1.0.3-x86_64-unknown-linux-gnu.tar.gz
mkdir ~/bin
mv lighthouse ~/bin
~/bin/lighthouse --version
```

### Install Lighthouse (Beacon) As A Systemd Service

```
sudo vi /etc/systemd/system/lighthouse_beacon.service
```

Add the following:

```
[Unit]
Description=Lighthouse Beacon

[Service]
Type=simple
User=lighthouse
ExecStart=/home/lighthouse/bin/lighthouse bn --http --staking --metrics

[Install]
WantedBy=default.target
```

Reload the sysmtemd configuration files

```
sudo systemctl daemon-reload
```

Enable The Lighthouse (Beacon) Service

```
sudo systemctl enable lighthouse_beacon
```

Start The Lighthouse (Beacon) Service

```
sudo systemctl start lighthouse_beacon
```

Check the logs for the Lighthouse (Beacon) Service

```
journalctl -f -u lighthouse_beacon
```

Hit ^C to exit

### Install Lighthouse (Validator) As A Systemd Service

Note: You will need to import the validator keys before this will work.

Warning: Running more than one instance of the validator with your validator keys can result in your account getting slashed.

```
sudo vi /etc/systemd/system/lighthouse_validator.service
```

Add the following:

```
[Unit]
Description=Lighthouse Validator

[Service]
Type=simple
User=lighthouse
ExecStart=/home/lighthouse/bin/lighthouse vc --metrics --graffiti "BokkyPooBah wuz here!"

[Install]
WantedBy=default.target
```

Reload the sysmtemd configuration files

```
sudo systemctl daemon-reload
```

Enable The Lighthouse (Validator) Service

```
sudo systemctl enable lighthouse_validator
```

Start The Lighthouse (Validator) Service

```
sudo systemctl start lighthouse_validator
```

Check the logs for the Lighthouse (Validator) Service

```
journalctl -f -u lighthouse_validator
```

Hit ^C to exit

<br />

<hr />

## Install Prometheus, Pushgateway, Node Exporter And Grafana

References: https://prometheus.io/, https://www.digitalocean.com/community/tutorials/how-to-install-prometheus-on-ubuntu-16-04

### Add Service Users

We will be running prometheus, pushgateway and node_exporter as separate service users, without login capability:

```
sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false pushgateway
sudo useradd --no-create-home --shell /bin/false node_exporter
```

### Install Prometheus

Create directories:

```
mkdir ~/prometheus_install
cd ~/prometheus_install
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

Download from https://prometheus.io/download/:

* prometheus-2.23.0.linux-amd64.tar.gz

Unpack and install Prometheus:

```
wget https://github.com/prometheus/prometheus/releases/download/v2.23.0/prometheus-2.23.0.linux-amd64.tar.gz
tar xvf prometheus-2.23.0.linux-amd64.tar.gz
sudo cp prometheus-2.23.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.23.0.linux-amd64/promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
sudo cp -r prometheus-2.23.0.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.23.0.linux-amd64/console_libraries /etc/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

`sudo vi /etc/prometheus/prometheus.yml`:

```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'Pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['localhost:9091']

  - job_name: 'eth2_beacon'
    static_configs:
      - targets: ['localhost:5054']

  - job_name: 'eth2_validator'
    static_configs:
      - targets: ['localhost:5064']
```

`sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml`

`sudo vi /etc/systemd/system/prometheus.service`:

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

Reload systemd configuration and enable Prometheus:

```
sudo systemctl daemon-reload
sudo systemctl enable prometheus
```

Commands to start, check status and stop Prometheus:

```
sudo systemctl start prometheus
sudo systemctl status prometheus
sudo systemctl stop prometheus
```

To check the logs:

```
journalctl -f -u prometheus
```

Check the Prometheus metrics page:

```
curl localhost:9090/metrics
```

### Install Node Exporter

```
cd ~/prometheus_install
```

Download from https://prometheus.io/download/:

* node_exporter-1.0.1.linux-amd64.tar.gz

Unpack and install Node Exporter:

```
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
tar xvf node_exporter-1.0.1.linux-amd64.tar.gz
sudo cp node_exporter-1.0.1.linux-amd64/node_exporter /usr/local/bin
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

`sudo vi /etc/systemd/system/node_exporter.service`:

```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Reload systemd configuration and enable Node Exporter:

```
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
```

Commands to start, check status and stop Node Exporter:

```
sudo systemctl start node_exporter
sudo systemctl status node_exporter
sudo systemctl stop node_exporter
```

To check the logs:

```
journalctl -f -u node_exporter
```

Check the Node Exporter metrics page:

```
curl localhost:9100/metrics
```

Check that /etc/prometheus/prometheus.yml has the following entry:

```
  - job_name: 'node_exporter'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9100']
```

Restart Prometheus:

```
sudo systemctl restart prometheus
```

### Install Pushgateway

Reference https://vinayakpandey-7997.medium.com/pushing-bash-script-result-to-prometheus-using-pushgateway-a0760cd261e

```
cd ~/prometheus_install
```

Download from https://prometheus.io/download/:

* pushgateway-1.3.0.linux-amd64.tar.gz

Unpack and install Pushgateway:

```
wget https://github.com/prometheus/pushgateway/releases/download/v1.3.0/pushgateway-1.3.0.linux-amd64.tar.gz
tar xvf pushgateway-1.3.0.linux-amd64.gz
sudo cp pushgateway-1.3.0.linux-amd64/pushgateway /usr/local/bin
sudo chown pushgateway:pushgateway /usr/local/bin/pushgateway
```

`sudo vi /etc/systemd/system/pushgateway.service`:

```
[Unit]
Description=Prometheus Pushgateway
Wants=network-online.target
After=network-online.target

[Service]
User=pushgateway
Group=pushgateway
Type=simple
ExecStart=/usr/local/bin/pushgateway

[Install]
WantedBy=multi-user.target
```

Reload systemd configuration and enable Pushgateway:

```
sudo systemctl daemon-reload
sudo systemctl enable pushgateway
```

Commands to start, check status and stop Pushgateway:

```
sudo systemctl start pushgateway
sudo systemctl status pushgateway
sudo systemctl stop pushgateway
```

To check the logs:

```
journalctl -f -u pushgateway
```

Check the Pushgateway metrics page:

```
curl localhost:9091/metrics
```

Check that /etc/prometheus/prometheus.yml has the following entry:

sudo vi /etc/prometheus/prometheus.yml

```
  - job_name: 'Pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['localhost:9091']
```

### Install Grafana

Reference: https://grafana.com/docs/grafana/latest/installation/debian/

```
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install grafana
```

```
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable grafana-server
sudo /bin/systemctl start grafana-server
```

Create a SSH tunnel from your laptop to the server:

```
ssh -nNT -L 3000:127.0.0.1:3000 hostname
```

You can then browse http://localhost:3000 on your laptop to configure Grafana. Initial username/password is admin/admin.

Then:

* Select Configuration -> Data Sources
* Select Prometheus. Set URL to `http://localhost:9090`

Set up the data source from the configuration icon. Select Prometheus.

<br />

<hr />

## TODO BELOW

```
https://www.medo64.com/2020/06/sendmail-via-gmail-on-ubuntu-server/

debconf-set-selections <<< "postfix postfix/main_mailer_type string 'zozzfozzel'"
debconf-set-selections <<< "postfix postfix/mailname string ''"
apt-get install --assume-yes postfix libsasl2-modules

unset HISTFILE
echo "[mail.server.com]:465 eth2@mail.server.com:password" > /etc/postfix/sasl/sasl_passwd
postmap /etc/postfix/sasl/sasl_passwd
chmod 0600 /etc/postfix/sasl/sasl_passwd /etc/postfix/sasl/sasl_passwd.db

sed -i 's/relayhost = /relayhost = [mail.server.com]:465/' /etc/postfix/main.cf
cat <<EOF >> /etc/postfix/main.cf
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_wrappermode = yes
smtp_tls_security_level = encrypt
EOF

systemctl restart postfix

echo "Subject: Test via sendmail" | sendmail -v eth2@mail.server.com


Nov 30 16:30:50 zozzfozzel postfix/smtp[30948]: SMTPS wrappermode (TCP port 465) requires setting "smtp_tls_wrappermode = yes", and "smtp_tls_security_level = encrypt" (or stronger)

/etc/grafana/grafana.ini

[smtp]
;enabled = false
enabled = true
;host = localhost:25
host = mail.server.com:465
;user =
user = eth2@mail.server.com
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
;password =
password = {email_password}
;cert_file =
cert_file =
;key_file =
key_file =
;skip_verify = false
skip_verify = false
;from_address = admin@grafana.localhost
from_address = eth2@mail.server.com
;from_name = Grafana
from_name = eth2_zozzfozzel
# EHLO identity in SMTP dialog (defaults to instance_name)
;ehlo_identity = dashboard.example.com
ehlo_identity =
# SMTP startTLS policy (defaults to 'OpportunisticStartTLS')
;startTLS_policy = NoStartTLS
```
