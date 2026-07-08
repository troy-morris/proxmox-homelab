# Proxmox Home Lab — Virtualized Infrastructure

A self-designed, self-hosted home lab built on **Proxmox VE**, used for hands-on IT and systems-administration skill development. It runs a segmented internal network behind a virtualized firewall, network-wide DNS filtering, GPU-accelerated workloads via hardware passthrough, and secure remote access — all on a single physical host.

This project demonstrates **virtualization, network segmentation, firewall administration, DNS/DHCP, Linux administration, GPU (VFIO) passthrough, and self-hosting**. Every layer was built and verified by hand to develop genuine, first-principles understanding rather than relying on helper scripts.

---

## Architecture

```mermaid
flowchart TD
    Internet([Internet]) --> HostNIC["Proxmox Host<br/>WiFi uplink (ICS/NAT)"]

    subgraph HOST["Proxmox VE Host — i7-7700K / GTX 1080 Ti"]
        HostNIC --> OPN["OPNsense VM<br/>Firewall / Router<br/>WAN: 10.0.0.3 · LAN: 10.10.10.1/24"]
        OPN --> LANBR["Internal LAN bridge (vmbr1)<br/>Lab subnet 10.10.10.0/24<br/>DHCP 10.10.10.100–200"]

        LANBR --> PIHOLE["Pi-hole (LXC)<br/>10.10.10.2<br/>DNS filtering"]
        LANBR --> LLM["LLM VM<br/>GTX 1080 Ti via VFIO<br/>Ollama + Open WebUI"]
        LANBR --> MC["Ubuntu Server VM<br/>Minecraft (systemd)"]
    end

    TAIL{{"Tailscale mesh<br/>remote access to all nodes"}}
    PIHOLE -.-> TAIL
    LLM -.-> TAIL
    MC -.-> TAIL
    OPN -.-> TAIL
```

---

## Host Hardware

| Component | Detail |
|-----------|--------|
| CPU | Intel Core i7-7700K |
| GPU | NVIDIA GTX 1080 Ti (passed through via VFIO) |
| RAM | 16 GB DDR4 |
| Boot | 120 GB SATA SSD (Proxmox VE OS) |
| VM storage | 2 TB NVMe (LVM-Thin, `nvme-thin`) |
| Bulk / backups | 3 TB HDD (`hdd-bulk` — ISOs & backups) |
| Hypervisor | Proxmox VE 9.2 |

Storage is split by role: a small SATA SSD for the hypervisor, an LVM-Thin NVMe pool (`nvme-thin`) for VM/CT disks, and a bulk HDD (`hdd-bulk`) for ISOs and backups.

![Proxmox storage pools](images/lab-01.png)
*Datacenter storage layout — separate pools for VM disks (`nvme-thin`), bulk/ISO storage (`hdd-bulk`), and the local hypervisor volumes.*

---

## Network Design

The lab uses segmented subnets to separate the internal lab network from the home LAN, with OPNsense as the boundary firewall/router.

| Network | Range | Purpose |
|---------|-------|---------|
| Home LAN | 192.168.4.0/22 | Existing home network |
| ICS link | 10.0.0.0/24 | Uplink between host and OPNsense WAN |
| Lab internal | 10.10.10.0/24 | Isolated lab network (behind OPNsense) |

- **`vmbr1`** (the internal LAN bridge) is configured as `inet manual` — no IP assigned to the host on this bridge, so all routing for the lab subnet is handled by OPNsense.
- Multicast snooping is disabled on the internal bridge, and per-VM firewalling on network devices is turned off so OPNsense is the single point of network control.

A managed TP-Link TL-SG108E switch connects the host's onboard NIC and the Intel dual-port NIC used for the lab, with a dedicated port for the daily-driver machine.

![Switch port map](images/lab-02.png)
*Port mapping for the TL-SG108E: onboard ethernet (LAN), the two Intel 82576 NIC ports feeding the lab, and the daily-driver machine.*

![Switch system info](images/lab-03.png)
*TL-SG108E management page — the switch sits at `192.168.4.40` on the home LAN.*

The Proxmox host is given a static address on the lab-facing interface (`eno1`) so it presents a stable gateway/uplink for the OPNsense WAN.

![Host static IP configuration](images/lab-04.png)
*Static addressing on `eno1` defined directly in the NetworkManager connection file — `10.0.0.1/24` for the ICS link plus `192.168.4.41/22` on the home LAN, with a source-based host route.*

![Host address verification](images/lab-05.png)
*`ip addr show eno1` confirming both addresses are live on the interface.*

---

## Components

### OPNsense — Firewall & Router (VM)

The boundary of the lab network. OPNsense runs as VM 100 with a WAN interface on the ICS link (`10.0.0.3`) and a LAN interface serving the isolated `10.10.10.0/24` subnet (`10.10.10.1`). All inter-network traffic and firewall policy is enforced here, keeping the lab segmented from the home LAN.

