# Home NAS

To run on Raspberry Pi 5 with the Radxa Penta SATA hat, on Ubuntu server. Should
be easy enough to adapt to other distros.

It uses ZFS, but only really to take advantage of compression (which is not
enabled explicitly, it seems to be enabled by default) and global snapshots. It
doesn't create datasets for different users or usecases.

The expectation is that ports 80 and 443 are exposed but the others are only
accessible on your VPN/home network. Caddy takes care of SSL (getting certs from
Let's Encrpyt) and serves FileBrowser via reverse-proxy. FileBrowser does its
own authentication. Prometheus is on port 9090, with no authentication or SSL.

## Why no dashboard?

There is no dashboard, because there aren't any dashboarding tools that I like:

 - I find Grafana really unwieldy it doesn't really support storing your
   configuration as code, it's quite heavyweight, and it also has an annoying
   commercial vibe about it.
 - [Perses](https://perses.dev/) looks promising and even uses
   [Cue](https://cuelang.org/) for configuration which seems great. But:
    - It's clearly really far from being ready for prime time, the docs are
      half-baked and seem really unprofessional.
    - It toots the "Dashboards-as-Code" thing really hard but then it still
      requires you to push your dashboards via its CLI? Argh. And they have a
      k8s operator? Why? It seems like they started on a good basis but they've
      gone down the path of something else way too complex.
  - I found this thing called [Dashbuilder](https://www.dashbuilder.org/) but it
    seems to be a dead project and the Github link goes to a "[home for leading
    Open Source projects that play a role in delivering solutions around Business
    Automation and Artificial Intelligence in the
    Cloud](https://github.com/kiegroup/kie-tools)."

I still believe there must be a way to build a dashboard from a lightweight
configuration, but I haven't found it yet.

On way to achieve that would be a simple [Vega](https://vega.github.io/)-based
static site. From my experience using Vega-Altair for plotting, I think Vega is
the best way to create graphs & charts. However, there's some special features
you need from the actual graphing library that are not there in a plain old
plotting system:

 - It should ideally update live.

 - You should be able to zoom out and scroll around and have extra data fetched
   as you do so.

 - You should be able to sync this logic across multiple graphs.

 - You should be able to dynamically breakdown and aggregate, and this should
   also be aggregated across multiple graphs.

Maybe it's possible to make Vega do this, but I'm not sure. Some more dynamicy
graphing libraries I found:

 - [uPlot](https://github.com/leeoniya/uPlot) - doesn't natively offer
   scrolling though.
 
 - [Chart.js](https://www.chartjs.org/) - I dunno also seems to require plugins.

## To deploy

These steps are done on your normal computer, not on the NAS.

This will install stuff in `~/nas` on the NAS host.

- Make sure all the hardware is already set up - this tool won't e.g. [enable
  PCIe on your
  Pi](https://docs.radxa.com/en/accessories/penta-sata-hat/penta-for-rpi5)

- If the disks you want to install ZFS on are anything other than `/dev/sda`,
  `/dev/sdb`, `/dev/sdc` and `/dev/sdd`, edit `group_vars/nas.yaml`.

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

        # Required - domain the reverse proxy will serve from
        nas_domain: your-domain.com

        # Required - credentials to log in to the FileBrowser UI. You can change
        # these in the UI later.
        filebrowser_username: you
        filebrowser_password: hunter2

        # Optional - skip this if you don't need dynamic DNS.
        # See https://manpages.debian.org/unstable/inadyn/inadyn.conf.5.en.html
        # for the format of this config.
        # I am using Cloudflare DNS - this inadyn config causes my DNS records
        # to be updated to point to my home IP when that changes.
        inadyn_config: |
          provider cloudflare.com {
              username = yawn.io
              password = "<token that I created in the Cloudflare UI>"
              hostname = my-subdomain.yawn.io
          }

        # Optional - GMail alert configuration
        # Check this is working using
        # curl -H 'Content-Type: application/json' -d '[{"labels":{"alertname":"myalert"}}]' http://nas.fritz.box:9093/api/v2/alerts
        # It might take a few minutes to come through.
        gmail:
          account: you@gmail.com
          # https://support.google.com/accounts/answer/185833?hl=en
          password: app-pasword
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
# Note - this is not needed, I dunno why I thought it was. POSIX ACLs are not
# the traditional file ownership biz.
sudo zfs set acltype=posixacl nas-pool
sudo zfs set xattr=sa nas-pool
sudo chmod $USER:$USER /mnt/nas-data
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

- [x] ~~Add MinIO~~ Disabled as the thing I wanted to use it for is
      [broken](https://github.com/laurent22/joplin/issues/9027)
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
- [x] Reduce power consumption
- [x] There's too much fiddly stuff happening on the host. Switch to ansible
  - [x] Back up the `system-data` directory from the device
  - [x] Restore it to a clean Ubuntu installation
  - [x] Rewrite the ZFS and `docker-compose` installation as Ansible tasks
  - [x] Set up FileBrowser without it coming up with `admin:admin` root creds
  - [x] Expose it to the internet again yolo
- [x] Set up dynamic DNS.
- [x] Set up something to automatically manage ZFS snapshots. [See how Jeff
      Geerling did it
      here](https://github.com/geerlingguy/arm-nas/blob/master/host_vars/nas02.mmoffice.net.yml)
- [x] Switch back to proper Let's Encrypt certs once rate limite recovers.
- [x] Add Prometheus
- [x] Add smartmontools
- [x] Plumb SMART data into Prometheus
- [ ] Configure Prometheus SMTP
- [ ] Add alerts :
  - [ ] for SMART data
  - [ ] for ZFS status
- [ ] Add Samba
- [ ] Check if snapshots are working and try to restore one
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
