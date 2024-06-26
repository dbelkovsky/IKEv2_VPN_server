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

## Brain start

### Installing StrongSwan

First, we’ll install `StrongSwan`, an open-source IPSec daemon which we’ll configure as our VPN server. We’ll also install the public key infrastructure (PKI) component so that we can create a Certificate Authority (CA) to provide credentials for our infrastructure.

For updating the local package cache:

```sh
sudo apt update
```

Then install the software by typing:

```sh
sudo apt install strongswan strongswan-pki libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins
```

_The additional `libcharon-extauth-plugins` package is used to ensure that various clients can authenticate to your server using a shared username and passphrase. The `libstrongswan-extra-plugins` package is included so that Strongswan supports elliptic curve cipher suites that use the `Curve25519` cryptography suite._

### Creating a Certificate Authority

An IKEv2 server requires a certificate to identify itself to clients. To help create the required certificate, the `strongswan-pki` package comes with a utility called `pki` to generate a Certificate Authority and server certificates.

To begin, let’s create a few directories to store all the assets we’ll be working on. The directory structure matches some of the directories in `/etc/ipsec.d.
Create followings directories and lock down the permissions so that our private files can’t be seen by other users::

```sh
mkdir -p ~/pki/{cacerts,certs,private}

chmod 700 ~/pki
```

Now that we have a directory structure to store everything, we can generate a root key. This will be a 4096-bit RSA key that will be used to sign our root certificate authority.

Use these commands to generate the key:

```sh
pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/ca-key.pem
```

Following that we can move on to creating our root certificate authority, using the key that we just generated to sign the root certificate:

```sh
pki --self --ca --lifetime 3650 --in ~/pki/private/ca-key.pem \
    --type rsa --dn "CN=VPN root CA" --outform pem > ~/pki/cacerts/ca-cert.pem
```

- The `--lifetime 3650` flag is used to ensure that the certificate authority’s root certificate will be valid for 10 years. The root certificate for an authority does not change typically, since it would have to be redistributed to every server and client that rely on it, so 10 years is a safe default expiry value.
- You can change the _distinguished name_ (DN) value to something else if you would like. The common name (CN field) here is just the indicator, so it doesn’t have to match anything in your infrastructure.
- Now that we’ve got our root certificate authority up and running, we can create a certificate that the VPN server will use.

### Generating a Certificate for the VPN Server

We’ll now create a certificate and key for the VPN server. This certificate will allow the client to verify the server’s authenticity using the CA certificate we just generated.

First, create a private key for the VPN server with the following command:

```sh
pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/server-key.pem
```

Now, create and sign the VPN server certificate with the certificate authority’s key you created in the previous step. Execute the following command, but change the Common Name (CN) and the Subject Alternate Name (SAN) field to your VPN server’s DNS name or IP address:

```sh
pki --pub --in ~/pki/private/server-key.pem --type rsa \
    | pki --issue --lifetime 1825 \
        --cacert ~/pki/cacerts/ca-cert.pem \
        --cakey ~/pki/private/ca-key.pem \
        --dn "CN=server_domain_or_IP" --san server_domain_or_IP \
        --flag serverAuth --flag ikeIntermediate --outform pem \
    >  ~/pki/certs/server-cert.pem
