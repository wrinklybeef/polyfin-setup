# Troubleshooting

Run Compose commands from the repository root. Start with the failing layer: VM networking, container networking, source discovery, playback, or HTTPS.

## Docker DNS and torrent peer discovery

**Symptoms:** sources appear, but playback remains at “Finding peers.” Logs show `0 peers found, 0 connected`, trackers with no responses, or DHT lookups returning no peers.

Inspect the container's DNS configuration and recent logs:

```bash
sudo docker compose --env-file .env -f docker/polyfin/compose.yaml exec polyfin cat /etc/resolv.conf
sudo docker compose --env-file .env -f docker/polyfin/compose.yaml logs --since=10m polyfin \
  | grep -iE 'torrent|tracker|peer|dht|dns|route'
```

Compose normally creates a user-defined network. Containers on that network use Docker's embedded resolver, `127.0.0.11`. Polyfin's **Network interface** torrent route skips loopback DNS resolvers for tracker lookups, so this resolver can prevent tracker hostname resolution. See [Polyfin torrent networking](https://github.com/moodiness/polyfin/blob/main/docs/torrents.md) and [Docker DNS behavior](https://docs.docker.com/engine/network/#dns-services).

The working change was to use Docker's built-in default bridge:

```yaml
services:
  polyfin:
    network_mode: bridge
```

This setting is already included in this repository. It replaces the implicit Compose network; do not also specify `networks:` on that service. It is different from creating a custom network with `driver: bridge`.

Recreate the container to apply it:

```bash
sudo docker compose --env-file .env -f docker/polyfin/compose.yaml up -d --force-recreate
sudo docker compose --env-file .env -f docker/polyfin/compose.yaml exec polyfin cat /etc/resolv.conf
```

The resolver changed to the LAN DNS server and playback started working. Verify that your container likewise has a reachable non-loopback resolver, then retry a source with available peers. The observed fix addresses the DNS path; zero peers alone does not prove a DNS problem, and the change also recreates the container's network state.

Directly editing `/etc/resolv.conf` failed because it was read-only in this deployment. Docker manages that file, so make persistent changes through container configuration instead of changing permissions inside it. After a VLAN or DHCP DNS change, recreate the container and check its resolver again.

If discovery still fails, check the selected torrent route, user torrent permission, outbound firewall logs, and source availability. Temporarily enable Polyfin's detailed diagnostics to identify tracker errors, then turn them off after troubleshooting. Opening inbound torrent ports was not needed for this fix.

## Missing streaming sources

Cinemeta supplies catalogs and metadata, not playable video. Add a Stremio-compatible streaming provider under **Content > Sources** and confirm it returns streams for the selected title.

For torrent sources, enable both the server's torrent engine and the user's **Can play torrent streams** permission. An add-on's source count is not the number of peers currently connected to the server.

## Jellyfin and Polyfin port conflicts

Both servers commonly use TCP 8096. If the browser opens the old Jellyfin interface or Docker cannot publish the port, identify the listener:

```bash
sudo ss -ltnp | grep ':8096'
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
sudo systemctl status jellyfin --no-pager
```

When intentionally replacing a native Jellyfin service, stop and disable it:

```bash
sudo systemctl disable --now jellyfin
sudo docker compose --env-file .env -f docker/polyfin/compose.yaml up -d --force-recreate
```

This preserves Jellyfin's data and Polyfin's persistent volume. For an old container installation, stop it through its own Compose project. Confirm that the new container publishes `8096:8096`.

## Reverse proxy connectivity

First test Polyfin locally, then Caddy with the correct hostname and certificate:

```bash
curl -I http://127.0.0.1:8096/web/
curl -I --resolve tv.example.com:8443:192.168.50.10 \
  https://tv.example.com:8443/web/
sudo docker compose --env-file .env -f docker/caddy/compose.yaml logs --tail=50 caddy
```

- **502 Bad Gateway:** check Polyfin is running and publishing 8096. Caddy uses `host.docker.internal:8096`, with the `host-gateway` entry in Compose. `127.0.0.1` inside Caddy refers to Caddy's container, not the VM or Polyfin.
- **Certificate issuance error:** check the domain, scoped Cloudflare token, DNS propagation, and outbound connectivity. The custom image must include `dns.providers.cloudflare`.
- **Local HTTPS works, remote connection fails:** check the DNS-only record, TCP 8443 VIP, WAN policy, source-IP allowlist, and explicit `:8443` in the URL. A VIP alone does not permit traffic.
- **Caddyfile formatting or UDP buffer warning:** formatting can be corrected with `caddy fmt`; the HTTP/3 UDP buffer warning does not prevent HTTPS over the published TCP port.

Inspect the plugin and validate Caddy's configuration with:

```bash
sudo docker compose --env-file .env -f docker/caddy/compose.yaml exec caddy caddy list-modules
sudo docker compose --env-file .env -f docker/caddy/compose.yaml exec caddy \
  caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

## Basic networking problems

- **No DHCP lease after assigning the VLAN:** check tagged membership on both switch uplinks, VLAN awareness on `vmbr0`, the VM tag, and the FortiGate DHCP server. Keep the host management VLAN untagged as before.
- **VM has an address but no internet:** check the default gateway and media-to-WAN policy with source NAT enabled. An ISP DNS address supplied by DHCP is not itself evidence of failure.
- **Application works, SSH fails:** add TCP 22 from the administration workstation. A TCP 8096 rule does not also allow SSH.
- **Inter-VLAN access fails:** use a Subnet address object with the VM's `/32`, the correct destination service, and no source NAT. Check policy order and Proxmox firewall rules.
- **TV cannot find the server:** enter its full URL manually. Discovery broadcasts do not normally cross routed VLANs.

## Playback and metadata

If hardware-encoder probes report a missing GPU library but Polyfin selects software transcoding, use a directly playable source or allow enough CPU for conversion. GPU passthrough is not required for this setup.

Check free disk space when large sources stall; temporary streaming caches still consume storage. For missing descriptions, refresh the title's metadata and inspect source errors in **System > Health**. A TMDB key enriches cast and artwork, but does not turn a metadata catalog into a streaming source.
