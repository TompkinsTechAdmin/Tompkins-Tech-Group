# HomeLab Network Migration & Performance Troubleshooting

*OpenWrt firewall · Netgear managed switching · Kea DHCP · Deco wireless mesh*

## Project overview

I migrated my home network from a consumer mesh router handling multiple infrastructure roles to a more modular design. A Cisco Meraki appliance running OpenWrt now provides routing and firewall services, while a dedicated Ubuntu virtual machine provides DHCP. A managed Netgear switch connects wired infrastructure, and the TP-Link Deco units continue providing wireless coverage in access-point mode. The goal was to improve control and flexibility without disrupting household connectivity.

## Goals

• Separate routing/firewall and DHCP responsibilities.
• Preserve the existing 192.168.68.0/24 network and critical server addresses.
• Maintain reliable wireless mesh coverage for household devices.
• Verify connectivity and performance before introducing VLAN segmentation.

## Equipment and software

• Cisco Meraki MX appliance running OpenWrt 24.10.x (observed as MX65 in project notes; one status screen identifies MX64, so exact model should be verified).
• Netgear ProSafe GS724TR managed Gigabit Ethernet switch.
• Four TP-Link Deco W6000 mesh units.
• Proxmox virtualization hosts.
• Net-Srv01: Ubuntu Server 24.04, static LAN address 192.168.68.21, running WireGuard and Kea DHCPv4; DNS service planned.
• Spectrum Internet service.

## Network design

Spectrum modem → OpenWrt WAN → OpenWrt LAN → Netgear managed switch → Proxmox servers and main Deco access point → wireless mesh satellites.

LAN subnet: 192.168.68.0/24
Default gateway: 192.168.68.1 (OpenWrt)
Infrastructure/static allocation plan: 192.168.68.1–192.168.68.50
Dynamic DHCP pool: 192.168.68.51–192.168.68.150
DHCP server: Net-Srv01 at 192.168.68.21

## Implementation

1. Tested the OpenWrt appliance on an isolated LAN before production deployment. Verified WAN DHCP, DNS resolution, firewall defaults, and throughput of approximately 782 Mbps in the initial test environment.
2. Assigned Net-Srv01 a static address using Ubuntu Netplan; verified its default route and WireGuard interface.
3. Installed Kea DHCPv4, prepared its configuration, and validated the syntax before activation.
4. Configured Kea to serve ens18 on 192.168.68.0/24, with a dynamic pool of .51–.150, gateway .1, and public DNS resolvers 1.1.1.1 and 9.9.9.9.
5. Disabled the OpenWrt LAN DHCP server and IPv6 router-advertisement/DHCPv6 services to avoid conflicting local address assignment during the planned IPv4 deployment.
6. Migrated the OpenWrt LAN address to 192.168.68.1 and confirmed management access.
7. Validated a Windows wireless client receiving an address and options from Kea: client 192.168.68.100, DHCP server 192.168.68.21, gateway 192.168.68.1.
8. Confirmed Internet access, ping, and a public WAN address on OpenWrt.

## The performance incident

After cutover, a wireless speed test produced only 88.74 Mbps download and 40.10 Mbps upload, substantially below an earlier Wi-Fi measurement of 923.53 Mbps download. The OpenWrt WAN and LAN uplinks both reported 1 Gbps, so investigation moved to the switch-to-Deco connection.

The Netgear switch showed the Deco connected on port g23 at 100 Mbps full duplex. Forcing 1 Gbps was not accepted as a durable solution; auto-negotiation was restored. After reseating the cable and moving the Deco uplink to port g24, the switch reported 1000 Mbps full duplex. A subsequent Wi-Fi test reached 813.88 Mbps download and 40.11 Mbps upload.

This strongly implicated the original Ethernet connection path, but the testing did not isolate whether the root cause was the cable termination, connector seating, or port g23 itself.

## Measured results

Initial isolated OpenWrt throughput: 781.84 Mbps down / 40.05 Mbps up.
Pre-migration Wi-Fi reference: 923.53 Mbps down / 39.82 Mbps up.
Degraded post-migration Wi-Fi: 88.74–89.67 Mbps down / approximately 40 Mbps up.
After moving the Deco uplink to g24: 813.88 Mbps down / 40.11 Mbps up.

The final download result was approximately 9.1 times the immediately preceding 89.67 Mbps test. Results are point-in-time observations, not guaranteed throughput.

## Key lessons

• A successful link-up does not mean a link is operating at its intended speed.
• Approximately 90 Mbps of application throughput is a useful clue for a 100 Mbps Ethernet bottleneck.
• Verify physical-layer negotiation before changing firewall acceleration or routing settings.
• Preserve auto-negotiation for Gigabit Ethernet and investigate the underlying physical problem.
• Test one change at a time, then measure again.
• Separate infrastructure services carefully and avoid simultaneous DHCP servers on the same broadcast domain.

## Next steps

• Investigate g23 and the original cable independently with known-good hardware.
• Confirm and document the exact Meraki model and OpenWrt build.
• Add and verify the intended DNS service on Net-Srv01 before advertising it through DHCP.
• Back up OpenWrt, Kea, and switch configurations securely, excluding secrets from public documentation.
• Consider VLAN segmentation only after the baseline network remains stable.

## Portfolio note

This write-up describes a personal learning environment. Public documentation intentionally omits credentials, VPN keys, public management access details, and full configuration backups.

