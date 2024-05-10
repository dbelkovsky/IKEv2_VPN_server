# Introduction

A virtual private network, or VPN, allows you to securely encrypt traffic as it travels through untrusted networks, such as those at the coffee shop, a conference, or an airport.

Internet Key Exchange v2, or IKEv2, is a protocol that allows for direct IPSec tunneling between the server and client. In IKEv2 VPN implementations, IPSec provides encryption for the network traffic. IKEv2 is natively supported on some platforms (OS X 10.11+, iOS 9.1+, and Windows 10) with no additional applications necessary, and it handles client hiccups quite smoothly.

In this tutorial, you’ll set up an IKEv2 VPN server using StrongSwan on an Ubuntu 20.04 server. You’ll then learn how to connect to it with Windows, macOS, Ubuntu, iOS, and Android clients.

## Fast Start

If you want a quick start and are not in the mood to understand how everything works, this [script](https://github.com/dbelkovsky/IKEv2_VPN_server/blob/master/startswan.sh) will help you!

\*\*\*_This script is designed to automatically install the Strongswan daemon and all other preferences. Use Ubuntu 20.04. On it this script is tested and works well_

### Installation with script

Before running the script you must be login with a sudo!

In the process, this script will offer options for entering data for installation. Such as: FQDN/IP, network interface, user login/pass etc. All the rest it will do automatically and before the start of the service will display in the console the data for configuring client applications.

Use the following commands:

```bash
sudo -i

chmod u+x startswan.sh

./startswan.sh
```
