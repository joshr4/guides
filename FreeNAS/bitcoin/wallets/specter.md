[ [<< Back to Extras](https://github.com/seth586/guides/blob/master/FreeNAS/bitcoin/extras.md) ]

## Guide to ₿itcoin & ⚡Lightning️⚡ on 🦈FreeNAS🦈


![Specter](img/spectersmall.png) 
## Specter Daemon
[Specter](https://github.com/cryptoadvance/specter-desktop) is an electrum alternative that connects directly to bitcoin core. By hosting the daemon version on freenas, you can remotely generate, sign with coldcard, and broadcast PSBT transactions without having to install client side software. 

If you want USB connectivity for a trezor or ledger type of device, you will need to run specter desktop on the client machine as well to create the USB bridge.

Note: if you manually compiled bitcoind with the `--disable-wallet`, you will need to recompile with wallet functionality for specter to work.

Minimizing the amount of software that has file access to your lightning hot wallet is considered a security best practice, so lets [spin up a new jail](https://github.com/seth586/guides/blob/master/FreeNAS/bitcoin/freenas_1_jail_creation.md). I will name this jail `specter` and assign it an ip address of `192.168.84.11`. SSH in and lets begin!

```
root@freenas:~ # iocage console specter
root@specter:~ # pkg install python3 py37-pip nano
root@specter:~ # fetch https://github.com/cryptoadvance/specter-desktop/archive/v0.7.1.tar.gz
root@specter:~ # tar -xvf v0.7.1.tar.gz
root@specter:~ # pip-3.7 install -e specter-desktop*/.
```

Lets create a service, `mkdir /usr/local/etc/rc.d && nano /usr/local/etc/rc.d/specter`:
```
#!/bin/sh
#
# PROVIDE: specter
# REQUIRE: bitcoind tor
# KEYWORD:

. /etc/rc.subr

name="specter"
rcvar="specter_enable"

specter_command="/usr/local/bin/python3 -m cryptoadvance.specter server --host 0.0.0.0"
pidfile="/var/run/${name}.pid"
command="/usr/sbin/daemon"
command_args="-P ${pidfile} -r -f ${specter_command}"

load_rc_config $name
: ${lnd_enable:=no}

run_rc_command "$1"
```
Save (CTRL+O, ENTER) and exit (CTRL+X). Make the rc.d script executable, enable our service, and start the daemon: 

```
root@specter:~ # chmod +x /usr/local/etc/rc.d/specter
root@specter:~ # sysrc specter_enable="YES"
root@specter:~ # service specter start
```

Point a web browser to http://192.168.84.11:25441/, success! Notice that specter is not connected to bitcoin core yet, so lets create RPC credentials and configure.

## Bitcoind RPC authentication
Switch to your bitcoin jail and generate RPC credentials. Download the rpcauth tool as documented [here](https://github.com/bitcoin/bitcoin/tree/master/share/rpcauth). Save this information.
```
root@specter:~ # exit
root@freenas[~]# iocage console bitcoin
root@bitcoin:~ # fetch https://raw.githubusercontent.com/bitcoin/bitcoin/master/share/rpcauth/rpcauth.py
root@bitcoin:~ # python3 ./rpcauth.py specter
String to be appended to bitcoin.conf:
rpcauth=specter:5d0d70936350d0a79b588a9bb2906ea1$82afc2d29dfcfd808acd98f855cf47989564d8f1cd55b515f23fb10ace0dd75a
Your password:
2tm5NiN8wZVyjx_hgUL5O8it68WfoadHDEZ-v6w_RhQ=
```

Add the following lines to bitcoin config with `nano /usr/local/etc/bitcoin.conf`:
```
rpcauth=specter:5d0d70936350d0a79b588a9bb2906ea1$82afc2d29dfcfd808acd98f855cf47989564d8f1cd55b515f23fb10ace0dd75a
rpcallowip=192.168.84.0/24
rpcbind=0.0.0.0
blockfilterindex=1
```
Save (CTRL+O, ENTER) and exit (CTRL+X). Restart bitcoind & verify it is running:
```
root@bitcoin:~ # service bitcoind restart
root@bitcoin:~ # ps aux
```

## Specter web configuration
Navigate to the specter website http://192.168.84.11:25441/, and click on "Bitcoin Core Unavailable. Click to configure", add the following information:

`auto detect` = off

`username` = `specter`

`password` = `2tm5NiN8wZVyjx_hgUL5O8it68WfoadHDEZ-v6w_RhQ=` (as previously generated by `rpcauth.py`)

`host` = `http://192.168.84.208` (whatever your bitcoin jail is)

`port` = `8332`

Click "test", you should get green checkmarks. Click "save".

Read the docuemntation to get started! 

## upgrade specter
```
root@specter:~ # service specter stop
root@specter:~ # rm -r specter-desktop*
root@specter:~ # fetch https://github.com/cryptoadvance/specter-desktop/archive/v0.7.1.tar.gz
root@specter:~ # tar -xvf v0.7.1.tar.gz
root@specter:~ # pip-3.7 install -e specter-desktop*/. --upgrade
root@specter:~ # service specter start
```

[ [<< Back to Extras](https://github.com/seth586/guides/blob/master/FreeNAS/bitcoin/extras.md) ]