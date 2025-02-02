# Home NAS

To run on Raspberry Pi 5 with the Radxa Penta SATA hat.

## To deploy

(Note: `podman-compose` doesn't work with `podman-remote`. I should have used
`docker-compose` I guess. But maybe I have to switch to k8s later anyway).

(OK no, `docker-compose` also sucks, it doesn't really support remote control in
a sensible way either, you just expose your Docker daemon socket over TCP? Are
all these simplified container tools just toys for babies? Do I have to use
k8s?)

Once:

```
sudo sysctl net.ipv4.ip_unprivileged_port_start=80

# TODO: This is dumb, podman-compose is dumb.
ssh $HOST mkdir -p nas/system-data
ssh $HOST touch nas/system-data/filebrowser.db
```

```
sudo zpool create -m /mnt/nas-data -f nas-pool raidz /dev/sda /dev/sdb /dev/sdc /dev/sdd
sudo zfs set acltype=posixacl nas-pool
sudo zfs set xattr=sa nas-pool
```

```
# https://github.com/canonical/docker-snap
sudo snap install docker
sudo addgroup --system docker
sudo adduser $USER docker
newgrp docker
sudo snap disable docker
sudo snap enable docker
```

To deploy

```
rsync --exclude=system-data/ -avz . $HOST:nas && ssh $HOST "cd nas/; docker-compose up"
```

## Notes and plan

What I need to run:

- ZFS for the main storage
- FileBrowser
- MinIO
- Samba
- smartmontools (for disk status) + smartctl_exporter (to export to Prometheus)
- Prometheus for SMTP alerts via GMail
- A reverse proxy that takes care of SSL and can route to the other HTTP
  services as needed.

The plan

- [x] Figure out how to run FileBrowser in `podman-compose`
- [x] Figure out basics of Caddy as a reverse proxy
- [x] Figure out how to add Tailscale. I only want the bare minimum of services.

      Answer: I think the solution is: don't really have the container engine be
      aware of tailscale - just configure that in the OS and then have Caddy act
      as a "firewall" by being aware of the [Tailscale CGNAT](https://tailscale.com/kb/1015/100.x-addresses) addresses and only routing to "internal" services from those.

- [x] Add MinIO
- [x] Figure out how to expose everything on Tailscale, but only the bare
      minimum on the internet

      ChatGPT and DeepSeek R1 gave me slightly differenta approaches for this.

      ChatGPT:

      ```
      # Public Service
      example.com {
            reverse_proxy http://service-public:8080
      }

      # Private Service (Tailscale Only)
      @internal {
            remote_ip 100.64.0.0/10
      }
      private.example.com {
            route {
                  abort
            }
            route @internal {
                  reverse_proxy http://service-private:8080
            }
      }
      ```

      DeepSeek R1:

      ```
      # Public Service (accessible to all)
      public.example.com {
          reverse_proxy localhost:8080
      }

      # Internal Service (Tailscale only)
      internal.example.com {
          @deny not remote_ip 100.64.0.0/10
          handle @deny {
              respond "Forbidden" 403
          }
          reverse_proxy localhost:8081
      }
      ```
- [x] Disable port forwarding to avoid exposing insecure FileBrowser defaults
- [x] Check for data on the old RAID fs, back it up
- [x] Install Ubuntu Server on the Pi
- [x] Create ZFS
  - [x] Create pool
  - [x] Ensure it's auto-mounted
  - [x] Figure out ACLs for the filesystem on it
- [ ] ZFS mounted exposed to FileBrowser
- [ ] Make NAS services start up on boot
- [ ] Add Prometheus
- [ ] Configure Prometheus SMTP
- [ ] Switch back to proper Let's Encrypt certs once rate limite recovers.
- [ ] Add smartmontools
- [ ] Plumb SMART data into Prometheus
- [ ] Add alerts :
  - [ ] for SMART data
  - [ ] for ZFS status
- [ ] Add Samba
- [ ] If I run out of memory on the Pi, port this to MicroK8s so I can scale it
      horizontally.
- [ ] Think about a way to avoid exposing random OSS projects to the internet on
      my home network. My ideal way to do this would be to use Google as an SSO
      provider, I think this is possible but looks like quite a faff:

      - [Example with Traefik](https://www.smarthomebeginner.com/google-oauth-traefik-forward-auth-2024/)
      - [Example with Caddy](https://beneaththeradar.blog/caddy-with-google-oauth2/)

      Alternatively I could run something like Authelia on the network, there is
      a well-lit path for Caddy to do this. Then I would have exposed Authelia
      and Caddy but I think those two things are likely to be safe enough.