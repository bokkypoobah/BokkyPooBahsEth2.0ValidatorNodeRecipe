# BokkyPooBahs eth 2.0 Validator Node Recipe





# Prometheus

https://prometheus.io/

## Install Go

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
