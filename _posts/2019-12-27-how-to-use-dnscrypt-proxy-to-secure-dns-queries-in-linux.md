---
# Global vars
layout: posts
title: How to use dnscrypt-proxy to secure DNS queries in Linux
date: 2019-12-27 14:00:00 +0300
published: true
tags: ["dnscrypt-proxy", "DNSCrypt", "DNS-over-HTTPS", "DoH", "DNS Security Extensions", "DNSSEC", "security", "hardening", "Linux", "Debian", "Ubuntu"]
# Theme specific vars
last_modified_at: 2019-12-27 14:00:00 +0300
thumbnail:
    name: dnscrypt-proxy_doh
    webp: true
summary: "This article describes how to install and configure dnscrypt-proxy to use DNSCrypt and DNS-over-HTTPS (DoH) with DNSSEC."
---
The article is intended for the following software versions:
+ OS: Linux with `systemd`
+ Package: `dnscrypt-proxy v2`
+ Tested on: Debian 10

Installation and configuration steps should be mostly applicable to distros using `systemd` and `dnscrypt-proxy v2`.

## What is DNSCrypt and DNS-over-HTTPS?
While browsing on the web, people almost always use the domain name instead of the IP address of the requested resource. However, the domain name must first be resolved to a corresponding IP address by a DNS resolver. In most cases, the DNS resolver is assigned by the Internet Service Provider (ISP). Moreover, even if the requested website uses HTTPS, DNS queries use a different protocol and they are sent in plain-text by default.

These problems are addressed by both DNSCrypt and DNS-over-HTTPS (DoH). To start with, DNSCrypt is the name of the protocol which provides encryption and authentication between the client and the DNS resolver. It uses elliptic curve cryptography. Like DNSCrypt, DNS-over-HTTPS also encrypts the data but using HTTPS protocol, as the name suggests. By using either one of them, people can significantly lower the probability of request and/or tampering, thus preventing man-in-the-middle attacks {% cite DNSCrypt_Wikipedia DNSoverHTTPS_Wikipedia --file web %}.

Apart from DNSCrypt and DoH, DNS data can be further secured by the use of DNS Security Extensions (DNSSEC). It provides DNS data authentication, data integrity and authenticated denial of existence. As a result, attacks such as DNS cache poisoning can be mitigated {% cite DNSSEC-Whatisitandwhyisitimportant_InternetCorporationforAssignedNamesandNumbersICANN --file web %}.

Considering the benefits, encrypting DNS queries is an important part of Linux hardening. In order to use DNSCrypt, DoH and DNSSEC, we are going to use `dnscrypt-proxy` as the client, which supports both protocols.

