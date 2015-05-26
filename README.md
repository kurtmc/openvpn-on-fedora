# OpenVPN on Fedora 21

##### Step 1 — Install and Configure OpenVPN's Server Environment

Update system

```bash
# yum update
```

Then we can install OpenVPN and Easy-RSA.
```bash
# yum install openvpn easy-rsa
```

The example VPN server configuration file needs to be copied to /etc/openvpn
```bash
# cp /usr/share/doc/openvpn/sample/sample-config-files/server.conf /etc/openvpn/server.conf
```

open server.conf in a text editor. This tutorial will use Vim but you can use whichever editor you prefer.
```bash
# vim /etc/openvpn/server.conf
```

There are several changes to make in this file. You will see a section looking like this:
```
# If enabled, this directive will configure
# all clients to redirect their default
# network gateway through the VPN, causing
# all IP traffic such as web browsing and
# and DNS lookups to go through the VPN
# (The OpenVPN server machine may need to NAT
# or bridge the TUN/TAP interface to the internet
# in order for this to work properly).
;push "redirect-gateway def1 bypass-dhcp"
```
Uncomment push "redirect-gateway def1 bypass-dhcp" so the VPN server passes on clients' web traffic to its destination. It should look like this when done:
```
push "redirect-gateway def1 bypass-dhcp"
```

The next edit to make is in this area:
```
# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
;push "dhcp-option DNS 208.67.222.222"
;push "dhcp-option DNS 208.67.220.220"
```

Uncomment push "dhcp-option DNS 208.67.222.222" and push "dhcp-option DNS 208.67.220.220". It should look like this when done:
```
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"
```

This tells the server to push OpenDNS to connected clients for DNS resolution where possible. This can help prevent DNS requests from leaking outside the VPN connection. However, it's important to specify desired DNS resolvers in client devices as well. Though OpenDNS is the default used by OpenVPN, you can use whichever DNS services you prefer.

The last area to change in server.conf is here:
```
# You can uncomment this out on
# non-Windows systems.
;user nobody
;group nogroup
```

Uncomment both user nobody and group nogroup. It should look like this when done:
```
user nobody
group nogroup
```

By default, OpenVPN runs as the root user and thus has full root access to the system. We'll instead confine OpenVPN to the user nobody and group nogroup. This is an unprivileged user with no default login capabilities, often reserved for running untrusted applications like web-facing servers.

Now save your changes and exit Vim.

## Packet Forwarding
This is a sysctl setting which tells the server's kernel to forward traffic from client devices out to the Internet. Otherwise, the traffic will stop at the server. Enable packet forwarding during runtime by entering this command:
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

We need to make this permanent so the server still forwards traffic after rebooting.
```
vim /etc/sysctl.d/00-sysctl.conf
```

Add the following:
```
net.ipv4.ip_forward=1
```

Firewall
```
yum install firewalld
systemctl enable firewalld
systemctl start firewalld
firewall-cmd --add-service=openvpn
firewall-cmd --direct --passthrough ipv4 -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```

##### Step 2 — Creating a Certificate Authority and Server-Side Certificate & Key
OpenVPN uses certificates to encrypt traffic.

Configure and Build the Certificate Authority

It is now time to set up our own Certificate Authority (CA) and generate a certificate and key for the OpenVPN server. OpenVPN supports bidirectional authentication based on certificates, meaning that the client must authenticate the server certificate and the server must authenticate the client certificate before mutual trust is established. We will use Easy RSA's scripts we copied earlier to do this.

First copy over the Easy-RSA generation scripts.
```
cp -r /usr/share/easy-rsa/2.0 /etc/openvpn/easy-rsa

```

Then make the key storage directory.
```
mkdir /etc/openvpn/easy-rsa/keys
```

Easy-RSA has a variables file we can edit to create certificates exclusive to our person, business, or whatever entity we choose. This information is copied to the certificates and keys, and will help identify the keys later.

```
vim /etc/openvpn/easy-rsa/vars
```

change these
```
export KEY_COUNTRY="US"
export KEY_PROVINCE="TX"
export KEY_CITY="Dallas"
export KEY_ORG="My Company Name"
export KEY_EMAIL="sammy@example.com"
export KEY_OU="MYOrganizationalUnit"
```

In the same vars file, also edit this one line shown below. For simplicity, we will use server as the key name. If you want to use a different name, you would also need to update the OpenVPN configuration files that reference server.key and server.crt.

```
export KEY_NAME="server"
```

We need to generate the Diffie-Hellman parameters; this can take several minutes.
```
openssl dhparam -out /etc/openvpn/dh2048.pem 2048
```

Now let's change directories so that we're working directly out of where we moved Easy-RSA's scripts to earlier in Step 2.

```
cd /etc/openvpn/easy-rsa
```

Initialize the PKI (Public Key Infrastructure). Pay attention to the dot (.) and space in front of ./vars command. That signifies the current working directory (source).
```
. ./vars
```