![OPNsense VM hardware](images/lab-06.png)
*VM 100: 2 vCPU / 2 GB RAM, OVMF (UEFI) boot, WAN on `vmbr0`, booting the OPNsense installer ISO from `hdd-bulk`.*

![OPNsense installation](images/lab-07.png)
*OPNsense installer cloning to the target disk.*

![WAN connectivity check](images/lab-08.png)
*From the OPNsense console — successful pings to the host gateway (`10.0.0.1`) and out to the internet (`8.8.8.8`), confirming the WAN path works.*

![OPNsense dashboard](images/lab-09.png)
*OPNsense Lobby dashboard: running on the i7-7700K, active WAN gateway, core services up, and the firewall summary showing a default-deny posture.*

---

## Firewall Rules & Verification

OPNsense is the single enforcement point between the isolated lab (`10.10.10.0/24`) and everything outside it. The ruleset is intentionally minimal and default-deny: only explicitly permitted traffic passes, and both blocks and passes are logged so the policy can be verified against live traffic.

![Firewall rules](images/lab-10.png)
*Interface ruleset — a scoped "Allow WAN GUI access from daily" rule (TCP 443 from `10.0.0.0/24` to the firewall) alongside the default LAN allow rules.*

![WAN GUI rule detail](images/lab-11.png)
*The WAN management rule in detail: quick-match, IPv4/TCP, source `10.0.0.0/24`, destination "This Firewall" on HTTPS — narrow access to the web UI rather than an open WAN.*

![Advanced logging settings](images/lab-12.png)
*Firewall logging enabled for default block, default pass, and blocked bogon/private-network traffic — this is what feeds the live log evidence below.*

Inspecting the packet filter directly confirms the lab subnet is fenced off at the ruleset level:

![pfctl lab-subnet block rules](images/lab-13.png)
*`pfctl -sr | grep 10.10.10` on the firewall — explicit block-drop rules covering the `10.10.10.0/24` segment.*

The live firewall log shows the policy working in real time — default-deny dropping unsolicited traffic, and named rules passing only what's intended:

![Live log — default deny](images/lab-14.png)
*Firewall live log: inbound WAN traffic dropped by the "Default deny / state violation" rule.*

![Live log — block and pass](images/lab-15.png)
*Blocks (red) and passes (green) side by side — the default-deny drops plus explicit passes for the firewall host itself and for DHCP.*

![Firewall state table](images/lab-16.png)
*Diagnostics → States showing the single established WAN-GUI session matched to its rule — state tracking working as configured.*

Verification was also done from inside the lab, proving segmentation from the client's perspective:

![Client outbound via firewall](images/lab-17.png)
*A lab client reaching the internet (ICMP to `8.8.8.8`) through OPNsense — outbound NAT working for permitted traffic.*

![Client default-deny behaviour](images/lab-18.png)
*The same client can ping its gateway (`10.10.10.1`) and pulls a DHCP lease/route, but is refused/timed out on traffic the firewall doesn't permit — default-deny confirmed end to end.*

---

## DNS & DHCP

DHCP and local name resolution for the lab are handled inside OPNsense by **Dnsmasq**, serving the `10.10.10.0/24` LAN with the `lab.local` domain. DHCP registers client hostnames (and matching firewall rules) automatically, so leased hosts are resolvable by name on the lab network. Filtering DNS is then layered on with **Pi-hole**.

![Dnsmasq DNS & DHCP](images/lab-19.png)
*Dnsmasq general settings — enabled on the LAN interface, `lab.local` domain, hostname/FQDN and firewall-rule registration on.*

![DHCP host reservation](images/lab-20.png)
*A static host override tying the `labvm` client to `10.10.10.137` by hardware address/client-id — reserved DHCP with a resolvable name.*

![Client DNS assignment](images/lab-21.png)
*`resolvectl` on a lab client confirming its DNS server is `10.10.10.1` (the firewall), handed out via DHCP.*

### Pi-hole — Network-wide DNS Filtering (LXC)

Pi-hole runs as a lightweight Debian LXC at `10.10.10.2`, acting as the filtering resolver for the lab and blocking ads/trackers network-wide. (Running Tailscale inside an unprivileged LXC required enabling the TUN device for the container.)

![Pi-hole installed](images/lab-22.png)
*Pi-hole install complete at `10.10.10.2`. (Admin password redacted.)*

![Pi-hole query log](images/lab-23.png)
*Live query log — Pi-hole receiving lab DNS queries and forwarding them upstream.*

![Pi-hole dashboard](images/lab-24.png)
*Pi-hole dashboard: active resolver with ~82,900 domains on blocklists, plus query-type and upstream-server breakdowns.*

