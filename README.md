btxcore
=======

This is Under's fork of Bitpay's Bitcore that uses bitcore 0.15.2. It has a limited segwit support.

It is HIGHLY recommended to use https://github.com/BTXexplorer/btxcore-deb to build and deploy packages for production use.

----
Getting Started
=====================================
Deploying btxcore full-stack manually:
----
````
##(add Unders key)##
$gpg --keyserver hkp://pgp.mit.edu:80 --recv-key B3BD190C
$sudo apt-get update
$sudo apt-get -y install libevent-dev libboost-all-dev libminiupnpc10 libzmq5 software-properties-common curl git build-essential libzmq3-dev
$sudo add-apt-repository ppa:bitcoin/bitcoin
$sudo apt-get update
$sudo apt-get -y install libdb4.8-dev libdb4.8++-dev
$curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash
##(restart your shell/os)##
$nvm install stable
$nvm install-latest-npm
$nvm use stable
##(install mongodb)##
$sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2930ADAE8CAF5059EE73BB4B58712A2291FA4AD5
$echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.6.list
$sudo apt-get update
$sudo apt-get install -y mongodb-org
$sudo systemctl enable mongod.service
##(install btxcore)##
$git clone https://github.com/BTXexplorer/btxcore.git
$npm install -g btxcore --production
````
Copy the following into a file named btxcore-node.json and place it in ~/.btxcore/ (be sure to customize username values(without angle brackets<>) and/or ports)
````json
{
  "network": "livenet",
  "port": 3001,
  "services": [
    "bitcored",
    "web",
    "insight-api",
    "insight-ui"
  ],
  "allowedOriginRegexp": "^https://<yourdomain>\\.<yourTLD>$",
  "messageLog": "",
  "servicesConfig": {
    "web": {
      "disablePolling": true,
      "enableSocketRPC": false
    },
    "insight-ui": {
      "routePrefix": "",
      "apiPrefix": "api"
    },
    "insight-api": {
      "routePrefix": "api",
      "coinTicker" : "https://api.coinmarketcap.com/v1/ticker/bitcore/?convert=USD",
      "coinShort": "RVN",
	    "db": {
		  "host": "127.0.0.1",
		  "port": "27017",
		  "database": "btx-api-livenet",
		  "user": "",
		  "password": ""
	  }
    },
    "bitcored": {
      "sendTxLog": "/home/<yourusername>/.btxcore/pushtx.log",
      "spawn": {
        "datadir": "/home/<yourusername>/.btxcore/data",
        "exec": "/home/<yourusername>/btxcore/node_modules/btxcore-node/bin/bitcored",
        "rpcqueue": 1000,
        "rpcport": 8556,
        "zmqpubrawtx": "tcp://127.0.0.1:28332",
        "zmqpubhashblock": "tcp://127.0.0.1:28332"
      }
    }
  }
}
````
Quick note on allowing socket.io from other services. 
- If you would like to have a seperate services be able to query your api with live updates, remove the "allowedOriginRegexp": setting and change "disablePolling": to false. 
- "enableSocketRPC" should remain false unless you can control who is connecting to your socket.io service. 
- The allowed OriginRegexp does not follow standard regex rules. If you have a subdomain, the format would be(without angle brackets<>):
````
"allowedOriginRegexp": "^https://<yoursubdomain>\\.<yourdomain>\\.<yourTLD>$",
````

To setup unique mongo credentials:
````
$mongo
>use btx-api-livenet
>db.createUser( { user: "test", pwd: "test1234", roles: [ "readWrite" ] } )
>exit
````
(then add these unique credentials to your btxcore-node.json)

Copy the following into a file named btx.conf and place it in ~/.btxcore/data
````json
server=1
whitelist=127.0.0.1
txindex=1
addressindex=1
timestampindex=1
spentindex=1
zmqpubrawtx=tcp://127.0.0.1:28332
zmqpubhashblock=tcp://127.0.0.1:28332
rpcport=8556
rpcallowip=127.0.0.1
rpcuser=bitcore
rpcpassword=local321 #change to something unique
uacomment=btxcore-sl

mempoolexpiry=72 # Default 336
rpcworkqueue=1100
maxmempool=2000
dbcache=1000
maxtxfee=1.0
dbmaxfilesize=64
````
Launch your copy of btxcore:
````
$btxcored
````
You can then view the bitcore block explorer at the location: `http://localhost:3001`

