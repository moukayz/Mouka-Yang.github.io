---
title: Cmd Proxy Settings
categories:
- Misc 
---

<!-- more -->

## Bash
In terminal, run
```shell
echo 'export http_proxy="protocol://proxy-address:port"' >> ~/.bashrc
echo 'export https_proxy="protocol://proxy-address:port"' >> ~/.bashrc
source ~/.bashrc
```
done!

## Git
In terminal
```shell
git config --global http.proxy protocol://proxy-address:port
git config --global https.proxy protocol://proxy-address:port
```

## Apt
In terminal
```shell
sudo echo """
Acquire::http::proxy "protocol://proxy-address:port";
Acquire::https::proxy "protocol://proxy-address:port";
Acquire::ftp::proxy "protocol://proxy-address:port";
""" >> /etc/apt/apt.conf
```

## Wget
In terminal 
```shell
echo """
use_proxy=yes
http_proxy=protocol://proxy-address:port
https_proxy=protocol://proxy-address:port
""" >> ~/.wgetrc
```

## Curl
In terminal
```shell
echo "protocol://proxy-address:port" >> ~/.curlrc
```
