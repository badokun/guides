[ [Intro](README.md) ] -- [ [Preparations](raspibolt_10_preparations.md) ] -- [ [Raspberry Pi](raspibolt_20_pi.md) ] -- [ [Bitcoin](raspibolt_30_bitcoin.md) ] -- [ [Lightning](raspibolt_40_lnd.md) ] -- [ **c-lightning** ] --  [ [Mainnet](raspibolt_50_mainnet.md) ] -- [ [Bonus](raspibolt_60_bonus.md) ] -- [ [Troubleshooting](raspibolt_70_troubleshooting.md) ]

-------
### Beginner’s Guide to ️⚡Lightning️⚡ on a Raspberry Pi
--------

# Lightning: c-lightning

There are multiple implementations for Lightning, LND currently is more dominant but c-lightning is a good alternative and since it's written in c is more performant, so a good choice for low powered devices. It's worth pointing out, currently c-lightning requires a full node and pruned nodes are not yet supported.

> IMPORTANT NOTE: This guide will use port `9536` instead of the default `9535`, so that both LND and c-lightning nodes can run concurrently

## Installation

As the `admin` user perform the following

```bash
sudo apt update
sudo apt -y upgrade
sudo apt-get install -y autoconf automake build-essential git libtool libgmp-dev libsqlite3-dev python python3 net-tools zlib1g-dev
```

Find the latest release branch [here](https://github.com/ElementsProject/lightning/releases/latest). Currently it's at [v0.7.0](https://github.com/ElementsProject/lightning/releases/tag/v0.7.0)

```bash
cd ~
git clone https://github.com/ElementsProject/lightning.git
cd lightning
git checkout v0.7.0
```

Compiling the source code. Get yourself a coffee, this may take a few minutes.

## Compiling and installing

```bash
./configure
make
sudo make install
```

Create the LND working directory and the corresponding symbolic link

```bash
sudo su - bitcoin
mkdir /mnt/hdd/lightning
ln -s /mnt/hdd/lightning /home/bitcoin/.lightning
ls -la
```

Create the c-lightning node's configuration file

```bash
nano ~/.lightning/config
```

## Configuration

### Get your RaspiBolt's external IP address

```bash
curl ipinfo.io/ip
```

Paste the following text and save (remember to change your `alias`). Update `PASSWORD_[B]` to the same one used in the [Bitcoin](raspibolt\raspibolt_30_bitcoin.md) configuration section. Update the `announce-addr` value to the IP address in the above step including the port `9736`.

> Reminder: This guide is using port `9736` instead of the default `9735`

```bash
daemon
network=bitcoin
alias=YOUR_NAME [LND]
rgb=000000
bitcoin-rpcuser=raspibolt
bitcoin-rpcpassword=PASSWORD_[B]
log-level=debug
log-file=/home/bitcoin/.lightning/debug.log
announce-addr=160.21.220.220:9736
bind-addr=0.0.0.0:9736
```

### Opening the firewall

```bash
sudo su
sudo ufw allow 9736 comment 'allow c-lightning'
```

### Configure router

Open port `9736` on your router's firewall and do a port-forward to your RaspiBolt.

## Create lightningd service

Make sure you are using the `admin` user. If you followed the steps above you'll need to run `exit` to exit the `bitcoin` user's session.

Create the service definition and paste the text below.
```bash
sudo nano /etc/systemd/system/lightningd.service
```

```bash
[Unit]
Description=c-lightning Daemon
Requires=bitcoind.service
After=bitcoind.service

[Service]
ExecStart=/usr/local/bin/lightningd --pid-file=/home/bitcoin/.lightning/lightning.pid
PIDFile=/home/bitcoin/.lightning/lightning.pid
User=bitcoin
Type=forking
Restart=on-failure
# Hardening measures
####################
# Provide a private /tmp and /var/tmp.
PrivateTmp=true
# Mount /usr, /boot/ and /etc read-only for the process.
ProtectSystem=full
# Disallow the process and all of its children to gain
# new privileges through execve().
NoNewPrivileges=true
# Use a new /dev namespace only populated with API pseudo devices
# such as /dev/null, /dev/zero and /dev/random.
PrivateDevices=true
# Deny the creation of writable and executable memory mappings.
MemoryDenyWriteExecute=true
[Install]
WantedBy=multi-user.target
```

Enable the service startup

```bash
sudo systemctl enable lightningd
```

Time to start the service

```bash
sudo systemctl start lightningd
```

## Verification

Confirm all is in working order

```bash
sudo su - bitcoin
lightning-cli getinfo | jq
```

Keep an eye on the logs to see how things are progressing

```bash
tail -n 100 -f ~/.lightning/debug.log
```

## Connecting to nodes

Unlike LND, c-lightning does not have the auto-pilot feature so you'll need to connect to nodes and open channels to them manually.

Here's a few useful `lightning-cli` commands

```bash
lightning-cli getinfo
              listfunds
              listconfigs
              listpeers
              listpayments
```

> You can find public nodes on the [1ml.com](https://1ml.com/node?order=capacity) website

## Spark Web UI

Once you get familiar with the cli you may want to switch over to a Web based interface, e.g. [Spark guide](https://medium.com/@notgrubles/spark-a-new-gui-for-c-lightning-2cf2f024500c) adapted for a RaspiBolt installation. GitHub project is located [here](https://github.com/shesek/spark-wallet)

As the `admin` user, execute the following commands

### !! [[ Remove this ] Add github's SSH key

Follow the steps outlined [here](https://help.github.com/en/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
to generate and then [here](https://help.github.com/en/articles/adding-a-new-ssh-key-to-your-github-account) to add it to your github profile.

```bash
cd
sudo apt-get update
sudo apt-get upgrade
sudo apt install nodejs
sudo npm install -g npm
```

Confirm `npm` version is at least `6.8.0`. If it's not look at this [guide](https://github.com/maddox/harmony-api/issues/84#issuecomment-357018266)

```bash
npm --version
```

Install Spark Wallet

#### From source
```bash
cd && mkdir spark && cd spark
git clone https://github.com/shesek/spark-wallet && cd spark-wallet
```

Get the latest [release version](https://github.com/shesek/spark-wallet/releases)

```bash
git checkout v0.2.5
npm install
```

```bash
sudo su - bitcoin
cd /home/admin/spark/spark-wallet
npm start --ln-path /mnt/hdd/lightning
```

#### Directly

```bash
npm install -g spark-wallet

```

### Open firewall

```bash
sudo su
sudo ufw allow from 192.168.1.0/24 to any port 9737 comment 'allow spark from Local LAN'
```

## Upgrading

```bash
cd ~
cd lightning
git pull
git checkout v0.7.0
```

Stop your c-lightning node before compiling and installing the newly built binaries. See the *Compiling and installing* section above.

```bash
sudo su - bitcoin
lightning-cli stop
```

## Reference material

- [Lightning Network node (c-lightning) on RBP3](https://medium.com/@meeDamian/c-lightning-node-on-rbp3-b950660fb835)

bit.ly/2U9OeUM