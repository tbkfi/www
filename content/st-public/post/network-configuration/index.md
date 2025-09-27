+++
title = "Network Configuration"
date = 2025-05-19

emoji = "🌎🛡️🔗"
banner_c = "Close-up photo of the network's current primary switch and router."

tags = [ "baseline", "infrastructure", "sysadmin", "networking", "freebsd", "pfsense", "netgate", "mikrotik" ]
draft = false
+++


## Baseline Description

This post details the current active configuration for my networks --both physical and virtual.
Listed is the network structure and subnets, hardware choices and capabilities, layer 2 (L2) configurations,
layer 3 (L3) configurations, and other network services and features.

This is the second in series of three and documents the IP-spaces and networks my machines reside in and take advantage of.
Effectively providing the groundwork and platform for device to device communications for all my systems.

## Goals

### Layer 2

The objects for this layer are to provide strong segregation of subnets via port isolation and VLANs (802.1Q), high-throughput 10GbE trunk connectivity,
1GbE connectivity via access ports, and PoE+ (802.3at) for 30W of power for various IoT and security devices.
The switches should not have any capabilities for Layer 3 routing!

Essentially, I need the L2 network to be as simple and infalliable as possible, low-maintenance, and performant enough to provide relatively high throughputs
while remaining light on power consumption.
The network was planned with 10GbE trunks and 1GbE access ports precisely because of efficiency targets (power usage, heat, cost).
Increasing either trunk- or access port speeds would significantly increase power draw and cost for realistically very little practical benefit for
my expected workloads and uses.

PoE is needed because the entire network and hypervisor will be powered from a single UPS.
Using PoE will bring the downstream switches and their IoT and security devices within that envelope,
so everything is power protected in case of outages or surges. There is an added benefit of reducing
the number of needed AC-adapters which simplifies maintenance.


Below is an anticipated end-result of the VLAN zone assignments per router, and crucially the
point-to-point links at L3. Some interface types have been labeled explicitly, so the depth of the networking
is more apparent at a glance.

![A graphic depicting the planned VLANs.](vlan.png)

Note that the Hypervisor's internal bridge layout is simplified in the previous graphic, and doesn't fully reflect reality. There wasn't
enough space to label and display everything in a way that didn't make the graphic too busy.

### Layer 3

For my primary router I want a dedicated machine. While I technically have the option to virtualise it, I'm kind of past doing that for my primary gateway (see [historical](#historical)).
I need routing, dns, and network time to be reliably available; and, as unaffected as possible by other upgrades and maintenance.

The router will need to provide sufficiently fast inter-vlan routing relative to my 10GbE trunk connections, enough processing power
to encrypt and decrypt multi-client VPN traffic (OpenVPN and Wireguard), flexible firewall rules, policy-based routing rules for multiple gateways, and on-device
certificate management and load-balancing for basic services.

The router should also provide at least local DHCP, NTP, DNS, TFTP, and limited IP and DNS-blocking capabilities.

## Hardware Choices

### Switching

For my L2 network I chose to go with Mikrotik switches. In part for their excellent price-to-performance and sensible SKUs, and in part
because they have a long-standing reputation for good reliability and support among the enthusiast and professional space.
They're also based in Europe, which is great because I get to support an European company.

Specifically I chose the following two switches:

![Mikrotik Switches](mikrotik-sw.jpg)