The output from the above command is shown below. Since we haven't generated anything in the keys directory yet, the warning is nothing to be concerned about.
```
NOTE: If you run ./clean-all, I will be doing a rm -rf on /etc/openvpn/easy-rsa/keys
```

Now we'll clear the working directory of any possible old or example keys to make way for our new ones.
```
./clean-all
```

This final command builds the certificate authority (CA) by invoking an interactive OpenSSL command. The output will prompt you to confirm the Distinguished Name variables that were entered earlier into the Easy-RSA's variable file (country name, organization, etc.).

Simply press ENTER to pass through each prompt. If something must be changed, you can do that from within the prompt.

## Generate a Certificate and Key for the Server
Still working from /etc/openvpn/easy-rsa, now enter the command to build the server's key. Where you see server marked in red is the export KEY_NAME variable we set in Easy-RSA's vars file earlier in Step 2.

```
./build-key-server server
```

Similar output is generated as when we ran ./build-ca, and you can again press ENTER to confirm each line of the Distinguished Name. However, this time there are two additional prompts:
```
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

Both should be left blank, so just press ENTER to pass through each one.

Two additional queries at the end require a positive (y) response:
```
Sign the certificate? [y/n]
1 out of 1 certificate requests certified, commit? [y/n]
```

The last prompt above should complete with:
```
Write out database with 1 new entries
Data Base Updated
```

# Move the Server Certificates and Keys
OpenVPN expects to see the server's CA, certificate and key in /etc/openvpn. Let's copy them into the proper location.
```
cp /etc/openvpn/easy-rsa/keys/{server.crt,server.key,ca.crt} /etc/openvpn
```

You can verify the copy was successful with:
```
ls /etc/openvpn
```

You should see the certificate and key files for the server.

At this point, the OpenVPN server is ready to go. Start it and check the status.

```
systemctl start openvpn@server.service
systemctl enable openvpn@server.service
systemctl status openvpn@server.service
```

##### Step 3 — Generate Certificates and Keys for Clients

So far we've installed and configured the OpenVPN server, created a Certificate Authority, and created the server's own certificate and key. In this step, we use the server's CA to generate certificates and keys for each client device which will be connecting to the VPN. These files will later be installed onto the client devices such as a laptop or smartphone.

Key and Certificate Building

It's ideal for each client connecting to the VPN to have its own unique certificate and key. This is preferable to generating one general certificate and key to use among all client devices.

Note: By default, OpenVPN does not allow simultaneous connections to the server from clients using the same certificate and key. (See duplicate-cn in /etc/openvpn/server.conf.)

To create separate authentication credentials for each device you intend to connect to the VPN, you should complete this step for each device, but change the name client1 below to something different such as client2 or iphone2. With separate credentials per device, they can later be deactivated at the server individually, if need be. The remaining examples in this tutorial will use client1 as our example client device's name.

As we did with the server's key, now we build one for our client1 example. You should still be working out of /etc/openvpn/easy-rsa.

```
./build-key client1
```

Once again, you'll be asked to change or confirm the Distinguished Name variables and these two prompts which should be left blank. Press ENTER to accept the defaults.

```
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

As before, these two confirmations at the end of the build process require a (y) response:

```
Sign the certificate? [y/n]
1 out of 1 certificate requests certified, commit? [y/n]
```

If the key build was successful, the output will again be:
```
Write out database with 1 new entries
Data Base Updated
```

The example client configuration file should be copied to the Easy-RSA key directory too. We'll use it as a template which will be downloaded to client devices for editing. In the copy process, we are changing the name of the example file from client.conf to client.ovpn because the .ovpn file extension is what the clients will expect to use.

```
cp /usr/share/doc/openvpn/sample/sample-config-files/client.conf /etc/openvpn/easy-rsa/keys/client.ovpn
```

You can repeat this section again for each client, replacing client1 with the appropriate client name throughout.

Transferring Certificates and Keys to Client Devices

Recall from the steps above that we created the client certificates and keys, and that they are stored on the OpenVPN server in the /etc/openvpn/easy-rsa/keys directory.

For each client we need to transfer the client certificate, key, and profile template files to a folder on our local computer or another client device.

In this example, our client1 device requires its certificate and key, located on the server in:


```
/etc/openvpn/easy-rsa/keys/client1.crt
/etc/openvpn/easy-rsa/keys/client1.key
```

The ca.crt and client.ovpn files are the same for all clients. Download these two files as well; note that the ca.crt file is in a different directory than the others.

```
/etc/openvpn/easy-rsa/keys/client.ovpn
/etc/openvpn/ca.crt
```

While the exact applications used to accomplish this transfer will depend on your choice and device's operating system, you want the application to use SFTP (SSH file transfer protocol) or SCP (Secure Copy) on the backend. This will transport your client's VPN authentication files over an encrypted connection.

Here is an example SCP command using our client1 example. It places the file client1.key into the Downloads directory on the local computer.

```
scp root@your-server-ip:/etc/openvpn/easy-rsa/keys/client1.key Downloads/
```

At the end of this section, make sure you have these four files on your client device:
```
client1.crt
client1.key
client.ovpn
ca.crt
```
