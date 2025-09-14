+++
title = "Workstation Configuration"
date = 2025-01-30

type_img = ""
banner_c = "An image of a pretty typical laptop."

tags = ["infrastructure", "linux", "sysadmin", "gentoo"]
draft = false
+++

## Baseline Description

This post details the current active configuration for my primary endpoint device,
listing the planned general structure, required capabilities, and implementation details.
It is the first in series of three and functions as the trusted cornerstone and platform for
the implementation and management of the remaining two and everything beyond.

## Goals

The goal is quite simple, to have an above all reliable and secure portable device that can be used
as a daily-driver for most computer-related tasks in most sensible environments. As such
some of the goals are influenced by the currently relevant workloads, but not by much.
I'm aiming for an all-rounder here.

Computers are so powerful nowadays that sufficient performance should be attainable on a wide
variety of hardware, so general efficiency should be prioritised where possible.
Anything more specialised workload-wise should be offloaded to purpose-specific machines with dedicated hardware
or connectivity that is optimal for the workload (e.g. GPU compute, intensive disk IO, highly parallel CPU, fast networking).

I refuse to use anything other than Open-Source software on the machine (with possible minor exceptions),
as the configuration has to be regarded as secure. So realistically the platform and hardware must have good Linux support. Anything by Microsoft or Apple is automatically disqualified due to their penchant for pushing arbitrary code
to their hardware and software platforms in a way that can't be disabled reliably, among other things like political affiliations (e.g. kowtowing, lobbying, etc.) and corporate interests (e.g. "Embrace, Extend, Extinguish", Halloween Papers, Data Gathering, etc.).

### Hardware

#### CPU, GPU

As there is a general emphasis on being efficient, the machine must have good battery life,
no dedicated GPU, and a CPU that is either very efficient out of the box or can be tuned to be so
without excessive workarounds via user space tools (e.g. TLP).

The integrated GPU should be performant enough for basic graphical output of a desktop environment, and support
hardware accelerated decoding of contemporary video codecs (AV1, HEVC) main profiles.

#### RAM

Secondly, sufficient RAM must be available for extensive multitasking for the following categories
as I don't want my workflows to be constrained by the computer I use. Medium- and long-term these
constraints would eat away too much time and opportunities to warrant the cost-savings on the hardware side.

- Multitasking Categories
    - Web (Tabs, VoIP, Streaming multimedia)
    - IDE and Building Software
    - Virtualisation (VMs, Containers and Orchestration)
    - Streaming low-latency remote desktop
    - Moderate signal and data processing (text, video, audio)
    - OS and Filesystem features (caching, /tmp, other ramdisks)

The aforementioned categories comprise most things I imagine doing on the regular, and typically
I can push my RAM usage on the desktop to be slightly above 40GB when at home and hooked to an external
display with couple workflows open simultaneously. This effectively mandates that the computer must have
more than 32GB of RAM.

#### Connectivity, IO, HID

There should be at least a single USB-A and USB-C port, a full-size HDMI port, and an Ethernet port.
The WiFi radio must have reliable drivers for Linux with a Bluetooth component that works simultaneously
without causing issues like stuck states or inability to reset.

Most solidstate drives are perfectly fine for regular use, so anything made by a reliable manufacturer should
suffice. No particular requirements for latencies or throughput, capacity should not exceed 1TB.

Screen should not exceed "FHD" resolutions to not waste battery life, and it should prioritise power efficiency over
high refresh rates, low response times, and extended colour gamuts (e.g. DCI-P3, BT2020); but, still aim to provide accurate sRGB representation.

Keyboard has to have a numpad for productivity, and be sensibly laid out (standard format!).
Omission of Delete, Insert, or End keys are automatic disqualifications, as they are often required
for remote management software or important keybinds depending on the UI.


### Software

From my experience, no single distribution manages to be perfect. There are specific benefits and shortcomings
to each of the popular distributions. I would also be vary of being stuck with any single one of them,
as the direction of a project can't reliably be guaranteed to stay true over the course of years and decades.

I've therefore decided to approach the problem with a light layer of abstraction, where I define sets of
packages and construct the environment successively from those sets.
This gives me the benefit of designing the environment free from distribution specific limitations, and
allows me to then later see what options best match my choices and approach.

My prioritised choices for distributions are:

- Distribution ranking (as of 2025)
    - Gentoo
        - Flexible base system profiles for variety of use cases
        - Powerful ebuild repositories (local and third-party)
        - Native support for hierarchies (world <- sets <- atoms)
        - Fine-grained control over pkg features via build time flags 
    - Debian
        - Strong FOSS emphasis
        - Stability and longevity
        - Trusted project leadership structure
    - Arch
        - Designed to provide full control over system components
        - Forceful proximity to upstream
        - Massive catalogue of software via AUR
        - Popular with lots of excellent documentation and guides
    - Fedora
        - Aggressively recent pkg versions
        - Relatively strong stability
        - Unconcerned with FOSS if functionality is on the line
        - Corporate bias

There's plenty of overlap between the previously listed distributions, and the choice between one
or the other is very much up to individual discretion. For my case, I need to toggle some build time
features that aren't typically enabled on distributions like Debian or Arch. This has steered me
towards Gentoo in the recent year, as it allows me to simplify my setup so every package can use the
same build and management system, instead of having multiple parallel pipelines that need to be managed
independently of each other.

If package specific build time configuration isn't a concern, basically any one of the aforementioned distributions
is a perfectly viable and functional base to work off. And in that case I would prefer something more
straightforward like Debian or Arch, depending on how many and how recent the packages need to be.

If the hardware platform requires ISV-certifications or is otherwise employing more closed source components, Fedora
would perhaps be the most stable option. Sadly I can no longer recommend Ubuntu for typical usage, as Canonical
has a habit of making very questionable choices when it comes to the direction of the distribution. Just be mindful
that Ubuntu is often used as the de facto platform for standardised workflows that employ more complicated components 
or supersets of software like CUDA, Cloud Native, or Machine Learning.

## Configuration Details

Pending review and proofread

### System Components

Pending review and proofread

### User Components

Pending review and proofread

## Implementation and Management

Pending review and proofread
