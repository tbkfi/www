+++
title = "Planning a Cellular Deployment"
date = 2025-08-30

type_img = ""
banner_c = ""

tags = ["infrastructure", "radio", "networking", "cellular", "5G", "linux", "sysadmin", "openwrt", "embedded", "teltonika", "mikrotik", "gl-inet"]
draft = false
+++


## Background information

A rural building needs reliable internet connectivity for regular household use. The location is not currently used year-round,
so finding a sufficiently performant solution that is cost effective is key.

Having gotten a very reasonable quote for building fiber connectivity to the location, it was the initial plan; however, it quickly became apparent that 
the ISPs inflexible plan structure was simply not cost-effective in the medium term --much less over the 60 month contract period.
A good price on fiber doesn't matter if its not getting any use most of the year, that's just called burning money.

Therefore I decided to investigate other available options, namely cellular connectivity.

### Requirements and expectations

Even as a rather advanced user, my connectivity needs to the internet are not vast. I don't really have much time to play video games
anymore, so I don't need sub 10ms latencies to multiplayer servers. And if I want to download something, I don't mind automating systems so they work in the background with traffic shaping.

I figure a modest 50mbps downlink and 10Mbps uplink with ~50ms latency will be entirely sufficient --and I have a good feeling achieving that will be more than doable for the location with current generation radio technologies.


## Site details

Before I could begin considering practical deployment options, I had to understand what sort of features were
inherent to the site. This information will dictate what kind of equipment needs to be procured,
desired features, and supported technologies (e.g. generation, target frequencies, antenna specifications).

### Candidate towers

The foremost limiting factor is the local cellular infrastructure and available operators and their service plans.
This is a detail that I can't control, so I have to stay within these specifications.

For longevity I will also plan slightly ahead, perhaps optimistically, so that the deployment can remain usable even when current technologies are slowly updated to the next generation of radio tech. In practical terms this means that I will simply not consider any LTE/4G only modems.

The following is a brief overview of the site, important geographical features that may impact radio performance, and the two best-candidate cell masts.

![Site map overview](map.png)

From the figure we can see there are three operators in the area between two of the closest towers. This is great for competetive pricing, and
if I ever want to have failover between two service providers (or towers).

Tower 1 (T1) is the immediately more appealing target, due to having two different operators and newer radio technology. 
This information will help me decide on antenna properties and mounting location for the site.

It is unclear how supported Carrier Aggregation (CA) is between the operators and available towers, or what the congestion levels are.
Ultimately real performance can only be reliably defined after practical benchmarking on-site.

### Geographical features

The area is rural, and luckily, there aren't many obstructions that could intefere with connectivity to either tower.

T2 has essentially as clear a line of sight (LOS) as possible to the site, while T1 has a rather sizeable hill *almost* in the way of direct LOS (see previous figure). I doubt any of the present features will pose a problem to connectivity.

The following data is extracted from a contour map to provide relative heights from the site to T1 and T2 (respectively).

![Site to T1 contour data](tower1-contour.png)
![Site to T2 contour data](tower2-contour.png)

We can see that in both cases the ground-level deltas are excellent and the masts have great vantage to the site.

Considering all the previous information, I can be quite confident that the site can get reliable connectivity within my set requirements and goals.

## Choosing hardware

Now that I know the locally supported radio frequencies, generations, and rough bearing of the mast cells, I can begin narrowing down
on what hardware could fulfill my requirements and fit into my budget.

### Modem and routing

Choosing the right modem is crucial for at least the following reasons: 

- Radio generation
    - *Mandate >=5G for longer service life. Even as LTE/4G eventually gets phased out, the device can transition to 5G-only networks.*
- Supported OS
    - *Will define expected service life through availability of system updates.*
    - *Prefer something with FOSS options (e.g. OpenWRT)*
- External antenna connectivity
    - *Crucial for first-rate external antenna support to enhance connectivity.*
- Software support for remote management options
    - *E.g. air heat pump, security cameras*
- GPS support
    - *Would be nice for accurate and reliable network time, but not a must.*

Based on the following checklist and general consensus on the internet, I narrowed the hunt down
to the following companies and their offerings: **Mikrotik**, **Teltonika**, and **GL-iNet**.

Essentially all the options that fit my checklist cost over 350€ as of writing. This initial investment is still
cheaper compared to the deployment of fiber to the location, so all is good. Still, I don't want to pay more than 400€ for the
modem, as the value seems to sharply decline past that mark. I will also have to account for the possible external antenna.

If you saw the banner for this post, you've already been spoiled on what I eventually settled on, but; here is the rundown of the candidates nonetheless:

