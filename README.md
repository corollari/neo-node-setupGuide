# Setup Guide for NEO nodes
> A guide on how to set up a neo node with SSL certificate and a reverse proxy

## Why run your own node
TODO

## Choosing where to run the node
First of all it's important to mention that you should probably run the node in a cloud provider instead of locally for the following reasons:
- You'll be able to get a much higher uptime (outages...)
- It will eliminate the hassle of managing a lot of stuff that is directly managed by the cloud provider
- Most customer internet connections are under firewalls set up by ISPs, these firewalls prevent new incoming connections to the nodes, making it impossible for other nodes or RPC clients to initiate connections to your node

With that out of the way all that it's left is picking a provider and an instance type. In this guide we will go with a t2.large on AWS, following a [recommendation from CityOfZion](https://www.reddit.com/r/NEO/comments/7zx7ur/public_call_for_projects_launching_in_neo/), which we will complement with a 100GB storage volume on AWS EBS. You might pick the AWS region that best suits your needs, for example the one that is geographically closer to you or most of your users in order to reduce RTT time.

In this guide we will install Ubuntu 18.04 LTS on the cloud instance, the reasoning behind that choice is based on these facts:
- Ubuntu is used by a large amount of people which leads to any security updates being published promptly
- Ubuntu has all the packages needed during the guide directly available
- Using a LTS release should reduce breakage caused by updates

