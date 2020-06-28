# OpenVPN
```
# Open VPN 
# Ubuntu 16.04

sudo apt-get update
sudo apt-get upgrade

sudo apt-get install openvpn easy-rsa
gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/server.conf

nano /etc/openvpn/server.conf
# Remove the “;” to uncomment lines or add new lines for the following entries in the configuration file.
 tls-auth ta.key 0
 key-direction 0
 cipher AES-256-CBC
 auth SHA256
 comp-lzo
 user nobody
 group nogroup
 cert server.crt
 key server.key
 push "redirect-gateway def1 bypass-dhcp"
 push "dhcp-option DNS 208.67.222.222"
 push "dhcp-option DNS 208.67.220.220"
 
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sudo sysctl -p

sudo modprobe iptable_nat
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

make-cadir /etc/openvpn/openvpn-ca/
cd /etc/openvpn/openvpn-ca/

nano vars
# Modify values
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="TecAdmin"
export KEY_EMAIL="info@example.com"
export KEY_OU="Security"

source vars
./clean-all
./build-ca

cd /etc/openvpn/openvpn-ca/
./build-key-server server

openssl dhparam -out /etc/openvpn/dh2048.pem 2048
openvpn --genkey --secret /etc/openvpn/openvpn-ca/keys/ta.key

cd /etc/openvpn/openvpn-ca/keys
sudo cp ca.crt ta.key server.crt server.key /etc/openvpn

sudo systemctl start openvpn@server
sudo systemctl status openvpn@server

ifconfig tun0

mkdir /etc/openvpn/clients
cd /etc/openvpn/clients

nano make-vpn-client.sh
#!/bin/bash
 
# Generate OpenVPN clients configuration files.
 
CLIENT_NAME=$1
OPENVPN_SERVER="192.168.1.237"
CA_DIR=/etc/openvpn/openvpn-ca
CLIENT_DIR=/etc/openvpn/clients
 
cd ${CA_DIR}
source vars
./build-key ${CLIENT_NAME}
 
echo "client
dev tun
proto udp
remote ${OPENVPN_SERVER} 1194
user nobody
group nogroup
persist-key
persist-tun
cipher AES-128-CBC
auth SHA256
key-direction 1
remote-cert-tls server
comp-lzo
verb 3" > ${CLIENT_DIR}/${CLIENT_NAME}.ovpn
 
cat <(echo -e '<ca>') \
    ${CA_DIR}/keys/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${CA_DIR}/keys/${CLIENT_NAME}.crt \
    <(echo -e '</cert>\n<key>') \
    ${CA_DIR}/keys/${CLIENT_NAME}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${CA_DIR}/keys/ta.key \
    <(echo -e '</tls-auth>') \
    >> ${CLIENT_DIR}/${CLIENT_NAME}.ovpn
 
echo -e "Client File Created - ${CLIENT_DIR}/${CLIENT_NAME}.ovp

chmod +x ./make-vpn-client.sh
./make-vpn-client.sh vpnclient1

openvpn --config client1.ovpn
```