## Install and configure dnscrypt-proxy
+ `dnscrypt-proxy` exists both in Debian and Ubuntu repositories. Thus, you can install it via:
```bash
sudo apt-get update
sudo apt-get install dnscrypt-proxy
```
+ Check the `systemd` socket file by:
```bash
cat /lib/systemd/system/dnscrypt-proxy.socket
```
You should see the listening IP and the port, similar to:
```
[Socket]
ListenStream=127.0.2.1:53
ListenDatagram=127.0.2.1:53
NoDelay=true
DeferAcceptSec=1
```
In this case it is listening on `127.0.2.1` and port 53.
+ Make sure that nothing else is listening on the same address:port pair from the previous step by:
```bash
sudo ss -lp 'sport = :domain'
```
+ Edit dnscrypt-proxy configuration in `/etc/dnscrypt-proxy/dnscrypt-proxy.toml` to include:
```toml
dnscrypt_servers = true
doh_servers = true
require_dnssec = true
```
This will enable both DNSCrypt and DoH supporting servers. In addition, it requires the DNSSEC support.
Optionally, if you would like to use only the no-log and no-filter (parental filters, ad blocking etc.) resolvers, you can add:
```toml
require_nolog = true
require_nofilter = true
```
+ Backup the existing `resolv.conf` file:
```bash
sudo cp /etc/resolv.conf /etc/resolv.conf.backup
```
+ Remove the existing `resolv.conf`:
```bash
sudo rm -f /etc/resolv.conf
```
+ Create and edit `/etc/resolv.conf` to have:
```
nameserver 127.0.2.1 # the listening address in dnscrypt-proxy.socket file
options edns0
```
Here, `nameserver 127.0.2.1` tells that DNS queries are sent to the specified address, at which `dnscrypt-proxy.socket` is listening. If your socket file has a different address, you should configure `resolv.conf` accordingly. Also, `edns0` is required to be able to use DNSSEC.
+ Prevent network manager from changing the `resolv.conf`:
```bash
sudo chattr +i /etc/resolv.conf
```
At least in Debian and Ubuntu, the Network manager configures the `nameserver` automatically when a connection is established. Therefore, if you do not set an immutable bit on `resolv.conf`, it will be overridden.
+ Start and enable the `dnscrypt-proxy` via `systemd`:
```bash
sudo systemctl start dnscrypt-proxy.socket
sudo systemctl enable dnscrypt-proxy.socket
sudo systemctl start dnscrypt-proxy.service
sudo systemctl enable dnscrypt-proxy.service
```
+ Test if EDNS0 is active by:
```bash
drill rs.dns-oarc.net TXT
```
Look for "EDNS: version: 0" entry in the output.
+ Test if the `dnscrypt-proxy` selected resolvers are really using DNSSEC validation: [DNSSEC Resolver Test](https://dnssec.vs.uni-due.de/){:rel="nofollow"}.
+ As the last thing, you need to check if there is a DNS leak. This can be done via [DNSleaktest](https://dnsleaktest.com/){:rel="nofollow"}. You should only see the resolvers specified in the following list: [DNSCrypt Resolvers](https://download.dnscrypt.info/resolvers-list/v2/public-resolvers.md). By default, this is the list used by `dnscrypt-proxy` when selecting the best resolver, constrained by your requirements in the `/etc/dnscrypt-proxy/dnscrypt-proxy.toml` configuration file.

Installation and configuration sources: {% cite dnscrypt-proxy_ArchWiki DNSCrypt_InstallGentooWiki Dnscrypt-proxyWiki_GitHub --file web %}

## Using dnscrypt-proxy with captive portals
If you have applied the steps until this section, you will realize that captive portal logins, such as those in hotels, cafes and airports would not work. The reason is that usually, the hotspot needs to first redirect the client to an internal authentication page before allowing the access to the web. However, it cannot do this, because the network manager cannot override the `nameserver` in `resolv.conf` with the one from the router.

Thus, you need to temporarily remove the modifications made to `resolv.conf` and restart the network manager, then after logging in into the captive portal, you can safely re-enable DNSCrypt/DoH. Below is a bash script to achieve this goal:

{% highlight bash linenos %}
#!/bin/bash

# Exit script as soon as a command fails.
set -o errexit

RESOLV_CONF=/etc/resolv.conf
DNSCRYPT_RESOLV=/etc/resolv.conf.dnscryptBackup

sudo cp $RESOLV_CONF $DNSCRYPT_RESOLV
sudo chattr -i $RESOLV_CONF
sudo rm -f $RESOLV_CONF
echo "Restarting the NetworkManager"
sudo systemctl restart NetworkManager
echo "Wait for the connection"
echo "Then, login to the captive portal"

while true
do
    read -p "Did you login?(Y/N) " answer
    case $answer in
        [yY]* ) sudo rm -f $RESOLV_CONF
                sudo cp $DNSCRYPT_RESOLV $RESOLV_CONF
                sudo rm -f $DNSCRYPT_RESOLV
                sudo chattr +i $RESOLV_CONF
                echo "Final nameservers:"
                cat $RESOLV_CONF
                echo "Restarting dnscrypt-proxy..."
                sudo systemctl restart dnscrypt-proxy
                echo "Done"
                break;;

        [nN]* ) exit;;

        * )     echo "Enter Y or N, please.";;
    esac
done
{% endhighlight %}

## References
{% bibliography --file web --cited_in_order %}
