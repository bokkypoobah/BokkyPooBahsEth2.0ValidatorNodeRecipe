# BokkyPooBahs eth 2.0 Validator Node Recipe

**Work in progress**

**NOTE**: This is my recipe. Use at your own risk. Do your own research.

Basic recipe to build an eth2.0 validator node on an Intel NUC running Ubuntu.

* go-ethereum
* Sigma Prime's [https://github.com/sigp/lighthouse](https://github.com/sigp/lighthouse)

Alternatives
* eth1
  * go Ethereum
  * OpenEthereum
  * besu
  * turbogeth
  * nethermind
* eth2.0
  * [@sigp_io](https://twitter.com/sigp_io)'s Lighthouse
  * [@prylabs](https://twitter.com/prylabs)'s Prysm
  * [@Teku_ConsenSys](https://twitter.com/Teku_ConsenSys)'s Teku
  * [@ethnimbus](https://twitter.com/ethnimbus)'s Nimbus

## Deposit 32 ETH


## Hardware


## Installing Ubuntu Linux



## geth
https://geth.ethereum.org/docs/install-and-build/installing-geth#install-on-ubuntu-via-ppas

    sudo add-apt-repository -y ppa:ethereum/ethereum
    sudo apt-get update
    sudo apt-get install ethereum
    # geth is now installed in /usr/local/bin/geth

Add the user **geth** with a home directory `/home/geth` with the bash shell:

```
sudo useradd -m -s /bin/bash geth
```

To log in as the **geth** user:

```
sudo su - geth


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