![Pi-hole over Tailscale](images/lab-25.png)
*The Pi-hole admin interface reached remotely over the Tailscale mesh.*

---

## GPU Passthrough + Local LLM (VM)

The standout piece of the build. The GTX 1080 Ti is passed through to a dedicated VM (VM 103) using **VFIO**:

- IOMMU enabled via GRUB
- GPU device IDs bound to `vfio-pci` and blacklisted from the host so Proxmox releases the card
- Secure boot handled (pre-enrolled keys) to allow the passthrough

The VM runs **Ollama** (serving local models such as Llama 3.1, Mistral, Gemma 2, and Qwen 2.5) with **Open WebUI** in Docker as the front end — a fully local, GPU-accelerated LLM environment.

![LLM VM with GPU passthrough](images/lab-26.png)
*VM 103 hardware: the GTX 1080 Ti attached as a PCI device (`hostpci0`, `x-vga=1`) on an OVMF/UEFI machine with pre-enrolled keys.*

![Ollama as a systemd service](images/lab-27.png)
*Ollama configured as a systemd service (`OLLAMA_HOST=0.0.0.0:11434`), edited remotely over a Tailscale SSH session.*

---

## Minecraft Server (VM)

An Ubuntu Server VM (VM 101) running a Minecraft server managed by **systemd**, demonstrating Linux service management and hosting on the segmented lab network.

## Tailscale — Secure Remote Access

All lab nodes are connected to a **Tailscale** mesh network, providing encrypted remote access to every VM and container from anywhere without exposing services to the public internet (see the Pi-hole and Ollama sections above, both administered over Tailscale).

---

## Monitoring — Grafana + Prometheus (LXC)

A dedicated Debian 12 LXC (CT 104) running **Prometheus** and **Grafana** provides real-time visibility across all lab nodes. **Node Exporter** is deployed on every VM and container, scraping system metrics every 15 seconds into Prometheus, with Grafana dashboards surfacing CPU, memory, disk, and network data in one place.

- Prometheus scrapes five targets: monitoring CT, Pi-hole, Ubuntu Server VM, LLM VM, and the Proxmox host
- Grafana connects to Prometheus at `http://10.10.10.3:9090` and is accessible over Tailscale
- Node Exporter Full dashboard (ID 1860) provides per-node drilldown across all metrics

[![Grafana dashboard](images/lab-28.png)](images/lab-28.png)
*Node Exporter Full dashboard — live metrics for the Proxmox host showing CPU, RAM, disk usage, and network traffic across all bridge interfaces.*
## Self-Hosted Git — Gitea (LXC)

**Gitea** runs as a lightweight Debian 12 LXC (CT 105) at `10.10.10.4`, providing a private self-hosted Git service accessible over Tailscale. All homelab repositories push simultaneously to both GitHub (public portfolio) and Gitea (private backup) via dual push remotes configured in git — a single `git push origin main` updates both.

[![Gitea web interface](https://github.com/troy-morris/proxmox-homelab/raw/main/images/lab-29.png)](/troy-morris/proxmox-homelab/blob/main/images/lab-29.png)
*Gitea dashboard — self-hosted Git service running on the lab network, accessible remotely over the Tailscale mesh.*

[![proxmox-homelab repo on Gitea](https://github.com/troy-morris/proxmox-homelab/raw/main/images/lab-30.png)](/troy-morris/proxmox-homelab/blob/main/images/lab-30.png)
*The proxmox-homelab repository mirrored to self-hosted Gitea alongside GitHub, pushed via dual remote configuration.*

---

## Skills Demonstrated

- **Virtualization:** Proxmox VE, KVM VMs and LXC containers, storage pools (LVM-Thin), UEFI/OVMF guests, backups
- **Networking:** subnetting and network segmentation, managed-switch port layout, static addressing and routing, DHCP, DNS
- **Firewall administration:** OPNsense WAN/LAN design, default-deny policy, scoped management access, live-log and state-table verification, `pfctl` inspection
- **DNS:** self-hosted resolver, DHCP-integrated local domains, network-wide filtering (Pi-hole)
- **Advanced virtualization:** VFIO / IOMMU GPU passthrough (device binding, host blacklisting, secure-boot handling)
- **Linux administration:** Debian/Ubuntu, systemd services, Docker, cloud-init
- **Remote access & security:** Tailscale mesh networking, keeping services off the public internet
- **Observability:** Prometheus metrics collection, Node Exporter deployment, Grafana dashboards for real-time visibility across heterogeneous lab nodes

---

## Notes

This lab is built and maintained by hand (manual installs over helper scripts) to build genuine understanding of each layer. Internal IP addresses shown are private (RFC 1918) ranges; no credentials, keys, or public addresses are included.

---

*Personal infrastructure lab for IT / systems administration skill development.*
