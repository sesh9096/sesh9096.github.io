---
layout: post
title:  "Setting up a wireguard vpn"
date: 2025-11-14 22:30:45 -07:00
---

# Introduction
Wireguard is a vpn protocol which according to their [website](https://www.wireguard.com/) is
> "an extremely simple yet fast and modern VPN that utilizes state-of-the-art cryptography"

I first became aware of wireguard as a protocol when reading through ProtonVPN's supported protocols.
I was recently attempting to set up a newly provisioned cloud virtual machine as a web server and then realized I didn't have much I wanted to run on a web server.
It was then that I remembered reading about wireguard and decided it might be fun to try and set up a wireguard tunnel.

Is this a good use of this machine? Probably not.
Will it be useful for anything? Hard to say, but unlikely.
With that out of the way, lets set up this thing.

# Wireguard Protocol Oversimplified
One of the key principles of Wireguard is to be as simple as possible, and one
way it achieves this is to use a small and opinionated set of cryptographic
algorithms:
- Elliptical curve Diffie-Hellman over Curve 25519 for key exchanges(exchanging
  information between 2 machines so that both may arrive at the same shared key
  but an attacker with only access to the shared data will be unable to find
  this key),
- ChaCha20-Poly1305 for encrypting and authenticating messages i.e. ensuring
  they cannot be read without a key and verifying they where sent by the
  supposed sender.
- BLAKE2 for hashing

All of these algorithms have been used in other applications such as SSH and SSL
and are known for being relatively fast without needing specialized hardware.

Wireguard sets up an network interface similar to those for Wi-fi or Ethernet
and then transmits messages via UDP between "peers".
Its canonical implementation exists in the linux kernel which makes things
easier for me since the server runs Ubuntu 24.04 and my own computer runs void
linux and uses Network Manager for network configuration.
# Setup
We first need to install the wireguard commands which are generally in the `wireguard-tools` or similar package.
To set up wireguard, we can use this snippet from the wg manpage:
``` sh
umask 077
wg genkey | tee private.key | wg pubkey > public.key
```
This generates a corresponding public and private key pair.
The private key identifies this device, so the public key must be copied to the other device and added to the list of authorized peers.
After running this command on both the server and client and then exchanging the public keys,
We must create configuration files for the interfaces, via systemd-networkd in the case of the Ubuntu server and Network manager in the case of my client.

*Note: we could have also used a preshared key (a symmetric key, so the same between machines) rather than public and private keys*

## Server
Since the server in this case uses systemd-networkd to configure network interfaces, we need to create the following 2 files in `/etc/systemd/network/`:
`wg0.netdev`
```
[NetDev]
Name = wg0
Kind = wireguard
Description = wireguard server interface

[WireGuard]
PrivateKey = <server-private-key>
ListenPort = 51280 # or other port of your choosing

[WireGuardPeer]
PublicKey = <client-public-key>
AllowedIPs = 10.1.0.2/32
```

`wg0.network`
```
[Match]
Name = wg0

[Network]
Address = 10.1.0.1/32

[Route]
Gateway = 10.1.0.1
Destination = 10.1.0.0/24
```
These files create an interface called wg0 and configure it so that packets can be forwarded to and from the interface.

Additionally, each wireguard client/peer needs to be assigned an IP address from
the subnet used for the interface, and in this case, we have assigned a subnet
of local IPv4 addresses which shouldn't cause any collisions with other devices.

After that, we need to tell systemd-networkd to reload its config or restart it via `systemctl restart systemd-networkd`.

The last step for the server was to punch some holes in the firewall, so that we could actually connect to the server.
My first step was to go into the cloud network security interface and allow traffic to the relevant ports to be routed to our server.
Then, I needed to allow incoming traffic over the public facing interface to the relevant ports, in this case udp via port 51280.
I did this via iptables, but this should be fairly simple with any other firewall configuration tool:
``` sh
sudo iptables-save -f /root/iptables.backup # not technically needed but always a good idea.
sudo iptables -A INPUT -m udp -p udp --dport 51820 -j ACCEPT
```
As it turned out, I would find out later that I had left out a couple of vital rules which caused some headaches later.
## Client
Since we will be using Network manager, we can use wireguard's config format
to configure the wireguard interface:

`wg-vpn.conf`
```
[Interface]
PrivateKey = <client-private-key>
Address = <client-address-on-server>/32

[Peer]
PublicKey = <server-public-key>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <server-public-ip:port>
```

After creating this file, we can import it with
`nmcli connection import type wireguard file wg-vpn.conf`.

Then we can connect to the server via `nmcli connection up wg-vpn`.

Success! The only problem is that our client now cannot seem to connect to the wider internet.
## Firewall headaches
After disconnecting from the vpn, I could see by running `sudo wg` on the server that my client had indeed connected to the server.
Thus, the issue was almost certainly with the firewall, and after some more research, I realized I probably needed to configure the server to forward packets.

This required first setting the relevant setting kernel packet forwarding and then adding some more firewall rules:
``` sh
sudo sysctl net/ipv4/ip_forward=1

sudo iptables -F FORWARD
sudo iptables -t nat -F POSTROUTING
sudo iptables -A FORWARD -i wg0 -o ens3 -j ACCEPT
sudo iptables -A FORWARD -i ens3 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 10.1.0.0/24 -o ens3 -j MASQUERADE
```
Then, we can use `iptables-save` to save the current firewall configuration and set the matching setting in in `/etc/sysctl.conf`. This time, connecting to our

# Conclusion
Now I have a working wireguard vpn which was probably more hassle to set up than what it is worth especially since the server does not seem to be connected to a very fast internet connection.
So, all in all, not particularly useful although I did learn quite a bit I didn't know about firewalls and which I am trying hard to forget.
## Resources:
[Wireguard (via systemd-networkd)](https://elou.world/en/tutorial/wireguard)
[Thomas Haller: WireGuard in NetworkManager]{https://blogs.gnome.org/thaller/2019/03/15/wireguard-in-networkmanager/}
