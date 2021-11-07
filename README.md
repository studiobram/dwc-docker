# DWC Docker Deployment

This provides a Docker setup, which enables you to run your own Multiplayer Server for several Nintendo Wii Games (Mario Kart Wii, SSBB, etc.) and connect to it using a vpn.

## Requirements

This has only been tested on a raspberry pi 2b.

 - Docker - ```sudo apt install docker.io```
 - Docker Compose - ```sudo apt install docker-compose```

## Network Setup - Server

Make sure that your server is reachable via the following ports:

| Protocol | Port  | Service                    |
|----------|-------|----------------------------|
| TCP      | 80    | WebServer                  |
| TCP      | 8000  | StorageServer              |
| TCP      | 9000  | NasServer                  |
| TCP      | 9001  | InternalStatsServer        |
| TCP      | 27500 | GameSpyManager             |
| TCP      | 28910 | GameSpyServerBrowserServer |
| TCP      | 29900 | GameSpyProfileServer       |
| TCP      | 29901 | GameSpyPlayerSearchServer  |
| TCP      | 29920 | GameSpyGamestatsServer     |
| UDP      | 27900 | GameSpyQRServer            |
| UDP      | 27901 | GameSpyNatNegServer        |

The following ports are optional:

| Protocol | Port  | Service                    |
|----------|-------|----------------------------|
| TCP      | 9003  | Dls1Server                 |
| TCP      | 9009  | AdminPage                  |
| TCP      | 9998  | RegisterPage               |

## Network Setup - Clients

Remember that you need a Dolphin Client with the correct modifications to be able to join a spoofed server.
Every client systems also needs the correct DNS records to be able to connect to your server. These can be provided by adding them to the Hosts file of each client:

```
10.13.13.2		gamestats.gs.nintendowifi.net
10.13.13.2		gamestats2.gs.nintendowifi.net
10.13.13.2		gpcm.gs.nintendowifi.net
10.13.13.2		gpsp.gs.nintendowifi.net
10.13.13.2		mariokartwii.available.gs.nintendowifi.net
10.13.13.2		mariokartwii.gamestats.gs.nintendowifi.net
10.13.13.2		mariokartwii.gamestats2.gs.nintendowifi.net
10.13.13.2		mariokartwii.master.gs.nintendowifi.net
10.13.13.2		mariokartwii.ms19.gs.nintendowifi.net
10.13.13.2		mariokartwii.natneg1.gs.nintendowifi.net
10.13.13.2		mariokartwii.natneg2.gs.nintendowifi.net
10.13.13.2		mariokartwii.natneg3.gs.nintendowifi.net
10.13.13.2		mariokartwii.sake.gs.nintendowifi.net
10.13.13.2		naswii.nintendowifi.net
10.13.13.2		nas.nintendowifi.net
10.13.13.2		nintendowifi.net
10.13.13.2		wiimmfi.de
```

## Installation
Port forward the port 51820/udp to you pi

```
git clone https://github.com/studiobram/dwc-docker
git checkout release/vpn
cd dwc-docker

docker run -d \
  --name=wireguard \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Europe/London \
  -e SERVERURL=[Your ip]\
  -e SERVERPORT=51820 \
  -e PEERS=dwc,user1,user2,user3,user4,user5,user6,user7,user8,user9,user10,user11,user12, \
  -e INTERNAL_SUBNET=10.13.13.0 \
  -e ALLOWEDIPS=10.13.13.0/24 \
  -p 51820:51820/udp \
  -v ./wireguard/config:/config \
  -v /lib/modules:/lib/modules \
  --sysctl="net.ipv4.conf.all.src_valid_mark=1, net.ipv4.ip_forward=1" \
  --restart unless-stopped \
  linuxserver/wireguard
  
# Copy the content from /wireguard/config/peer_dwc/peer_dwc.conf to /wireguard/wg0.conf, you might want to change the ip of the endpoint to the local ip of your pi to make it a bit quicker
# Add the following code to every wireguard .conf you are gone use
# [Interface]
# PostUp = ping -c1 10.13.13.1
# [Peer]
# PersistentKeepalive = 25

sudo docker-compose up -d

Afterwords connect to the wireguard vpn on pi with the configuration found in /wireguard/config/peer_user[1-12]/peer_user[1-12].conf
```

Persistent data is stored in a Docker-managed volume. With a usual Docker installation, you should find the databases in ```/var/lib/docker/volumes/dwc_data/_data```.

## Other tidbits

- Some of the HTTP requests from Dolphin contain a duplicate "Host" Header for whatever reason. An up-to-date NGINX complains, logs this as an INFO error (does not show up by default in error_log) and does not proxy the request to the server. This could be solved via some rewrite magic, for now we are using an older version of NGINX.

- Services running in containers on the localhost interface will not be able to communicate with other container services. Therefore these services now run on all interfaces, so the NGINX <--> server communication can take place.