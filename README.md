# Home NAS

To run on Raspberry Pi 5 with the Radxa Penta SATA hat.

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
- [ ] Figure out how to add Tailscale. I only want the bare minimum of services
- [ ] Figure out how to expose everything on Tailscale, but only the bare
      minimum on the internet
- [ ] Add MinIO
- [ ] Add Prometheus
- [ ] Configure Prometheus SMTP
- [ ] Disable port forwarding to avoid exposing insecure FileBrowser defaults
- [ ] Check for data on the old RAID fs, back it up
- [ ] Install Ubuntu Server on the Pi
- [ ] Create ZFS
- [ ] ZFS mounted exposed to FileBrowser
- [ ] Add smartmontools
- [ ] Plumb SMART data into Prometheus
- [ ] Add alerts for SMART data
- [ ] Add Samba
- [ ] If I run out of memory on the Pi, port this to MicroK8s so I can scale it
      horizontally.