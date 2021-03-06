[ [Intro](README.md) ] -- [ [Preparations](raspibolt_10_preparations.md) ] -- [ [Raspberry Pi](raspibolt_20_pi.md) ] -- [ [Bitcoin](raspibolt_30_bitcoin.md) ] -- [ [Lightning](raspibolt_40_lnd.md) ] -- [ [Mainnet](raspibolt_50_mainnet.md) ] -- [ [**Bonus**](raspibolt_60_bonus.md) ] -- [ [FAQ](raspibolt_faq.md) ] -- [ [Updates](raspibolt_updates.md) ]

------

### Beginner’s Guide to ️⚡Lightning️⚡ on a Raspberry Pi

------

## Bonus guide: Lightning web interface
*Difficulty: medium*

### Introduction

Having a personal full node is great, but the command line interface is just not for everyday usage. At the end of the day the RaspiBolt is a backend that enables additional applications to run without a 3rd party. In my opinion these are the most important components that exist today:

* Desktop wallet with hardware wallet integration for on-chain transactions: [Electrum Personal Server](raspibolt_64_electrum.md)
* Web interface for Lightning for desktop / mobile usage: <u>Lnd Web Client</u> (this guide)
* Mobile wallet for Lightning payments on the go: [Shango wallet](raspibolt_68_shango.md)

It would be great to have combined Bitcoin on-chain / Lightning wallets both for desktop and mobile usage. Rumors are that Electrum is working on it, and who knows, maybe Samourai will enable complete full node integration soon?

### Lnd Web Client