- Mikrotik Chateau 5G R17 ax
    - IPQ-6010 (arm64), 4 core, 1GB RAM, 128MB NAND.
    - RouterOS (not foss, but trusted and feature rich)
    - Modem: 5G RGE650E-EU (Quectel)
    - MIMO DL: 4x4
    - MIMO UL: 2x2
    - Bands: LTE/5G (excellent), and 3G.
    - Ethernet: WAN(1x2.5Gbit), LAN(4x1Gbit).
    - Price: 360€
    - SMA: acceptable external connectivity (can be modded)
    - Wifi 6
    - Useful online material at [confusedbird (thread 119)](https://confusedbird.com/thread-119.html), for antenna mappings and internal layout.
    - [Flyer](https://cdn.mikrotik.com/web-assets/product_files/Chateau5GR17ax_250616.pdf)
- GL-iNET Spitz AX (GL-X3000)
    - MT7981A (arm64), 2 core, 512MB RAM, 8GB NAND.
    - OpenWRT (modified, should have mainline support)
    - Modem: 5G X62 (Qualcomm)
    - MIMO DL: 4x4
    - MIMO UL: 2x2
    - Bands: LTE/5G (excellent), and 3G.
    - Ethernet: WAN(1x2.5Gbit), LAN(4x1Gbit).
    - Price: 355€
    - SMA: great external connectivity (6x)
    - Wifi 6
    - [Flyer](https://static.gl-inet.com/www/images/products/datasheet/x3000_datasheet_20250704.pdf) (they call this a datasheet, it's not really)
- Teltonika RUTX50
    - IPQ4018 (armhf), 4 core, 256MB RAM, 256MB NAND.
    - RutOS (modified OpenWRT, has mainline support)
    - Modem: RG501Q-EU (Quectel)
    - MIMO DL: 4x4
    - MIMO UL: 2x2
    - Bands: LTE/5G (excellent), and 3G.
    - Ethernet: WAN(1x1Gbit), LAN(4x1Gbit).
    - Price: 500€
    - SMA: excellent (4x cellular, 2x wifi, 1x gps)
    - Wifi 4 (2.4GHz), Wifi 5 (5GHz)
    - [Flyer](https://teltonika-networks.com/cdn/products/2023/01/63b7f88131a497-19366814/flyer-2025-01/rutx50-flyer-2025-v11.pdf), [Datasheet](https://teltonika-networks.com/cdn/products/2023/01/63b7f88131a497-19366814/datasheet-pdf/rutx50-datasheet-v182.pdf)

So what's actually the difference between these models? For the sake of technical details, lets ignore the price outlier.

Both Mikrotik and Teltonika use Quectel modems, which I've had better experiences with compared to Qualcomm. Notable is also the OS, where the Chateau uses Mikrotik's RouterOS, instead of OpenWRT. RouterOS vs OpenWRT is more of a taste question, as I would trust both options for an acceptable service life and quality of features.

Comparing the CPUs, RAM, and NAND next. While having more sounds nice, I'm not entirely convinced of the benefits.
I shouldn't be running a lot of services on the modem itself, and I'd rather throw a small SBC machine onto the network for more advanced services anyway, rather than risk the stability and security of my networks backbone.

Ethernet and Wifi is mostly similar accross the devices, with Teltonika's RUTX50 falling behind quite significantly on the Wifi spec.
Wifi 5 is still "fine" for regular use, and most of my endpoint devices don't have super contemporary modems either, so I'm more inclined to not care. Furthermore, if the on-site network grows beyond this one device it would make more sense to have a dedicated L2 network and WAP, at which point that won't be an issue.

### Antennas

The overwhelmingly strongest option would be a directional antenna aimed at the tower that provides best service on-site (tentatively T1). And that will be the plan, as long as the modem is meant to be left on-site for remote management.

The benefit of an omnidirectional antenna is simply that it could interface with both T1 and T2, and make failover simple. It could also technically be taken with if the modem needs to be used elsewhere more easily.
An omnidirectional antenna is also less likely to be affected by harsh weather and lose its bearing if something really rough happens.

There are a lot of antenna options with varying specifications. Importantly I should focus on the low-frequency 700-800MHz band first, which is used for well-penetrating rural radio connectivity (on both T1 and T2). The antenna should also be 4x4 MIMO, since the candidate modems are all 4x4. It would also be nice if I could somehow bundle Cell, Wifi, and GPS into one antenna, but that's a little overtly optimistic. And it may be more cost effective to buy a separate antenna (or WAP) for Wifi anyhow if long-range is needed for yard use.

## Procurement

### Modem

I chose Teltonika's RUTX50 as the best candidate for the following features: OpenWRT, Rugged chassis, SMA connectivity, and GPS. Regrettably it doesn't have eSIM support, but my management plans don't require them anyhow.

So what about the price? I didn't want to pay over 400€ for the modem, but the MSRP for RUTX50 is between 500-600€. Ah, well. I happened to snag one very lightly (looks new to me) used for 350€ plus shipping locally. So with that, I didn't blow my budget either and managed to get the device I preferred with excellent external connectivity.

![Teltonika RUTX50 external sma-connectors](rutx50-sma.jpg)

### Antennae

Undecided.