```

**If you are using an IP address instead of a DNS name, you will need to specify multiple --san entries. The line in the previous command block where you specify the distinguished name (--dn ...) will need to be modified with the extra entry like the following excerpted line:**

```sh
--dn "CN=IP address" --san @IP_address --san IP_address \
```

**The reason for this extra --san @IP_address entry is that some clients will check whether the TLS certificate has both an DNS entry and an IP Address entry for a server when they verify its identity.**

The `--flag serverAuth` option is used to indicate that the certificate will be used explicitly for server authentication, before the encrypted tunnel is established. The `--flag ikeIntermediate` option is used to support older macOS clients.

Now that we’ve generated all of the TLS/SSL files StrongSwan needs, we can move the files into place in the /etc/ipsec.d directory by typing:

```sh
sudo cp -r ~/pki/* /etc/ipsec.d/
```

In this step, we’ve created a certificate pair that will be used to secure communications between the client and the server. We’ve also signed the certificates with the CA key, so the client will be able to verify the authenticity of the VPN server using the CA certificate. With all of these certificates ready, we’ll move on to configuring the software.

### Configuring StrongSwan

StrongSwan has a default configuration file with some examples, but we will have to do most of the configuration ourselves. Let’s back up the file for reference before starting from scratch:

```sh
sudo mv /etc/ipsec.conf{,.original}
```

Create and open a new blank configuration file using your preferred text editor:

```sh
sudo vi /etc/ipsec.conf
```

**_As you work through this section to configure the server portion of your VPN, you will encounter settings that refer to left and right sides of a connection. When working with IPSec VPNs, the left side by convention refers to the local system that you are configuring, in this case the server. The right side directives in these settings will refer to remote clients, like phones and other computers. When you move on to configuring clients later in this tutorial, the client configuration files will refer to themselves using various left directives, and the server will be referred to using right side terminology._**

First, we’ll tell StrongSwan to log daemon statuses for debugging and allow duplicate connections. Add these lines to the file:

```
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no
```

Then, we’ll create a configuration section for our VPN. We’ll also tell StrongSwan to create IKEv2 VPN Tunnels and to automatically load this configuration section when it starts up. Append the following lines to the file:

```
. . .
conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
```

We’ll also configure dead-peer detection to clear any “dangling” connections in case the client unexpectedly disconnects. Add these lines:

```
. . .
conn ikev2-vpn
    . . .
    dpdaction=clear
    dpddelay=300s
    rekey=no
```

Next, we’ll configure the server’s “left” side IPSec parameters. Each of the following parameters ensures that the server is configured to accept connections from clients and to identify itself correctly. You’ll add each of these settings to the /etc/ipsec.conf file once you are familiar with what they are and why they are used:

- `left=%any` The `%any` value ensures that the server will use the network interface where it receives incoming connections for subsequent communication with clients. For example, if you are connecting a client over a private network, the server will use the private IP address where it receives traffic for the rest of the connection.
- `leftid=`**_@server_domain_or_IP_** This option controls the name that the server presents to clients. When combined with the next option `leftcert`, the `leftid` option ensures that the server’s configured name and the Distinguished Name (DN) that is contained in the public certificate match.
- `leftcert=server-cert.pem` This option is the path to the public certificate for the server that you configured in Step 3. Without it, the server will not be able to authenticate itself with clients, or finish negotiating the IKEv2 set up.
- `leftsendcert=always` The `always` value ensures that any client that connects to the server will always receive a copy of the server’s public certificate as part of the initial connection set up.
- `leftsubnet=0.0.0.0/0` The last “left” side option that you will add tells clients about the subnets that are reachable behind the server. In this case, `0.0.0.0/0` is used to represent the entire set of IPv4 addresses, meaning that the server will tell clients to send all their traffic over the VPN by default.

Now that you are familiar with each of the relevant “left” side options, add them all to the file like this:

```
. . .
conn ikev2-vpn
    . . .
    left=%any
    leftid=@server_domain_or_IP
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
```

**_When configuring the server ID (leftid), only include the @ character if your VPN server will be identified by a domain name:_**

```sh
    . . .    leftid=@vpn.example.com
    . . .
```

**_If the server will be identified by its IP address, just put the IP address in:_**

```sh
    . . .
    leftid=your_server_ip
    . . .
```

Next, we can configure the client’s “right” side IPSec parameters. Each of the following parameters tells the server how to accept connections from clients, how clients should authenticate to the server, and the private IP address ranges and DNS servers that clients will use. Add each of these settings to the `/etc/ipsec.conf` file once you are familiar with what they are and why they are used:

- `right=%any` The `%any` option for the `right` side of the connection instructs the server to accept incoming connections from any remote client.
- `rightid=%any` This option ensures that the server will not reject connections from clients that provide an identity before the encrypted tunnel is established.
- `rightauth=eap-mschapv2` This option configures the authentication method that clients will use to authenticate to the server. `eap-mschapv2` is used here for broad compatibility to support clients like Windows, macOS, and Android devices.
- `rightsourceip=10.10.10.0/24` This option instructs the server to assign private IP addresses to clients from the specified `10.10.10.0/24` pool of IPs.
- `rightdns=8.8.8.8,8.8.4.4` These IP addresses are Google’s public DNS resolvers. They can be changed to use other public resolvers, the VPN server’s resolvers, or any other resolver that clients can reach.
- `rightsendcert=never` This option instructs the server that clients do not need to send a certificate to authenticate themselves.

Now that you are familiar with the required “right” side options for the VPN, add the following lines to /etc/ipsec.conf:

```sh
. . .
conn ikev2-vpn
    . . .
    right=%any
    rightid=%any
    rightauth=eap-mschapv2
    rightsourceip=10.10.10.0/24
    rightdns=8.8.8.8,8.8.4.4
    rightsendcert=never
```

Now we’ll tell StrongSwan to ask the client for user credentials when they connect:

```sh
. . .
conn ikev2-vpn
    . . .
    eap_identity=%identity
```

Finally, add the following lines to support Linux, Windows, macOS, iOS, and Android clients. These lines specify the various key exchange, hashing, authentication, and encryption algorithms (commonly referred to as Cipher Suites) that StrongSwan will allow different clients to use:

```sh
. . .
conn ikev2-vpn
    . . .
    ike=chacha20poly1305-sha512-curve25519-prfsha512,aes256gcm16-sha384-prfsha384-ecp384,aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
    esp=chacha20poly1305-sha512,aes256gcm16-ecp384,aes256-sha256,aes256-sha1,3des-sha1!
```

Each supported cipher suite is delineated from the others by a comma. For example `chacha20poly1305-sha512-curve25519-prfsha512` is one suite, and `aes256gcm16-sha384-prfsha384-ecp384` is another. The cipher suites that are listed here are selected to ensure the widest range of compatibility across Windows, macOS, iOS, Android, and Linux clients.

The complete configuration file should look like this:

```sh
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid=@server_domain_or_IP
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    right=%any
    rightid=%any
    rightauth=eap-mschapv2
    rightsourceip=10.10.10.0/24
    rightdns=8.8.8.8,8.8.4.4
    rightsendcert=never
    eap_identity=%identity
    ike=chacha20poly1305-sha512-curve25519-prfsha512,aes256gcm16-sha384-prfsha384-ecp384,aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
    esp=chacha20poly1305-sha512,aes256gcm16-ecp384,aes256-sha256,aes256-sha1,3des-sha1!
```

Save and close the file once you’ve verified that you’ve added each line correctly. If you used nano, do so by pressing CTRL + X, Y, then ENTER.

Now that we’ve configured the VPN parameters, let’s move on to creating an account so our users can connect to the server.

### Configuring VPN Authentication

Our VPN server is now configured to accept client connections, but we don’t have any credentials configured yet. We’ll need to configure a couple things in a special configuration file called ipsec.secrets:

- We need to tell StrongSwan where to find the private key for our server certificate, so the server will be able to authenticate to clients.
- We also need to set up a list of users that will be allowed to connect to the VPN.
  Let’s open the secrets file for editing:

```sh
sudo vi /etc/ipsec.secrets
```

First, we’ll tell StrongSwan where to find our private key and how to parse it.

```sh
: RSA "server-key.pem"
```

Make sure that the line begins with the : character and that there is a space after it so that the entire line reads `: RSA "server-key.pem".`

Then, we’ll define the user credentials. You can make up any username or password combination that you like:

```sh
your_username : EAP "your_password"
```

Save and close the file. Now that we’ve finished working with the VPN parameters, we’ll restart the VPN service so that our configuration is applied:

```sh
sudo systemctl restart strongswan-starter
```

**Now that the VPN server has been fully configured with both server options and user credentials, it’s time to move on to configuring the most important part: the firewall.**

### Configuring the Firewall & Kernel IP Forwarding

With the StrongSwan configuration complete, we need to configure the firewall to allow VPN traffic through and forward it.

If you followed the prerequisite initial server setup tutorial, you should have a UFW firewall enabled. If you don’t yet have UFW configured, you should start by adding a rule to allow SSH connections through the firewall so your current session doesn’t close when you enable UFW:

```sh
sudo ufw allow OpenSSH

sudo ufw enable

sudo ufw allow 500,4500/udp
```

**_500 and 4500 is the standard IPSec ports for UDP traffic_**

Next, we will open up one of UFW’s configuration files to add a few low-level policies for routing and forwarding IPSec packets. Before we we can do this, though, we need to find which network interface on our server is used for internet access. Find this interface by querying for the device associated with the default route:

```sh
ip route show default
```

Your public interface should follow the word “dev”. For example, this result shows the interface named eth0, which is highlighted in the following example:

```sh
Output
default via your_server_ip dev eth0 proto static
```

When you have your public network interface, open the /etc/ufw/before.rules file in your text editor. The rules in this file are added to the firewall before the rest of the usual input and output rules. They are used to configure network address translation (NAT) so that the server can correctly route connections to and from clients and the Internet.

```sh
sudo nano /etc/ufw/before.rules
```

Near the top of the file (before the *filter line), add the following configuration block. Change each instance of eth0 in the above configuration to match the interface name you found with ip route. The *nat lines create rules so that the firewall can correctly route and manipulate traffic between the VPN clients and the internet. The \*mangle line adjusts the maximum packet segment size to prevent potential issues with certain VPN clients:

```sh
*nat
-A POSTROUTING -s 10.10.10.0/24 -o eth0 -m policy --pol ipsec --dir out -j ACCEPT
-A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE
COMMIT

*mangle
-A FORWARD --match policy --pol ipsec --dir in -s 10.10.10.0/24 -o eth0 -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
COMMIT

*filter
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]
. . .
```

Next, after the \*filter and chain definition lines, add one more block of configuration:

```sh
. . .
*filter
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]

-A ufw-before-forward --match policy --pol ipsec --dir in --proto esp -s 10.10.10.0/24 -j ACCEPT
-A ufw-before-forward --match policy --pol ipsec --dir out --proto esp -d 10.10.10.0/24 -j ACCEPT
```

These lines tell the firewall to forward ESP (Encapsulating Security Payload) traffic so the VPN clients will be able to connect. ESP provides additional security for our VPN packets as they’re traversing untrusted networks.

When you’re finished, ave and close the file once you’ve verified that you’ve added each line correctly. If you used nano, do so by pressing CTRL + X, Y, then ENTER.

Before restarting the firewall, we’ll change some network kernel parameters to allow routing from one interface to another. The file that controls these settings is called /etc/ufw/sysctl.conf. We’ll need to configure a few things in the file.

First IPv4 packet forwarding needs to be turned on so that traffic can move between the VPN and public facing network interfaces on the server. Next we’ll disable Path MTU discovery to prevent packet fragmentation problems. Finally we will not accept ICMP redirects nor send ICMP redirects to prevent man-in-the-middle attacks.

Open UFW’s kernel parameters configuration file using your preferred text editor:

```sh
sudo vi /etc/ufw/sysctl.conf
```

Now add the following net/ipv4/ip_forward=1 setting at the end of the file to enable forwarding packets between interfaces:

```sh
. . .
net/ipv4/ip_forward=1
```

Next block sending and receiving ICMP redirect packets by adding the following lines to the end of the file:

```sh
. . .
net/ipv4/conf/all/accept_redirects=0
net/ipv4/conf/all/send_redirects=0
```

Finally, turn off Path MTU discovery by adding this line to the end of the file:

```sh
. . .
net/ipv4/ip_no_pmtu_disc=1
```

Save the file when you are finished. Now we can enable all of our changes by disabling and re-enabling the firewall, since UFW applies these settings any time that it restarts:

```sh
sudo ufw disable
sudo ufw enable
```

You’ll be prompted to confirm the process. Type Y to enable UFW again with the new settings.