There are already several web interfaces available for LND. At the moment, the **[Lnd Web Client](https://github.com/mably/lncli-web/blob/master/README.md)** by François Masurel ([mably](https://keybase.io/mably)) seems a good choice, but it's still early days. 

![](images/60_lncliweb_gui.png)

Starting as the "admin" user, we add the new user "web", install Node.js, create new certificates, get the interface running and automate the startup.

#### Install Node.js

Installation and update of Node.js as described on [nodejs.org](https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions). I recommend to use Node.js 8, as version 10 can lead to issues. We also make sure that python is installed.

```
$ curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
$ sudo apt-get install nodejs python
$ sudo npm i npm@latest -g
```

#### Get Lnd Web Client

Add new user and download the source code

```
$ sudo adduser web
$ sudo su - web
$ git clone https://github.com/mably/lncli-web.git
$ cd lncli-web
$ npm install

# Run the following command, but NOT with the --force option. This would/will break stuff.
$ npm audit fix

$ exit
```

#### FIX - If you encounter a grpc error on npm install

You may receive a 404 error when attempting to download gRPC. Pre-built binaries not found for gRPC do not exist for ARM linux systems, this process is a bandaid fix to compile gRPC from source. Grab a coffee or a beer, it takes a little while.

```
$ npm --unsafe-perm install
```

#### Create new certificates

Unfortunately, LND autogenerated certificates are not compatible with the current NodeJS gRPC module implementation. We therefore need to generate new certificates. You need to enter a password (4+ characters), but we remove it again directly. 

```
$ sudo su - bitcoin
$ cd .lnd
$ openssl ecparam -genkey -name prime256v1 -out tls.key
$ openssl req -new -sha256 -key tls.key -out csr.csr -subj '/CN=localhost/O=lnd'
$ openssl req -x509 -sha256 -days 36500 -key tls.key -in csr.csr -out tls.cert
$ rm csr.csr
$ exit
```

:warning: **These certificates do not allow remote access** with `lncli` or Shango mobile wallet as configured with `tlsextraip` and `tlsextradomain`  in "lnd.config". To create extended certificates, please refer to the optional part "Certificates for remote usage" at the bottom.

The new certificates need to be distributed and LND is restarted to load the new configuration.

```
$ sudo cp /home/bitcoin/.lnd/tls.cert /home/admin/.lnd
$ sudo cp /home/bitcoin/.lnd/tls.cert /home/web/lncli-web/lnd.cert

#The way macaroons work has changed since LND version 0.5 therefore we have to check our version.
$ lnd --version
# If using a LND with version 0.5.0 or above:
$ sudo cp /home/bitcoin/.lnd/data/chain/bitcoin/mainnet/admin.macaroon /home/web/lncli-web/
# Else (if using a LND version lower then 0.5.0):
$ sudo cp /home/bitcoin/.lnd/admin.macaroon /home/web/lncli-web/

$ sudo chown web:web /home/web/lncli-web/*

# restart lnd, unlock the wallet and check the startup process
$ sudo systemctl restart lnd
$ lncli unlock
$ sudo journalctl -u lnd -f

# wait a bit and check lncli. If this works, the new certificates are ok:
$ lncli getinfo
```

#### Start Lnd Web Client

We configure the firewall for local network access only, generate self signed SSL certificates and start the web interface manually to test it. See [my comments in the original setup](raspibolt_20_pi.md#enabling-the-uncomplicated-firewall) on how you may need to adjust the ip subnet mask.

```
# still as "admin" user
$ sudo ufw allow from 192.168.0.0/24 to any port 8280 comment 'allow WebGUI from local network'
$ sudo ufw enable
$ sudo ufw status

# now as "web" user
$ sudo su - web
$ cd lncli-web

# generate self-signed SSL certificate
$ openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 36500
$ openssl rsa -in key.pem -out newkey.pem && mv newkey.pem key.pem

# start Lnd Web Client (change passwords!)
$ node server --usetls . --serverhost 0.0.0.0 --user manager --pwd YourOwnPassword --limituser lnd --limitpwd YourOwnPassword
```

#### Connect to Lnd Web Client

You now have a webserver running on your RaspiBolt and should be able to open the web application in a web browser on your regular computer (ajust the ip address):

https://192.168.0.20:8280/

:information_source: You probably get a warning as the SSL certificate is not signed by a trusted certificate authority. You can safely ignore this message (and maybe add permanent exception rule, depending on your browser).

There are two users:

* Standard user: can manage everything
* Limited user: has limited functionality, eg. no possibility to make payments

Make sure to test thoroughly before we proceed to automate the startup process.

The visualization library needed for the network graph is not installed. You could install it, but building the graph takes several hours and can crash your Pi. Not recommended, just use [Reckslplorer](https://lnmainnet.gaben.win/#) instead.

#### Keep Node Server Running

If you wish to keep the node server running in the background use:

```
$ node server --usetls . --serverhost 0.0.0.0 --user manager --pwd YourOwnPassword --limituser lnd --limitpwd YourOwnPassword > stdout.txt 2> stderr.txt &
```

#### Automate startup 

:warning: **This startup process does not work yet**. The server needs to be startet manually at the moment. Related github [issue #153](https://github.com/mably/lncli-web/issues/153).

* Create a new systemd unit for Lnd Web Client that is executed on startup. Copy and paste the following text block into the new file.  
  `$ sudo nano /etc/systemd/system/lncli-web.service` 

```
# RaspiBolt LND Mainnet: systemd unit for lncli-web
# /etc/systemd/system/lncli-web.service

[Unit]
Description=Lnd Web Client
Requires=lnd.service

[Service]
User=web
Group=web
ExecStart=/usr/bin/node /home/web/lncli-web/server.js --usetls /home/web/lncli-web --serverhost 0.0.0.0 --user manager --pwd YourOwnPW --limituser lnd --limitpwd YourOwnPW 
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

* Make sure to **use your own passwords**! Save and exit.
* Enable the configuration file  
   `$ sudo systemctl enable lncli-web`

### Optional: Certificates for remote usage

The certificates above do not support the LND options `tlsextraip` and `tlsextradomain` that are necessary to connect to the gRPC from a differenct location (eg. used for Shango wallet). Here's how you can create certificates that contain extra ip addresses and domain names. 

You can remove and add additional DNS and IP entries, just pay attention to the commas and line breaks. 

```
$ sudo su - bitcoin
$ cd .lnd
$ openssl ecparam -genkey -name prime256v1 -out tls.key
$ openssl req -new -sha256 \
            -key tls.key \
            -subj "/CN=localhost/O=lnd" \
            -reqexts SAN \
            -config <(cat /etc/ssl/openssl.cnf \
                <(printf "\n[SAN]\nsubjectAltName=\
                     DNS:localhost,\
                     DNS:ln.yourdomain.com,\
                     IP:127.0.0.1,\
                     IP:192.168.0.20,\
                     IP:11.22.33.44\
                 ")) \
            -out csr.csr
# verify the DNS and IP entries
$ openssl req -in csr.csr -text -noout

$ openssl req -x509 -sha256 -days 36500 \
            -key tls.key \
            -in csr.csr -out tls.cert \
            -extensions SAN \
            -config <(cat /etc/ssl/openssl.cnf \
                <(printf "\n[SAN]\nsubjectAltName=\
                     DNS:localhost,\
                     DNS:ln.yourdomain.com,\
                     IP:127.0.0.1,\
                     IP:192.168.0.20,\
                     IP:11.22.33.44\
                 "))
# verify the DNS and IP entries
$ openssl x509 -in tls.cert -text -noout
```

Copy these new certificates as described above to all locations and restart the lnd service. Now you should be able to connect remotely to your RaspiBolt again.

------

<< Back: [Bonus guides](raspibolt_60_bonus.md) 