### Launching & setting up a t2.large instance
1. Go to [AWS' EC2 dashboard](https://console.aws.amazon.com/ec2/) & start the `Launch Instance` wizard

2. Select `Ubuntu Server 18.04 LTS`

3. Select `t2.large` and go to the next step

4. Skip to the next step

5. Modify the storage to `100 GiB`

6. Skip to the next step

7. Add the following security rules (do not remove the SSH rule added by default):

| Type     | Protocol  | Port Range | Source   |
|----------|-----------|------------|----------|
| HTTP     | TCP       | 80         | Anywhere |
| Custom TCP Rule     | TCP       | 10333 (20333 if testnet)  | Anywhere |
| HTTPS    | TCP       | 443        | Anywhere |

8. Launch the instance

9. Connect to the instance using `ssh -i Downloads/neo-node.pem ubuntu@1.2.3.4`, replacing 1.2.3.4 with your node's IP and `Downloads/neo-node.pem` with the path of your AWS' private key

## Installing the node
```bash
# Basic OS updating
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install libleveldb-dev sqlite3 libsqlite3-dev libunwind8-dev unzip nginx
sudo reboot
# Connect again through SSH
# Install neo-cli
wget https://github.com/neo-project/neo-node/releases/download/v2.10.3/neo-cli-linux-x64.zip # Download neo-cli
unzip neo-cli-linux-x64.zip
cd neo-cli
chmod +x neo-cli
```

## Downloading chain snapshots
Downloading a snapshot of the chain data is needed in order to reduce the startup time of the node to an acceptable level, here we are using the snapshots provided by NGD, which are usually updated regularly. To get these snapshots visit [sync.ngd.network](https://sync.ngd.network/), get the links of the packaged data and download them into your server.

The following lines provide example instructions to download these packages, but you should get new links from the page provided before, as that will reduce the time spent syncing. Also, remember to download only one of these packages, either the Mainnet or Testnet one, but not both.
```bash
wget https://packet.azureedge.net/neochain/mainnet/full/0-4765691/293B6BBE9E541A2FEF37654964EE8787/chain.acc.zip # Mainnet
wget https://packet.azureedge.net/neochain/testnet/full/0-3634399/DE680EF7CAB4646C725660F5B5A92F3C/chain.acc.zip # Testnet
```

## Installing the required plugins
```bash
./neo-cli # Run neo-cli
install SimplePolicy
install CoreMetrics
install RpcNep5Tracker
install ImportBlocks
install RpcSystemAssetTracker
install ApplicationLogs
install RpcWallet
exit
```
These instructions install all the required and recommended plugins, you can browse other optional plugins [here](https://docs.neo.org/docs/en-us/node/cli/config.html#downloading-plugins-from-github) and install them in the same way.

## Choosing the network
### MainNet
In the case of wanting to run the node on Mainnet skip this step and go directly to the next one, [Running the node](#running-the-node).

### Testnet
You'll need to run the following instructions to make the node be part of the Testnet network:
```bash
cp config.testnet.json config.json
cp protocol.testnet.json protocol.json
```

## Running the node
```bash
screen -S neo # Create a new screen
./neo-cli --rpc
show state
```
Now press `Ctrl-a` followed by `d` to detach the screen session, after which you can safely exit the ssh connection with `exit`.

All that's left now is to wait for the node to sync with the current state of the blokchain, you can check the current blockheight of the node by reattaching the screen with `screen -r neo`, which should be equal to the block height displayed on [CoZ's monitor](http://monitor.cityofzion.io/) when it has finished synching.

Following is a table of the time it took us to sync our nodes on the 4th Of January of 2020:

| MainNet   | TestNet   |
|:---------:|:---------:|
|  24 hours | 3.5 hours |

## Setting up a reverse proxy
**Note**: In this whole section you should replace all instances of `example.com` with the domain from which you plan to serve the RPC calls.

Create a new configuration file for nginx:
```bash
cd /etc/nginx/sites-enabled
sudo vi example.com.conf
```
Paste the following code inside the file:
```
server {
	server_name example.com;
	set $upstream 127.0.0.1:10332;
	location / {
		proxy_pass_header Authorization;
		proxy_pass http://$upstream;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_http_version 1.1;
		proxy_set_header Connection “”;
		proxy_buffering off;
		client_max_body_size 0;
		proxy_read_timeout 36000s;
		proxy_redirect off;
	}
	listen 80;
}
```
Remember to replace 10332 with 20332 in case of running a testnet node.

Finally, restart nginx:
```
sudo service nginx reload
```

## Getting SSL certificates
```bash
# Install certbot
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot python-certbot-nginx

# Run the certbot wizard
sudo certbot --nginx
```

Certbot automatically sets up a renewal timer so certificate renewal will be automatic and won't require any further work.

## Set up a supervisor
A supervisor will automatically restart your node in case it goes down or the server restarts. To install it create the following file:
```bash
sudo vi /etc/systemd/system/neoseed.service
```
And paste this code in there:
```
[Unit]
After=network-online.target
Requires=network-online.target
[Service]
WorkingDirectory=/home/ubuntu/neo-cli
ExecStart=/home/ubuntu/neo-cli/neo-cli --rpc
ExecStop=/bin/kill -SIGINT `ps ax | grep neo-cli | grep -v grep | awk ‘{print $1}’`
Restart=always
StandardInput=tty-force
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=neoseed
User=ubuntu
Group=ubuntu
[Install]
WantedBy=multi-user.target
```
Afterwards you only need to activate it:
```
sudo systemctl enable neoseed
sudo systemctl start neoseed
```

Special thanks to Alex Guba, as this section was taken from [on of his Medium posts](https://medium.com/@gubanotorious/creating-and-running-a-neo-node-on-microsoft-azure-in-under-30-minutes-ad8d79b9edf).

## Extra: Add your node to CoZ's monitor
1. Fork [neo-mon](https://github.com/CityOfZion/neo-mon)
2. Modify [mainnet.json](https://github.com/CityOfZion/neo-mon/blob/master/docs/assets/mainnet.json) and/or [testnet.json](https://github.com/CityOfZion/neo-mon/blob/master/docs/assets/testnet.json), adding your nodes to the list
3. Create a Pull Request to move your changes into the CityOfZion repo

## Appendix: Making your node ultra-secure
Follow [the standard](https://github.com/CityOfZion/standards/blob/master/nodes.md) created by CityOfZion to be used in the consensus nodes that they run.