- Main L2 Switch (SW1)
    - [CSS610-8P-2S+IN](https://mikrotik.com/product/css610_8p_2s_in)
    - SwitchOS Lite
        - L2 features only
        - Dot1q VLANs, MAC filters, MAC-IP headers, traffic mirrors, bw limits, ...
        - PoE monitoring (Current, Voltage, Power)
    - 2 x SFP+ (10GbE Copper or Optical)
    - 8 x 1GbE (with PoE+ out)
    - 12 Watt w/o attachments
    - 160 Watt PoE budget
- Secondary L2 Switch (SW2)
    - [CSS610-8G-2S+IN](https://mikrotik.com/product/css610_8g_2s_in)
    - SwitchOS Lite
        - L2 features only
        - Dot1q VLANs, MAC filters, MAC-IP headers, traffic mirrors, bw limits, ...
        - PoE monitoring (Current, Voltage, Power)
    - 2 x SFP+ (10GbE Copper or Optical)
    - 8 x 1GbE
    - Passive PoE input for power
    - 5 Watt w/o attachments

These switches are effectively the same product, with the difference being one has PoE+ outputs and the
other can be powered via its PoE input. The unit without PoE outs is also conveniently much smaller, so it can
fit into a small patch panel as seen from the photo below.

Note that the loads in the cables below will be low --just that one PoE input to SW2. So the very convenient loop pictured in the photo
won't be melting anything in its vicinity or causing other trouble... But yes, it's indeed bad practice to make loops like that.
I just got fed up with the cables pushing the panel door open.

![SW2 installed into a small patch panel](network-sw2.jpg)

Another notable choice for the L2 network was the SFP+ modules I use for the trunk links. Due to what is in-between the walls and
what NICs I have available on my hypervisor, I had to use 10GBASE-T on my SFP+ ports. Optics or DAC would've been more efficient
financially and thermally, but neither was an option in my case.

I initially purchased two Mikrotik branded SFP+ modules; and, while they work great they're also based on older (Marvell?) chips that use a lot more power.
The downside to using more power is critically the additional heat, and below is an example of just that.

![Comparison of old 10GBASE-T SFP+ modules with newer variants](sfpp-comparison.jpg)

Firstly, we can see that having two of the newer modules side-by-side in SW1 yields temperatures around 50-54 Celsius, while
the single older variant SFP+ module from Mikrotik hovers in excess of 80 Celsius when on its own. Both SW1 and SW2 chassis are passively cooled
and have no fans circulating air. This temperature difference is enormous, and I would readily replace the Mikrotik
module if the 80 meter 10GBASE-T modules didn't cost 70€ a pop (*ouch...*).

So if you're in a spot where you must use copper, and a DAC isn't an option, make sure you get the newer Broadcom chipped modules.
Look for something with BCM84891L in it, though you may have to take someone's word on it. When I was looking, it was difficult if not impossible
to find indication on the chip in product listings. Look for brands that other people online have verified to run cooler and behave well
in use, and make sure the listing indicates the module is for either 80 or 100 meters. The support for longer runs indicates
a more efficient model!

[This is what I bought, and what works well for me](https://www.amazon.de/-/en/10Gtek%C2%AE-10GBase-SFP-transceiver-module/dp/B0BRQ5MDHY), but it's
practically for 10Gbps only!

### Routing

For my primary router I chose to go with a first-party pfSense router from Netgate.
Its specs are strong but modest. It doesn't provide any crazy unique connectivity or features, and the RAM is soldered,
but it's all low-power, has a well balanced specsheet, and uses reliable parts.

![Netgate 4200 Router](netgate-router.jpg)

- Netgate 4200 BASE
    - Software: pfSense+ included
    - CPU: Intel Atom C1110
    - RAM: 4GB LPDDR5
    - DRIVE: NVME drive (M-key), and internal eMMC (shouldn't be used, design mistake!)
    - Console ports ("Cisco pinout" RJ45, micro-USB)
    - PORTS: 4 x 2.5Gbps unswitched Ethernet
    - Speed (L3 Forwarding): ~9Gbps
    - Speed (Firewall, 10k ACLs): ~8Gbps
    - Speed (AES-GCM128 w/ AES-NI): ~3Gbps
    - Power consumption: ~13-20 Watts

I've been using pfSense since 2018 as my primary firewall and router OS, so I knew it serves my needs
well. As to why I selected a first-party router that comes at a price premium...

pfSense (CE) is Free & Open-Source software (FOSS), but its existence and continued availability should not be taken as a given.
As I've had the privilege of using it free for many years since I initially virtualised it back in 2018, and have used it ever since.
I feel it was only fair to kick back a little thanks for all that time and opportunities to learn and tinker,
even if in the grand scheme of things it counts for very little.

Additionally, as efficiency was a key part of my design, I specifically wanted a low-power machine. I couldn't just use
an old computer because they tend to draw a lot more power, specially when kitted with the neccessary NICs.
This is why I selected from the purpose built routers on the market.

For anyone considering a new router, I would really urge to consider if it's a smart choice to buy a no-name
chinese router as your primary firewall and router. This is the device that sits between all your networks, data, machines, and the internet.
For a modest amount more you could have something from an American or European company securing your network
with reliable documentation, while also kicking back a little support for the projects they advance and maintain.
If Netgate's models are outside of your budget or needs, Mikrotik makes some very good budget friendly routers that use their RouterOS!

## Software and Configuration

Pending Review

### Mikrotik SwitchOS Lite

Pending Review

### pfSense

Pending Review

### OPNsense

Pending Review

## Historical

Pending Review

## Wrap-up

Pending Review

![SW1 and R1 sitting on top of a cabinet](network-r1-sw1.jpg)
