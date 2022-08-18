---
title: Configure Wireguard VPN for Remote Access
---

Wireguard is a simple, yet powerful VPN solution that is both fast and very easy
to setup in contrast to e.g. openVPN. It can be utilized to access your Home
Assistant instance securely from anywhere in the world.


## Prerequisites

- Home Assistant running on a Linux machine
- a machine with a static hostname/IP, e.g. a VPS in a public cloud


## Configuring the Server

Your server needs to run at least Kernel 5.6 to support Wireguard and you have
to install the wireguard userspace tools in addition to that. Find the
installation instructions for your operating system on the [Wireguard home
page](https://www.wireguard.com/install/).

Wireguard can be installed and configured on the system directly as follows.

1. Create the wireguard keys:
```ShellSession
 ❯ cd /etc/wireguard
 ❯ umask 077
 ❯ wg genkey | tee privatekey | wg pubkey > publickey
```
The public key can then be found in the file `/etc/wireguard/publickey` while the private
key has been written to `/etc/wireguard/privatekey`.

{{< hint type="caution" >}}
Keep the private key secure! Anyone with access to it can impersonate your
machine and gain access to your network.
{{< /hint >}}

2. Create the configuration file `/etc/wireguard/wg0.conf`:
```ini
[Interface]
Address = 10.200.200.1/24
ListenPort = 51820
PrivateKey = # insert /etc/wireguard/privatekey here
# optional
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

3. Start the wireguard tunnel via `systemctl enable --now wg-quick@wg0` and
   allow wireguard to be accessed through the firewall. For `firewalld` execute:
   `firewall-cmd --add-service=wireguard --permanent`.


## Configuring your Home Assistant Machine

We have to repeat the first step of the server setup of Wireguard on the machine
which is running Home Assistant as well, however our configuration file
`/etc/wireguard/wg0.conf` will be different:
```ini
[Interface]
PrivateKey = # insert /etc/wireguard/privatekey here
Address = 10.200.200.2/24
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o wlp59s0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o wlp59s0 -j MASQUERADE
ListenPort = 51820

[Peer]
PublicKey = # /etc/wireguard/publibkey from the server
Endpoint = # IP of the server
AllowedIPs = 10.200.200.0/24
PersistentKeepalive = 25 # optional, but recommended for reliable reconnects
```

You now have to register this peer on the server as well. Append the following
entry to `/etc/wireguard/wg0.conf` on the server:
```ini
[Peer]
PublicKey = # insert /etc/wireguard/publickey of Home Assistant
AllowedIPs = 10.200.200.2/32
PersistentKeepalive = 25 # optional, but recommended for reliable reconnects
```

After restarting wireguard on the server via `systemctl restart wg-quick@wg0`,
you can enable wireguard on this machine as well: `systemctl enable --now
wg-quick@wg0`.

Your Home Assistant machine and the server should now be able to reach each
other. You can verify this by e.g. pinging the server from Home Assistant via:
`ping 10.200.200.1` or by executing `wg show wg0` on either of the machines
which look roughly like the following:
```ShellSession
# wg show wg0
interface: wg0
  public key: VM9jO8G6pH43KIobViKFB9gk3Z9WK65oONLIAa8lEC8=
  private key: (hidden)
  listening port: 51820

peer: sA1VKDhTDYDsaEvDHGXhHJMd/nT9NTgD5ICO/DT6V2w=
  endpoint: # ENDPOINT IP will appear here
  allowed ips: 10.200.201.0/24
  latest handshake: 51 seconds ago
  transfer: 23.26 MiB received, 15.24 MiB sent
  persistent keepalive: every 25 seconds
```

## Adding peers to your Wireguard VPN

Adding a new peer to your VPN is the same process on the server irrespective of
the client OS. You have to append a new `[Peer]` entry to
`/etc/wireguard/wg0.conf` on the server:
```ini
[Peer]
PublicKey = # insert /etc/wireguard/publickey or the public key from the new peer
AllowedIPs = 10.200.200.N/32  # increment N from your last peer
PersistentKeepalive = 25 # optional, but recommended for roaming peers
```
and restart wireguard on the server.


### Linux

Adding Linux hosts is completely analogous to configuring the [Home Assistant
machine](./#configuring-your-home-assistant-machine). Follow the above guide,
but change the `Address` value in `/etc/wireguard/wg0.conf` to the same address
that you added to the server's `/etc/wireguard/wg0.conf`.

### Windows

### Android

Connecting your Android phone to the Wireguard network is very simple,
especially if you have access to a Linux machine. Then you can generate the
configuration for wireguard on the Linux machine and transfer it as a QR code to
your phone.

1. Install the Wireguard App from either the [Google Play
   Store](https://play.google.com/store/apps/details?id=com.wireguard.android)
   or [FDroid](https://f-droid.org/en/packages/com.wireguard.android/).

2. Generate a new configuration file as you would for a Linux host and save it
   somewhere on your file system, e.g. `/tmp/wg0.conf`.

3. Convert the configuration file into a qrcode using `qrencode`:
```ShellSession
qrencode -t ansiutf8 < "/tmp/wg0.conf"
```

4. Open the Wireguard App on your phone, click the plus sign in the bottom right
   corner, select "scan from QR code" and scan the QR code that appeared in your
   terminal.

5. Activate the connection in the Wireguard App using the toggle next to the
   tunnel name.


## Alternatives

If you do not own a VPS and do not want to run one as your wireguard server, you
can use [tailscale](https://tailscale.com/) to setup a wireguard network without
a central instance.

You can also run wireguard itself from within a container, if you do not want to
run it on your host system. This allows you to only allow certain containers to
have access to the wireguard network and can be used for a more declarative
system configuration. A good wireguard container is [Thorsten Kukuk's Wireguard
container based on openSUSE](https://github.com/thkukuk/wireguard-container) or
the [linuxserver/wireguard](https://hub.docker.com/r/linuxserver/wireguard)
image (this one does **not** function on non-Ubuntu/Debian systems!).
