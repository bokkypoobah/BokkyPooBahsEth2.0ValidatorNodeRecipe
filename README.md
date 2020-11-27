# BokkyPooBahs eth 2.0 Validator Node Recipe

**Work in progress**


Basic recipe to build an eth2.0 validator node on an Intel NUC running Ubuntu.

* go-ethereum
* Sigma Prime's [https://github.com/sigp/lighthouse](https://github.com/sigp/lighthouse)

Alternatives
* eth1
  * [https://twitter.com/sigp_io](@sigp_io)'s
  * go Ethereum
  * OpenEthereum
  * besu
  * turbogeth
  * nethermind
* eth2.0
  * [https://twitter.com/sigp_io](@sigp_io)'s Lighthouse
  * [https://twitter.com/prylabs](@prylabs)'s Prysm
  * [https://twitter.com/Teku_ConsenSys](@Teku_ConsenSys)'s Teku
  * [https://twitter.com/ethnimbus](@ethnimbus)'s Nimbus

# Prometheus

https://prometheus.io/

    mkdir ~/prometheus
    cd ~/prometheus
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


## Install Go
Only if building Prometheus from source
https://golang.org/doc/install

    mkdir ~/go
    cd ~/go
    # Check website for latest version
    wget https://golang.org/dl/go1.15.5.linux-amd64.tar.gz
    sudo tar -C /usr/local -xzf go1.15.5.linux-amd64.tar.gz
    vi ~/.profile
    # add export PATH=$PATH:/usr/local/go/bin
    . ~/.profile
    go version
    # go version go1.15.5 linux/amd64
