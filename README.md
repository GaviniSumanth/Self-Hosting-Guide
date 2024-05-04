
# Self Hosting Guide

A simple guide for self hosting using podman and quadlets.

This guide is intended to consolidate the knowledge from various blogs, docs and forums into a single unified guide that is easy to understand and useful. It contains enough information to get you started but [Further Reading](#further-reading) is advised.

> You should consider looking at the [Security Recommendations](#security-recommendations) to find some known security related mistakes which people who are new to self hosting may make (including some mistakes that I made in the past) and how to fix them.
> There may be some other security issues which I may be unaware of so **due diligence is advised**.
>
> **NOTE:** If you find any issues with this guide please open a pull request so that the issues can be rectified.

## Why should I consider using podman when I can use docker?

- Podman containers are rootless. (i.e, they do not require root privileges and hence are more secure)
- Podman has a daemon-less architecture (i.e, it can run containers under the user starting the container)
- It is well integrated with the system and hence easier to manage if you are familiar with linux.
- They are lightweight.
- Podman CLI API is the same as that of docker so the commands are similar.
- It is compatible with docker-compose so you can migrate your existing configuration from docker to podman easily.

## DNS Setup

You should create the CNAME and ALIAS records as shown in the following figure. (The exact steps to change the records may vary based on the registrar but the records should be the same.)
![My Hostinger DNS Configuration](/images/DNS%20Configuration.png "My Hostinger DNS Configuration")
Create a username.duckdns.org record in DuckDNS and point it to your IP address.
![My Duck DNS Configuration](/images/Duck%20DNS%20Configuration.png "My Duck DNS Configuration")

On your router, you can either forward packets recieved on ports 80 and 443 to your PC or add your PC as a DMZ host. And set a static IP address for your PC on the LAN side.
![Static IP Configuration](/images/Static%20IP%20Configuration.png "Static IP Configuration")

## Get SSL Certificates for free with certbot

Caddy should be able to automatically handle the SSL part but if you cannot open port 80 (because it is blocked by your ISP) then you may need to do it manually using certbot.

It will ask you for some general information like email and domain name. Give those details and continue.

```bash
sudo certbot certonly --manual --preferred-challenges dns -d "*.domain.tld"
```

You will be asked to add a TXT record which you need to include in your DNS records.
![My Hostinger DNS Configuration](/images/DNS%20Configuration.png "My Hostinger DNS Configuration")

If successful, it will generate and give you the SSL Certificates which you need to add to include in your caddyfile.

## Caddy

### Caddy Installation

You can install caddy from your linux distribution's repository or use a podman container.

#### Method 1: To run caddy as a rootful container

Run it as a rootful container

You need to place the caddy container file in

```bash
/usr/share/containers/systemd/
```

or

```bash
/etc/containers/systemd/
```

Reload the systemd daemon

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl start caddy.service
```

#### Method 2

> A proxy server, kernel firewall rule, or redirection tool such as [redir](https://github.com/troglobit/redir) may be used to redirect traffic from a privileged port to an unprivileged one (where a podman pod is bound) in a server scenario - where a user has access to the root account (or setuid on the binary would be an acceptable risk), but wants to run the containers as an unprivileged user for enhanced security and for a limited number of pre-known ports.

#### Method 3: To run caddy as a rootless container

This method is not recommended because changing unprivileged ports start from the default 1024 may cause security issues.

For temporary testing:

```bash
sysctl net.ipv4.ip_unprivileged_port_start=80
```

This change will be undone after a restart.

To make it permanent:

Append this snippet to /etc/sysctl.conf

```bash
net.ipv4.ip_unprivileged_port_start=80
```

and then execute:

```bash
sudo sysctl -p
```

The rest of the steps are the same as that of any other rootless container so go to [Running Podman Containers](#running-podman-containers)

### Caddy Configuration

Take a look at the Caddyfile in this repo to check out how to serve a static page, set up a reverse proxy, logging etc.

## Running Podman Containers

### Step 1: Install Podman

On Debian/Ubuntu based systems:

```bash
sudo apt install podman podman-compose
```

On Fedora:
It is pre-installed.

### Step 2: Create a service user

You can change the SERVICE_USER variable to anything you like.

```bash
export SERVICE_USER="podman_user"
sudo useradd -r -m -d "/var/lib/${SERVICE_USER}" -s /bin/false "${SERVICE_USER}"
```

### Step 3: Set subuid ranges for the service user

```bash
NEW_SUBUID=$(($(tail -1 /etc/subuid |awk -F ":" '{print $2}')+65536))
NEW_SUBGID=$(($(tail -1 /etc/subgid |awk -F ":" '{print $2}')+65536))

sudo usermod \
--add-subuids ${NEW_SUBUID}-$((${NEW_SUBUID}+65535)) \
--add-subgids ${NEW_SUBGID}-$((${NEW_SUBGID}+65535)) \
  "${SERVICE_USER}"
```

### Step 4: Switch to the System User

```bash
sudo -H -u "${SERVICE_USER}" bash -c 'cd; bash'
```

### Step 5: Enable Linger for the service user

Run this with sudo from your main user account with sudo privileges

```bash
sudo loginctl enable-linger "${SERVICE_USER}"
```


### Step 6: Allow the service user to access the bus

```bash
echo "export XDG_RUNTIME_DIR=/run/user/$(id -u)" > ~/.bashrc
source ~/.bashrc
```

### Step 7: Run the Quadlet container

Place the quadlet container files in the user's systemd directory.

```bash
cp /path/to/container /var/lib/${SERVICE_USER}/.config/containers/systemd/
```

Reload the systemd daemon

```bash
systemctl --user daemon-reload
```

### Step 8: Auto start container on boot

Add this snippet to your container files to enable auto-start on boot

```ini
[Install]
WantedBy=default.target
```

## Quadlet Container Example

```ini
# vaultwarden.container
[Container]
ContainerName=vaultwarden
Image=registry.hub.docker.com/vaultwarden/server:latest
AutoUpdate=registry
PublishPort=127.0.0.1:8080:80
Volume=./data:/data

[Service]
Restart=always

[Install]
WantedBy=default.target
```

| Key           | Description                                                                            |
| :------------ | :------------------------------------------------------------------------------------- |
| ContainerName | Name of the container                                                                  |
| Image         | The image from which the container will be built                                       |
| AutoUpdate    | similar to io.containers.autoupdate label                                              |
| PublishPort   | To expose a port                                                                       |
| Volume        | To access a podman volume or to grant access to certain direcories on host file system |

## Security Recommendations

- Keep the system and containers updated.
- Use a firewall. I use firewalld through firewall-config's GUI as it allows me to add rich rules through a GUI which can be quite useful. If you prefer a simpler alternative then use gufw (Uncomplicated Firewall GUI).
- By default, the ports are published to 0.0.0.0:<PORT_NUMBER>, you may want to change it to 127.0.0.1:<PORT_NUMBER> (localhost) otherwise podman will expose those ports to other systems on the network.
- Scan your web server for any open ports using online tools or if you are an advanced user you can use nmap from the terminal on another system to check for open ports. If you are not sure about which online tool to use, I mentioned the one that I use to scan for open ports in the [Resources Section](#resources).
- If you do not intend to access the server through localhost and only use it via the registered domain name through the reverse proxy then remove the PublishPort line from container files (except the reverse proxy's container) and allow communication between the containers by adding all your hosted services and the reverse proxy that you use to a pod.
- If you want to block connections from certain countries or allow access to your server from certain whitelisted countries, you can use country IP block allocations from IPDeny.com.

## Common Questions

### Is it safe to self host on my PC at home?

As with any service or application that is exposed to the internet there are some risks involved but they can be mitigated to a great extent if you configure your system and server properly.

Alternatively, you can setup a VPN like Wireguard, Tailscale and Headscale to access your server remotely but this makes it somewhat harder to use as the user needs to have a client application otherwise you can use Cloudflare Tunnels.

### Do I need to register my domain name under a Domain registrar?

Well, strictly speaking you **do not need** to have a domain name but it can be very useful as you can have sub-domain routing and it saves you from the pain of having to type in the IP address of your server each time to access it and if you have DHCP on the WAN side then good luck trying to find your server's new IP address after your ISP changes it while you are away.

I recommend buying a cheap domain name (the price of domain names depends on their names and top level domain) and if you are hosting the server from your PC at home like I do and have DHCP on the WAN side then use duckdns (free of cost) to automatically update the IP whenever it gets changed.

### Which Domain registrar should I use?

From what I read online, if you want something **cheap** and are new to self hosting, go for **Hostinger** or if you want a **good** one go for **Cloudflare**. I use Hostinger because it is cheap and my server has very few users since my server is meant for private use only. So far, I don't have anything to complain about.

You can also look at other Domain registrars if you don't want Hostinger or Cloudflare for some reason.

### Which reverse proxy should I use?

This is a subjective question and it is up to you to choose which one you want to use.

I would recommend **caddy** as I think it has the right balance between ease of configuration and flexibility. It is easy to read and it prevents misconfiguration to a certain extent.

### I need my server for \<some_task\> so which service or application do I need to host?

Go to **Awesome Selfhosted** and find what you need. The link is in the [Resources Section](#resources).

### The application I want to host only has a docker-compose.yml file. How can I use it with podman?

You can use it directly with podman compose or use Podlet to convert the docker-compose.yml file into a Quadlet. You may need to make a few changes depending on the application but in most cases this should not be necessary.

### Can I generate my own SSL Certificates?

Yes, but I would not recommend it. If you create your own SSL records, you will have to add your Certificate Authority(CA) Certificate on every system where you access your server otherwise expect your browser to show warnings whenever you visit.

### What do the quadlets in this repo do?

| Container File           |      Name      | What it does                    |
| :----------------------- | :------------: | :------------------------------ |
| caddy.container          |     Caddy      | Reverse Proxy                   |
| audiobookshelf.container | Audiobookshelf | Audiobook and E-book Management |
| kavita.container         |     Kavita     | E-books and Comics              |
| vaultwarden.container    |  Vaultwarden   | Password Management             |

You can use the quadlets in this repo directly but I suggest you use it after you make the appropriate changes such as changing the volume mounts and exposed port numbers.

## Resources

- [IP Deny](https://www.ipdeny.com/ipblocks/)

    IPDeny.com provides downloads of country IP block allocations.

- [Podlet](https://github.com/containers/podlet)

    Podlet generates podman quadlet files from a podman command, compose file, or existing object.

- [pentest-tools.com](https://pentest-tools.com/network-vulnerability-scanning/port-scanner-online-nmap)

    You can use this for free port scans.

- [Awesome Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted)

    It is a list of Software services and web applications which can be hosted on your own server(s).

## Further Reading

- [Podman auto-update](https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html)
- [How to bind a rootless container to a privileged port on Linux](https://linuxconfig.org/how-to-bind-a-rootless-container-to-a-privileged-port-on-linux)
- [Shortcomings of Rootless Podman](https://github.com/containers/podman/blob/main/rootless.md)
