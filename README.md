# Fedora RPM/Yum Repository Proxy and cache using Squid

This is the configuration I am using to create a local HTTP proxy and cache for Fedora updates using Squid. The setup is mostly immediately usable, except some minor configuration tweaks needed to personalize things for your own setup, like hostname and whatnot. 

As of this writing I have this running on a Synology NAS on my local network, it's a lightweight setup that doesn't require all that much horsepower, even on underpowered hardware like a Synology NAS it's working fine.

# How to launch

The included docker-compose.yml file is all you would need to start things up using Docker Compose. This of course means you need docker, but if you want you could also opt to install Squid directly on whatever system you have, and either use the included squid.conf directly, or use it for inspiration.

# How to use

## Switch repositories to using HTTP (not HTTPS)

We cannot cache HTTPS requests without complicating the setup quite a bit, so instead it's easier to simply modify the `baseurl` of each repository configured in `/etc/yum.repos.d`. By default Fedora usualy configures its repository configuration files with a `metalink` instead, which basically returns a bunch of mirrors that it then (somewhat randomly) picks from. I have instead modified all repositories to have a `baseurl` instead with some specific mirrors I want to make use of, and ensured that they're all `http`, not `https`.

For example, here's my `fedora.repo` changes:

```conf
#metalink=http://mirrors.fedoraproject.org/metalink?repo=fedora-$releasever&arch=$basearch
baseurl=http://ftp.riken.jp/Linux/fedora/development/$releasever/Everything/$basearch/os/
        http://coresite.mm.fcix.net/fedora/linux/development/$releasever/Everything/$basearch/os/
```

Do this for all repositories you use and/or wish to cache.

## Fedora Silverblue

For Silverblue there is an `rpm-ostreed` service that actually handles all the fetching and downloading of updates, which also supports making use of the standardized `curl` `http_header` environment variable. The easiest way is to create an override file that sets this environment variable specifically for this service:

```bash
sudo mkdir -p /etc/systemd/system/rpm-ostreed.service.d
echo "[Service]
Environment=\"http_proxy=http://10.0.0.5:8888\"" | sudo tee /etc/systemd/system/rpm-ostreed.service.d/http-proxy.conf
sudo systemctl daemon-reload
sudo systemctl restart rpm-ostreed
```

## Toolbox

Each Toolbox container will have their own `/etc/yum.repos.d` directory and repository configuration files, so there too you'll want to update their respective `baseurl` values, as mentioned above.

Separate to that, you can use a simple if check in your `~/.profile` or whatever appropriate file depending on what shell environment you're using to conditionally set the `http_proxy` environment variable. 

As an example, this is what I have within my `~/.config/fish/config.fish` (as I use Fish shell):

```conf
# Set http_proxy environment variables if we're inside a toolbox container
if [ -f /run/.toolboxenv ]
   export ALL_PROXY="http://10.0.0.5:8888"
   export http_proxy=$ALL_PROXY
end
```

## Fedora Workstation

For Workstation you have two ways that I can think of; either set the `http_proxy` environment variable system-wide (ie. in `~/.profile`, or via your network settings), or you can specify the `proxy=` value in each of the repository configuration files.