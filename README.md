# Home NAS

To run on Raspberry Pi 5 with the Radxa Penta SATA hat.

## To deploy

These steps are done on your normal computer, not on the NAS.

This will install stuff in `~/nas` on the NAS host.

- If you want your node on Tailscale, generate an [Auth
  Key](https://login.tailscale.com/admin/settings/keys) to associated the device
  with your account.

- Install Ansible (e.g. `sudo apt install ansible`)

- Write `./inventory.yaml`:

  ```yaml
  all:
    hosts:
      nas:
        # Required - FQDN, IP address. whatever you'd ssh to.
        ansible_host: your-nas
        # If different from local username
        ansible_user: your-user
        # See https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html
        # (ctrl-f "Variable: " to see the relevant variables you can set here)

        # Optional - if you skip this TailScale won't be set up.
        tailscale_authkey: tskey-auth-12345abc-adjkldsajkl
  ```

- Run `ansible-ansible-playbook site.yaml -i inventory.yam`

> [!TIP]
> If you wanna fiddle with the playbook and re-run it:
> Ansible playbooks are supposed to be idempotent but they are often slow at
> deciding they have nothing to do on subseuent runs. This one certainly is. You can set stuff like
> --skip-tags=tailscale,ansible to skip the tasks in the playbook based on the
> `tags` field.

Once:

```
sudo sysctl net.ipv4.ip_unprivileged_port_start=80
```

```
sudo zpool create -m /mnt/nas-data -f nas-pool raidz /dev/sda /dev/sdb /dev/sdc /dev/sdd
sudo zfs set acltype=posixacl nas-pool
sudo zfs set xattr=sa nas-pool
sudo chmod $USER:$USER /mnt/nas-data
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
# Note neither podman-compose nor docker-compose good at reloading the config so
# sometimes you have to down then up.
rsync --exclude=system-data/ -avz . $HOST:nas && ssh $HOST "cd nas/; podman-compose up"
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
- [x] Disable port forwarding to avoid exposing insecure FileBrowser defaults
- [x] Check for data on the old RAID fs, back it up
- [x] Install Ubuntu Server on the Pi
- [x] Create ZFS
  - [x] Create pool
  - [x] Ensure it's auto-mounted
  - [x] Figure out ACLs for the filesystem on it
- [x] ZFS mounted exposed to FileBrowser
- [x] Make NAS services start up on boot
- [ ] There's too much fiddly stuff happening on the host. Switch to ansible
  - [ ] Back up the `system-data` directory from the device
  - [ ] Restore it to a clean Ubuntu installation
  - [ ] Rewrite the ZFS and `docker-compose` installation as Ansible tasks
- [ ] Set up something to automatically manage ZFS snapshots. [See how Jeff
      Geerling did it
      here](https://github.com/geerlingguy/arm-nas/blob/master/host_vars/nas02.mmoffice.net.yml)
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


## On docker-compose

(Note: `podman-compose` doesn't work with `podman-remote`. I should have used
`docker-compose` I guess. But maybe I have to switch to k8s later anyway).

(OK no, `docker-compose` also sucks, it doesn't really support remote control in
a sensible way either, you just expose your Docker daemon socket over TCP? Are
all these simplified container tools just toys for babies? Do I have to use
k8s?)