Create an Nginx proxy to forward port 80 and 443(with a snakeoil ssl cert)traffic:
----
IMPORTANT: this "nginx-btxcore" config is not meant for production use
see this guide [here](https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/) for production usage
````
$sudo apt-get install -y nginx ssl-cert
````
copy the following into a file named "nginx-btxcore" and place it in /etc/nginx/sites-available/
````
server {
    listen 80;
    listen 443 ssl;
        
    include snippets/snakeoil.conf;
    root /home/btxcore/www;
    access_log /var/log/nginx/btxcore-access.log;
    error_log /var/log/nginx/btxcore-error.log;
    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_connect_timeout 10;
        proxy_send_timeout 10;
        proxy_read_timeout 100; # 100s is timeout of Cloudflare
        send_timeout 10;
    }
    location /robots.txt {
       add_header Content-Type text/plain;
       return 200 "User-agent: *\nallow: /\n";
    }
    location /btxcore-hostname.txt {
        alias /var/www/html/btxcore-hostname.txt;
    }
}
````
Then enable your site:
````
$cd /etc/nginx/sites-enabled
$sudo ln -s ../sites-available/nginx-btxcore .
$sudo rm default
$sudo rm ../sites-available/default
$sudo printf "[Service]\nExecStartPost=/bin/sleep 0.1\n" | sudo tee /etc/systemd/system/nginx.service.d/override.conf
$sudo systemctl daemon-reload
$sudo service nginx restart
````
Upgrading btxcore full-stack manually:
----

- This will leave the local blockchain copy intact:
Shutdown the btxcored application first, and backup your unique btx.conf and btxcore-node.json
````
$cd ~/
$rm -rf .npm .node-gyp btxcore
$rm .btxcore/data/btx.conf .btxcore/btxcore-node.json
##reboot##
$git clone https://github.com/BTXexplorer/btxcore.git
$npm install -g btxcore --production
````
(recreate your unique btx.conf and btxcore-node.json)

- This will redownload a new blockchain copy:
(Some updates may require you to reindex the blockchain data. If this is the case, redownloading the blockchain only takes 20 minutes)
Shutdown the btxcored application first, and backup your unique btx.conf and btxcore-node.json
````
$cd ~/
$rm -rf .npm .node-gyp btxcore
$rm -rf .btxcore
##reboot##
$git clone https://github.com/BTXexplorer/btxcore.git
$npm install -g btxcore --production
````
(recreate your unique btx.conf and btxcore-node.json)

-Some upgrades may require you to rebuild the statistics database
````
$mongo
>use btx-api-livenet
>db.dropDatabase()
>exit
````

Undeploying btxcore full-stack manually:
----
````
$nvm deactivate
$nvm uninstall stable
$rm -rf .npm .node-gyp btxcore
$rm .btxcore
$mongo
>use btx-api-livenet
>db.dropDatabase()
>exit
````

## Applications

- [Node](https://github.com/BTXexplorer/btxcore-node) - A full node with extended capabilities using bitcore Core
- [Insight API](https://github.com/BTXexplorer/insight-api) - A blockchain explorer HTTP API
- [Insight UI](https://github.com/BTXexplorer/insight) - A blockchain explorer web user interface
- (to-do) [Wallet Service](https://github.com/BTXexplorer/btxcore-wallet-service) - A multisig HD service for wallets
- (to-do) [Wallet Client](https://github.com/BTXexplorer/btxcore-wallet-client) - A client for the wallet service
- (to-do) [CLI Wallet](https://github.com/BTXexplorer/btxcore-wallet) - A command-line based wallet client
- (to-do) [Angular Wallet Client](https://github.com/BTXexplorer/angular-btxcore-wallet-client) - An Angular based wallet client
- (to-do) [Copay](https://github.com/BTXexplorer/copay) - An easy-to-use, multiplatform, multisignature, secure bitcore wallet

## Libraries

- [Lib](https://github.com/BTXexplorer/btxcore-lib) - All of the core bitcore primatives including transactions, private key management and others
- (to-do) [Payment Protocol](https://github.com/BTXexplorer/btxcore-payment-protocol) - A protocol for communication between a merchant and customer
- [P2P](https://github.com/BTXexplorer/btxcore-p2p) - The peer-to-peer networking protocol
- (to-do) [Mnemonic](https://github.com/BTXexplorer/btxcore-mnemonic) - Implements mnemonic code for generating deterministic keys
- (to-do) [Channel](https://github.com/BTXexplorer/btxcore-channel) - Micropayment channels for rapidly adjusting bitcore transactions
- [Message](https://github.com/BTXexplorer/btxcore-message) - bitcore message verification and signing
- (to-do) [ECIES](https://github.com/BTXexplorer/btxcore-ecies) - Uses ECIES symmetric key negotiation from public keys to encrypt arbitrarily long data streams.

## Security

We're using btxcore in production, but please use common sense when doing anything related to finances! We take no responsibility for your implementation decisions.

## Contributing

Please send pull requests for bug fixes, code optimization, and ideas for improvement. For more information on how to contribute, please refer to our [CONTRIBUTING](https://github.com/BTXexplorer/btxcore/blob/master/CONTRIBUTING.md) file.

To verify signatures, use the following PGP keys:
- @BTXexplorer: http://pgp.mit.edu/pks/lookup?op=get&search=0x009BAB88B3BD190C `EE6F 9673 1EF6 ED85 B12B  0A3F 009B AB88 B3BD 190C`

## License

Code released under [the MIT license](https://github.com/BTXexplorer/btxcore/blob/master/LICENSE).
