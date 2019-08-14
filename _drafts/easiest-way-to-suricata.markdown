---
layout: "post"
title: "The Easiest Way to Suricata"
date: "2018-05-20 09:54"
---

The easiest way to get to network detection with suricata.

**install suri**
- [install via ppa](https://redmine.openinfosecfoundation.org/projects/suricata/wiki/Ubuntu_Installation_-_Personal_Package_Archives_%28PPA%29)
```
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt-get update
sudo apt-get install suricata 
```
- test install, show errors
```
sudo suricata -c /etc/suricata/suricata.yaml -i eth1 -k none --init-errors-fatal
```

**patch suricata.yaml**
```
curl https://pastebin.com/raw/2bEQGucv > /tmp/suricata.yaml.patch
patch -b /etc/suricata/suricata.yaml < /tmp/suricata.yaml.patch
apt-get install -y jq curl
tail -f eve.json | grep '"event_type":"alert"' | jq '.alert'
```
- insert some stuff about using for 1 pcap file and 1 rules file
- talk about variants of this for dropping files or listing http connections
- engine analysis?

**install suricata-update**
```
pip install pyyaml
pip install --pre --upgrade suricata-update

curl https://pastebin.com/raw/A8ShYngZ > /tmp/suricata_update.yaml.patch
patch -b /etc/suricata/suricata.yaml < /tmp/suricata_update.yaml.patch

curl https://pastebin.com/raw/ueEgbhTq >> /etc/suricata/disablesid.conf # disable ntp sigs

suricata-update enable-source sslbl/ssl-fp-blacklist
suricata-update enable-source ptresearch/attackdetection
suricata-update enable-source et/open
suricata-update
service suricata restart
```




















