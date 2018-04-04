# Comments
It is very controversial to embed a password in a plain text file.

* In the testnet environment - there is no real risk
* In the mainnet environment - there is a risk. It would be up to the system manager to decide if to include this or not.

# Justification
Every time LND starts, the wallet is locked, and no lightning transactions can occur with this node. Examples of when LND may (re)start include:
* Power failure 
* External IP address change requires a restart of LND so that --externalip can be updated

To create a 'lights-off' (unattended) LND node, it is necessary that a fresh bootup results in the LND node being fully operational (i.e. with the wallet unlocked) with no human input.

# Procedure

* login as admin

* Install expect
  
  `admin ~ ฿ sudo apt-get install expect`

* Create the following script, edit with your Password_[C], save, and exit.

  `admin ~ ฿ sudo nano /home/bitcoin/.lnd/lnd_unlock.exp`

```
#!/usr/bin/expect
#
# File invoked by unlock_wallet.service to unlock the lnd wallet
# /home/bitcoin/.lnd/lnd_unlock.exp
#
# Change this line with your LND Wallet Password
set walletPW Password_[C]
#
set timeout 40
spawn lncli unlock
expect "Input wallet password: "
log_user 0
send "$walletPW\r"
log_user 1
expect "lnd successfully unlocked!"
```

* Create the unlock script, save, and exit

  `admin ~ ฿ sudo nano /home/bitcoin/.lnd/lnd_unlock.sh`

```
#!/bin/bash
# RaspiBolt LND: Script to unlock wallet
# /home/bitcoin/.lnd/lnd_unlock.sh

echo 'lnd_unlock started - monitoring every 10 mins'
lnd_dir="/home/bitcoin/.lnd"
while [ 0 ];
 do
 /usr/local/bin/lncli --macaroonpath=${lnd_dir}/readonly.macaroon --tlscertpath=${lnd_dir}/tls.cert getinfo 2>&1 | grep "identity_pubkey" >/dev/null
 wallet_unlocked=$?
 if [ "$wallet_unlocked" -eq 1 ] ; then
  /usr/bin/expect /home/bitcoin/.lnd/lnd_unlock.exp
  sleep 60
 else 
  sleep 600
 fi
done;
```

* Make it executable

  `admin ~  ฿  sudo chmod +x /home/bitcoin/.lnd/lnd_unlock.sh`

* Create the corresponding systemd unit, save, and exit.

  `admin ~ ฿ sudo nano /etc/systemd/system/unlock_wallet.service`

```
# Service to run looking to see if the LND wallet is locked (ie after every start/restart)
# /etc/systemd/system/unlock_wallet.service

[Unit]
Description=Unlock Wallet Service
After=lnd.service
Wants=lnd.service
PartOf=lnd.service

[Service]
User=admin
Group=admin
LimitNOFILE=65536
Type=simple
ExecStart=/bin/bash /home/bitcoin/.lnd/lnd_unlock.sh 2>&1 /dev/null
Restart=always
RestartSec=600
TimeoutSec=10

[Install]
WantedBy=multi-user.target
```

* Enable systemd startup

```
   admin ~ ฿ sudo systemctl enable unlock_wallet
   admin ~ ฿ sudo systemctl start  unlock_wallet
   admin ~ ฿ sudo systemctl status unlock_wallet
```

