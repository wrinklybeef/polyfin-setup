# Networking

Keep the Proxmox host on its existing management network and place only the media VM in a dedicated VLAN. The FortiGate routes between networks and enforces access policies; the Cisco switch carries tagged traffic between it and Proxmox.

## Example addressing

| Purpose | Example |
| --- | --- |
| Trusted LAN / Proxmox management | `192.168.10.0/24` |
| Media VLAN | `50` |
| Media subnet / gateway | `192.168.50.0/24` / `192.168.50.1` |
| DHCP pool | `192.168.50.100-192.168.50.199` |
| Reserved media VM | `192.168.50.10` |
| Public WAN address | `203.0.113.10` |
| Allowed remote household | `198.51.100.25/32` |
| Public hostname | `tv.example.com` |

These are example values, not a real network inventory. Public IP examples use documentation-only ranges and must be replaced.

## 1. FortiGate VLAN and DHCP

Under **Network > Interfaces**, create a VLAN interface named `media` on the interface connected to the switch trunk:

- VLAN ID: `50`
- IP/netmask: `192.168.50.1/24`
- Role: `LAN`
- Administrative access: none
- DHCP: enabled, range `192.168.50.100-192.168.50.199`
- Gateway: interface address
- DNS: specify `1.1.1.1` and `1.0.0.1`

The LAN role exposes DHCP options in the GUI; firewall policies provide isolation regardless of the role label. Reserve `192.168.50.10` for the VM's MAC address, outside the dynamic pool but within the media subnet. A DNS resolver on another internal VLAN needs its own narrow access policy for TCP/UDP 53.

## 2. Cisco switch

Create VLAN 50. On an SG300, use **VLAN Management > Port to VLAN** to set these memberships:

| Port role | Existing management/native VLAN | Media VLAN 50 |
| --- | --- | --- |
| Uplink toward FortiGate | Preserve existing membership | Tagged |
| Port toward media Proxmox host | Preserve existing untagged membership | Tagged |
| Intermediate uplinks, if present | Preserve existing membership | Tagged |

Use trunk/general mode as appropriate for the existing switch configuration. Do not change the Proxmox port into a media-only access port or change its PVID. Apply and save the running configuration to startup configuration.

## 3. Proxmox and Ubuntu

1. Open the Proxmox node's **System > Network > vmbr0**.
2. Enable **VLAN aware** and ensure the permitted VLAN IDs include 50.
3. Preserve the bridge's management IP, gateway, and physical uplink. Apply with console access available.
4. Open the VM's **Hardware > Network Device**. Keep `vmbr0`, the VirtIO model, and the existing MAC address; set **VLAN Tag** to `50`.
5. In the VM console, check `ip -4 addr show` and `ip route`. Renew DHCP or reboot the VM if it retains its old lease.

Ubuntu receives an ordinary untagged interface, commonly `ens18`; Proxmox applies the VLAN tag. Do not add another VLAN tag inside the guest. Check any enabled Proxmox firewall rules as well as FortiGate policies.

## 4. Internal and outbound policies

Create address objects for the media subnet, the VM as a `/32`, the trusted LAN, and the administration workstation. Define custom TCP services for 8096 and 8443.

| Traffic | Source | Destination | Service | Source NAT |
| --- | --- | --- | --- | --- |
| Media > WAN | Media subnet | Internet | ALL, supporting source-dependent peer ports | Enabled, outgoing interface address |
| LAN > Media | Trusted LAN | Media VM `/32` | TCP 8096 and 8443 | Disabled |
| Admin > Media | Admin workstation `/32` | Media VM `/32` | TCP 22 | Disabled |
| Media > Other internal networks | Media subnet | Internal networks | Deny except explicit requirements | Disabled |

Enable logging and place specific permits above applicable deny rules. Do not add a broad media-to-LAN allow policy. Established connections have return traffic handled by the stateful firewall; they do not require reverse allow policies.

Test SSH and `http://192.168.50.10:8096` from an allowed LAN device. Test internet access from the VM with `curl -I --max-time 10 https://www.cloudflare.com`.

## External access

Cloudflare's DNS-only A record points `tv.example.com` at `203.0.113.10`. Caddy obtains its certificate through DNS validation as described in the [README](../README.md#12-configure-cloudflare-dns-and-the-api-token).

Create a **port-forwarding VIP** on the internet-facing interface:

| VIP setting | Value |
| --- | --- |
| Name | `Polyfin-HTTPS-VIP` |
| External IP | `203.0.113.10` |
| Mapped IP | `192.168.50.10` |
| Protocol | TCP |
| External port | 8443 |
| Mapped port | 8443 |

Then create the firewall policy:

| Policy setting | Value |
| --- | --- |
| Name | `WAN-to-Polyfin` |
| Incoming / outgoing interfaces | WAN / media |
| Source | Remote household public IP, `198.51.100.25/32` |
| Destination | `Polyfin-HTTPS-VIP` |
| Service | Custom TCP 8443 |
| Schedule / action | Always / accept |
| Source NAT | Disabled |
| Logging | All sessions |

The VIP performs destination NAT; the policy permits the connection. Both are required. Do not forward SSH or TCP 8096 from the WAN.

Existing applications can keep their own public 80/443 mappings. An existing **all-port VIP** on the same public IP overlaps with the new VIP: narrow it to the ports its application actually needs before adding another mapping, preserving that application's required access. Different hostnames alone do not separate FortiGate port-forwarding destinations.

Test `https://tv.example.com:8443/web/` from the allowed external network. Testing from inside the LAN through the public IP depends on hairpin NAT; it is not a substitute for an external test. The `curl --resolve` command in the README tests local HTTPS directly.

## References

- [Proxmox network configuration](https://pve.proxmox.com/wiki/Network_Configuration)
- [FortiGate administration documentation](https://docs.fortinet.com/product/fortigate)
- [Cisco 300 Series administration guides](https://www.cisco.com/c/en/us/support/switches/small-business-300-series-managed-switches/products-installation-and-configuration-guides-list.html)
