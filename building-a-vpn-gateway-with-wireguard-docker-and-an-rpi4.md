#Building a VPN gateway with Wireguard, Docker and an RPi4

I've recently encountered a problem; owing to how my ISP handles NAT, I couldn't play reliably online games on the Nintendo Switch. The problem is summarised in this [Nintendo Support article](https://www.nintendo.co.uk/Support/Nintendo-Switch/Unable-to-Connect-With-Others-Online-Errors-During-Matchmaking-Process--1512808.html). It seems to be a consequence of my primary broadband connection being a 4G connection and isn't limited to my ISP.

One way I found around this was to use another computer as a bridge to connect the Switch to a VPN network, but my initial approach, connecting a laptop to the router with an ethernet cable, connecting the laptop to a VPN network, creating an *ad hoc* wifi network to share that VPN connection and connecting the Switch to that *ad hoc* wifi network, was very disruptive and added a lot of friction to jumping on a quick game of Mario Kart.

As a more permanent solution, I set out to create an always-on Wireguard client within my home network which is connected to my off-site Wireguard server. This client would be configured to act as a gateway, forwarding traffic it receives on the local interface out over the Wireguard interface. Ideally I wanted a solution that would not entail tunnelling all of the host computer's traffic over the Wireguard interface. [^1] 

[^1]: Strictly speaking, Wireguard doesn't have servers and clients. I find it easier to think in those terms but in the end both the "server" and the "client" will be peers of one-another. The configuration of the peers determines what capabilities each will have. 

I already run an off-site Wireguard server, with site-to-site networking, allowing remote access to my home network and a secure route for traffic when using unsecured third-party wifi networks. I won't cover in this post the steps for setting up a Wireguard server to enable all of the client's traffic to be routed over the Wireguard interface and onwards to the internet, but many of the steps in this post are similar as they broadly achieve the same thing in reverse. Stavros Korokithakis has written a [good guide for how to do the initial set up ](https://www.stavros.io/posts/how-to-configure-wireguard/). 

I have a Raspberry Pi4, operating as a mini-server for some other services, which is always-on. I have recently been experimenting with Docker as a way of building in additional functionality to that server while keeping things broadly isolated. I won't cover installing and setting up Docker on the Raspberry Pi as this is well-covered in the [Docker documentation](https://docs.docker.com/engine/install/debian/#install-using-the-convenience-script).

I have figured this out with the help of many blog and forum posts from others. I have set out a list of the main references at the bottom. 

I'm pretty new to Docker and there may be errors or optimisations which I haven't found yet. Please do point them out, but you're responsible for making sure you're comfortable and for the outcome if you follow the same steps I have. There is probably a more efficient way to set everything up using a docker-compose but I haven't fully figured that out yet.

##Requirements

- Docker
- [linuxserver/wireguard Docker container](https://hub.docker.com/r/linuxserver/wireguard)
- [portainer/portainer-ce Docker container](https://hub.docker.com/r/portainer/portainer-ce)
- Wireguard running on an off-site server configured for IP forwarding.
- A Raspberry Pi4 (although these steps should work with any Linux-based computer) configured for SSH access and optionally configured with a static IP address (see below; this isn't required but will make things a little easier).

##Information to gather beforehand

- Home network IP subnet (e.g. 192.168.1.0/24).
- Home network gateway IP address (usually the IP address of your router, e.g. 192.168.1.1).
- An IP address within your home network IP subnet to use as a static IP for the router. If your home network has a DHCP server (usually one of the functions of your router), ideally configure the range of allocatable IP addresses to exclude a portion of the subnet e.g. addresses 192.168.1.2 - 192.168.1.149 are DHCP-assignable and 192.168.1.150 - 192.168.1.255 are not DHCP-assignable. This allows you to set an IP in the latter range as a static IP without concern than there will be a conflict with a DHCP-assigned IP address.
- The hostname or static IP address of your Wireguard server and its Wireguard port (usually 51820).
- The public key of your Wireguard server.

##Setting it all up
###Update host

Before starting, I recommend making sure your Raspberry Pi is up to date. Connect to the Raspberry Pi by SSH and run


`sudo apt-get update && sudo apt-get upgrade`

###Enable IP Forwarding on the host

Follow the steps set out [here](https://www.ducea.com/2006/08/01/how-to-enable-ip-forwarding-in-linux/) to enable IP forwarding on your host.

###Portainer

First, start a portainer container. Portainer is really powerful, and offers a nice interface for managing and making changes to your Docker containers. It also offers an easy way to get a console inside your Docker container to check things are running how you think they should be. The command is below, but I recommend watching the relevant section of [Steve's Tech Stuff's YouTube video](https://www.youtube.com/watch?v=FPuqK0shzuQ&t=295s) about setting up an ExpressVPN Docker container as this shows how to do the first-run after setting up portainer. Once the container is running, you should be able to navigate to http://[server-ip]:9000 in a browser to access portainer. linuxserver.io has a [good post](https://docs.linuxserver.io/general/understanding-puid-and-pgid) explaining what the PUID and PGID flags are doing. My understanding is these should be UID and GID of a user which is not an administrative user but which does have access rights to the host filesystem folders which are to be connected to the container. You can find your current user's PID and GID by running `id $user` in the shell.

```
docker run -d \
	--name=portainer \
	-e PUID=1000 \
	-e PGID=1000 \
	-p 8000:8000 \
	-p 9000:9000 \
	--restart=always \
	-v /var/run/docker.sock:/var/run/docker.sock \
	-v portainer_data:/data \
	portainer/portainer-ce
```

###Docker macvlan network
Next, we need to create a Docker [macvlan](https://docs.docker.com/network/macvlan/) network to allow clients on your home network to connect to the Docker container. I tried both [ipvlan](https://docs.docker.com/network/ipvlan/) level 2 and macvlan, but ipvlan did not work. The subnet should be the subnet of your home network e.g. 192.168.1.0/24. The gateway should be the IP address of your home network's gateway e.g. your router, such as 192.168.1.1. "parent" is the logical name of the host's network interface, which for me is eth0.  If you're unsure, you can check by running `ip addr` and looking for the interface which is attached to your home network. The final term "host_macvlan" is the name given to our new Docker network.

```
docker network create -d macvlan \
	--subnet=xxx.xxx.xxx.xxx/24 \
	--gateway=xxx.xxx.xxx.xxx \
	-o parent=eth0 host_macvlan
```

###Set up Wireguard Docker container
Next, we will set up the Wireguard Docker container and configure it to connect to our server. In your home directory, create a folder which will hold the configuration files for the container.


`mkdir ~/docker_wg_conf`

Next, create the Docker container and do the first run. This won't work to connect to the server yet as we haven't created a wg0.conf file, but first we need to generate the keypair for the new client. A few things to note about this command: 

1. Set the PUID and PGID for your user as with portainer.
2. Set TZ= to your [appropriate region](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).
3. Insert the full path of your `docker_wg_conf` folder created above as `/home/[user]/docker_wg_conf`.
4. Decide a static IP address for your new Docker container, which should be in your home network's subnet but not an address already used by any devices. Insert this as `-ip=xxx.xxx.xxx.xxx`
5. Note that the docker must be `--privileged` as otherwise there is an error with the `--sysctl="net.ipv4.conf.all.src_valid_mark=1"` command. Ideally this wouldn't need a privileged container but Wireguard insists on setting this and I haven't found a way to achieve that without giving *privileged* status to the container. To read more about this and decide if you want to proceed, I suggest the [Docker docs](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities) and [this Docker blogpost](https://www.docker.com/blog/docker-can-now-run-within-docker/) on what it means. I accepted it as if I were running the client outside of Docker Wireguard would also have these privileges. 

```
docker run -d \
  --name=wireguard \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Europe/London \
  -e ALLOWEDIPS=0.0.0.0/0 \
  -p 51820:51820/udp \
  -v [Full path to docker_wg_conf folder]:/config \
  -v /lib/modules:/lib/modules \
  --sysctl="net.ipv4.conf.all.src_valid_mark=1" \
  --restart unless-stopped \
  --net=host_macvlan --ip=xxx.xxx.xxx.xxx \
  --privileged \
  ghcr.io/linuxserver/wireguard
```

Once it's running, head over to portainer in your browser, select Containers and go into your newly created wireguard container. Check that its status is running and check the log for errors. You should see an error that there is no wg0.conf but that is fine for now. Assuming you have that, open a console in the container through portainer and change directory to the config directory `cd /config`, which is linked to your `docker_wg_conf` folder. Once there, we need to generate the keypair for this client:

```umask 077
wg keygen > privatekey
wg keygen < privatekey > publickey
```

Before we leave the container, run `ip addr` and make a note of the full interface logical name of your container's network interface on your home network, such as `eth0@if2`. We will need this in a second.

Once that's done, stop the wireguard container and open the terminal in your host machine (directly or via SSH). `cd ~/docker_wg_conf` and then run `ls` to confirm you can see the private key and public key files. If you can't something has gone wrong in one of the earlier steps. 

Create the wg0.conf file e.g. `nano ~/docker_wg_conf/wg0.conf` and fill in the details as follows, replacing square brackets with your values. A few things to note:

1. The PostUp and PostDown lines are very important. These create the iptables rules which will route the traffic from the home network to the Wireguard tunnel and vice versa.
2. AllowedIPs should be set to 0.0.0.0/0 so that all of the container's network traffic is routed over the Wireguard tunnel.
3. PersistentKeepalive helps to ensure that the tunnel remains active even when we are not using it. Every 25 seconds the client will handshake with the server. 
4. Make a note of the IP you assign to the client here, as we will need to update the server's wg0.conf with this and the client's publickey.


```
[Interface]
PrivateKey = [INSERT PRIVATE KEY from privatekey]
Address = [ASSIGN YOUR NEW CLIENT AN IP ON THE WIREGUARD SUBNET]/32
PostUp = iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE; iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o [CONTAINER'S INTERFACE e.g eth0@if2] -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE; iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o [CONTAINER'S INTERFACE e.g eth0@if2] -j MASQUERADE


[Peer]

PublicKey = [INSERT PUBLIC KEY from SERVER's publickey]
Endpoint = [INSERT SERVER'S PUBLIC IP ADDRESS AND WIREGUARD PORT E.G. xxx.xxx.xxx.xxx:51820]
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Save and close the file, then ssh into your Wireguard server and update its wg0.conf file with the new client. You will need the container's publickey, the IP address and suggest an `Endpoint=` although this last value will be updated automatically once the peers are connected. Remember you will need to take the Wireguard interface down `wg-quick down wg0`, update the configuration file and then bring the Wireguard interface back up `wg-quick up wg0`. If you edit the configuration file before bringing the interface down, your changes may be overwritten if you have `SaveConfig = true` set. 

Once the server's configuration has been updated, return to portainer and start the wireguard container. Check the log and it should now report the creation of the wg0 interface and the iptables rules in PostUp. Open a console into the container and run `curl ifconfig.co`. This will return the public IP address of the container, which should now be the public IP address of the server. 

Assuming everything has gone right up to this point, the container should now be running with Wireguard connected and the iptables rules created to forward traffic. The final step is to edit the network configuration of the Switch to use the new gateway. Navigate into your network configuration and change the settings. The Switch will need a static IP address, leave the subnet mask unchanged, and enter the home network private IP address of the wireguard container as the gateway. You also need to set DNS server settings manually; either direct this to your ISPs DNS server or use something like Cloudflare (1.1.1.1 and 1.0.0.1).