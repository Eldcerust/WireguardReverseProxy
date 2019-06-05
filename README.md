# WireguardReverseProxy

On your VPS and your client, ensure that you have done below.

```
apt-get update
apt-get upgrade
apt-get dist-upgrade
```

Then, install WireGuard through the following method.

```
sudo add-apt-repository ppa:wireguard/wireguard
sudo apt-get install wireguard
```

Execute this command both on the VPS and the client.

```
(umask 077 && printf "[Interface]\nPrivateKey = " | sudo tee /etc/wireguard/wg0.conf > /dev/null)
wg genkey | sudo tee -a /etc/wireguard/wg0.conf | wg pubkey | sudo tee /etc/wireguard/publickey
```

Take a note on both the public keys that is sent on the output above. These public keys will be exchanged later on the wg0.conf for client and server.

The VPS should have its /etc/wireguard/wg0.conf set as below:

```
[Interface]
PrivateKey = your VPS private key, will already be generated by the command above
ListenPort = *insert your port here that you want to connect for wireguard.*
Address = 192.168.4.1

[Peer]
PublicKey = *insert the key for client here*
AllowedIPs = 192.168.4.2/32
```

And the clients' /etc/wireguard/wg0.conf should be set as below:

```
[Interface]
PrivateKey = *auto generated*
Address = 192.168.4.2

[Peer]
PublicKey = *public key from vps*
AllowedIPs = 192.168.4.1/32
Endpoint = vps.external.ip.here:portdeclaredabove
PersistentKeepalive = 25
```

Run the command here below to start the wireguard service.
```
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

Make sure that the ping from both ends of the wireguard work. Possible problems include not having firewall allow udp connection from Wireguard as Wireguard will always use UDP configuration.

To forward the port from VPS to client, insert the IPtables below at the VPS only. Replace eth0 with whatever outward facing public IP the interface is found in the ifconfig.

```
sudo iptables -A FORWARD -i eth0 -o wg0 -p tcp --syn --dport 6667 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 6667 -j DNAT --to-destination 192.168.4.2
sudo iptables -t nat -A POSTROUTING -o wg0 -p tcp --dport 6667 -d 192.168.4.2 -j SNAT --to-source 192.168.4.1
```

Do not forget to enable port input and output from both client and server side with this syntax.

```
sudo iptables -I INPUT -p tcp --dport 6667 --syn -j ACCEPT
```

For different port configuration, do the following:

```
sudo iptables -A FORWARD -i eth0 -o wg0 -p tcp --syn --dport 22 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 22222 -j DNAT --to-destination 192.168.4.2:22
sudo iptables -t nat -A POSTROUTING -o wg0 -p tcp --dport 22 -d 192.168.4.2 -j SNAT --to-source 192.168.4.1
```

Put the port enabled as below:

```
sudo iptables -I INPUT -p tcp --dport 22 --syn -j ACCEPT  -------> Client
sudo iptables -I INPUT -p tcp --dport 22222 --syn -j ACCEPT  --------> Server
```


Since iptables is non-persistent by default, do not forget to setup persistence through this set of commands:

```
sudo apt-get install netfilter-persistent
sudo netfilter-persistent save
sudo systemctl enable netfilter-persistent
sudo systemctl start netfilter-persistent
```

Everytime a change is made on iptables that has been finalized, please do the following command:
```
sudo netfilter-persistent save
```
