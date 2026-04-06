# Wireguard
The VPN. Apart from protecting your data, wireguard can be handily used to remotely connect to a remote machine or a remote (local) network.

Note: The ip addresses might differ depending on your router.

## Basic setup
Assuming you are on an debian/ubuntu-based system.
We assume there are two devices:
- The host (or server) - the device running wireguard
- The client - the device that connects to the host (and to host's local network)
Note: It is recommended to set a constant IP for the host.

Install wireguard on both host and client: 
```bash
sudo apt update
sudo apt install wireguard
```


### Host
Now, we generate the private and public keys for the client. The command to generate wireguard keys is:
```bash
sudo mkdir -p /etc/wireguard                                                                                                                 ─╯
sudo bash -c 'umask 077; wg genkey > /etc/wireguard/server_private.key'
sudo bash -c 'wg pubkey < /etc/wireguard/server_private.key > /etc/wireguard/server_public.key'

sudo chown root:root /etc/wireguard/server_private.key /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key
sudo chmod 644 /etc/wireguard/server_public.key
```


To create the server config, put the following into `/etc/wireguard/wg0.conf`:
```ini
[Interface]
Address = 10.200.200.1/24
ListenPort = 51820
PrivateKey = <server_privkey>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o wlp109s0f0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o wlp109s0f0 -j MASQUERADE

[Peer]
PublicKey = <client_pubkey>
AllowedIPs = 10.200.200.2/32
```
Note: You can add as many `[Peer]` blocks as many client machines you want to be connecting from.

To start wireguard:
```bash
sudo systemctl start wg-quick@wg0
```
To check status:
```bash
sudo systemctl status wg-quick@wg0
sudo wg show
```
To stop wireguard:
```
sudo systemctl stop wg-quick@wg0
```

**IP fowarding**
Wireguard is generally considered safe, but expose your ports to the internet at your own risk ... tho none of this will work without that, of course.

You need to do this in the "Port Forwarding" tab on your router (likely `192.168.1.1` or `192.168.2.254` in your browser). The login credentials are at the back of the router. Then, you want to:
- Use the UDP protocol
- Set internet port number to 51820
- Set destination IP address to your host ip address (you can check that by running `hostname -I` in the terminal)
- Set destination port number to 51820

Note: Running the following commands might or might not be necessary depending on your network/system etc.
```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system
```

### Client
Now, we generate the private and public keys for both the host. The command to generate wireguard keys is:
```bash
sudo mkdir -p /etc/wireguard                                                                                                                 ─╯
sudo bash -c 'umask 077; wg genkey > /etc/wireguard/client_private.key'
sudo bash -c 'wg pubkey < /etc/wireguard/client_private.key > /etc/wireguard/client_public.key'

sudo chown root:root /etc/wireguard/client_private.key /etc/wireguard/client_public.key
sudo chmod 600 /etc/wireguard/client_private.key
sudo chmod 644 /etc/wireguard/client_public.key
```

To start wireguard:
```bash
sudo systemctl start wg-quick@wg0
```
To check status:
```bash
sudo systemctl status wg-quick@wg0
sudo wg show
```
To stop wireguard:
```
sudo systemctl stop wg-quick@wg0
```


Create the client-side config in `/etc/wireguard/wg0.conf`:
```ini
[Interface]
Address = 10.200.200.2/32
PrivateKey = <client_privkey>
DNS = 1.1.1.1

[Peer]
PublicKey = <server_pubkey>
Endpoint = your-home-ddns-or-ip:51820
AllowedIPs = 10.200.200.0/24
PersistentKeepalive = 25
```



## Android Setup (Client only)
Install [the android app](https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en).

To create the interface section on Android, click the "+" and then "Create from scratch", and name it, e.g.: "home-vpn".
Tap generate on the private key (to generate the private key, duh). 
Pick a private ip address different from your other clients, here, we are going to use `10.200.200.3/32`.
Note: The DNS can be left empty.

Next, tap "Add peer", enter your:
- `<SERVER_PUBLIC_KEY>` in the "public key",
- `<YOUR_PUBLIC_IP>:51820` in the "Endpoint", and
- `10.200.200.0/24, 192.168.2.0/24` in the "Allowed IPs". Additionally, it is recommended (tho I am not sure on the actual effect) to put 25 into "Persistent Keepalive".

Make sure to add a `[Peer]` section on the host matching the phone's ip into `/etc/wireguard/wg0.conf`. E.g., for the values above:
```ini
[Peer]
PublicKey = <ANDROID_PUBLIC_KEY>
AllowedIPs = 10.200.200.3/32
```
And restart wireguard: `sudo systemctl restart wg-quick@wg0`.
