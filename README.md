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


## Installing the node
```bash
# Basic OS updating
sudo apt-get update
sudo apt-get upgrade
sudo reboot
# Connect again through SSH
# Install neo-cli
sudo apt-get install libleveldb-dev sqlite3 libsqlite3-dev libunwind8-dev unzip
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
install SimplePolicyPlugin
install CoreMetrics
install RpcNep5Tracker
install ImportBlocks
install RpcSystemAssetTrackerPlugin
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
./neo-cli
show state
```
Now press `Ctrl-a` followed by `d` to detach the screen session, after which you can safely exit the ssh connection with `exit`.

All that's left now is to wait for the node to sync with the current state of the blokchain, you can check the current blockheight of the node by reattaching the screen with `screen -r neo`, which should be equal to the block height displayed on [CoZ's monitor](http://monitor.cityofzion.io/) when it has finished synching.

Following is a table of the time it took us to sync our nodes on the 4th Of January of 2020:

| MainNet   | TestNet |
|:---------:|:-------:|
| ~36 hours |    -    |

## Getting SSL certificates
TODO

## Setting up a reverse proxy
TODO

## Appendix: Making your node ultra-secure
Follow [the standard](https://github.com/CityOfZion/standards/blob/master/nodes.md) created by CityOfZion to be used in the consensus nodes that they run.
