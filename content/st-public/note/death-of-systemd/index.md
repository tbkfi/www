+++
title = "The Death of Systemd"
date = 2026-03-25

emoji = ""
banner_c = ""

tags = ["linux", "systemd", "init", "sysadmin", "openrc", "infosec"]
draft = false
+++

Systemd has long been tolerated as a neccessary evil for the tools and integration it has
provided for managing Linux systems. Historically and individually these overreaches have
been rather quaint, for example redundant system programs for DHCP, NTP, etc., or the various
control interfaces for managing system users, services, and jobs. Some of these have been
quite all right, like journalctl and its binary logs --maybe even systemd-boot, but only just.
Still, no one in their right mind would deny that all of this is far beyond the scope of what an
init system ought to be doing. Its usage has been a calculated compromise between idealism
in system and program architecture, and the practical realities of needing to get things done
and keep the systems managed.

Unfortunately for the project and the people who have vested time and resources to systems
that depend on it... Systemd has now bowed down to malicious expectations of implementing
the first steps to an authoritarian system that attempts to limit the very basic usage
of personal computers by ordinary citizens. This will not be tolerated, and is very likely
going to be the catalyst to a much needed spring cleaning within the initialisation stack
for at least some Linux distributions. Work that I fully expect to propagate downstream in
due time.

The issue is simply that once the mechanism and the database --no matter how rudimentary-- is in place,
extending it will become a much more trivial matter. Right now its spearheaded by a seemingly innocuous need
to make sure that underage users aren't able to access or be exposed to harmful content by storing
the user's age. But we all know they're not implementing an OS-level API and a local database just to store
a single interger. Next they'll push for a country code, so radio transmit powers can be enforced automatically
at an OS-level to comply with country specific regulations which, again, seems reasonable at face value.
And so with each round of additinal information added to the database, the machine's fingerprint becomes more and more unique.

This entire plan is a deeply misguided and perverted version of the need for strong digital authentication.
The current enforcement originates from and is lobbied for by people who are corrupt and have evil intentions.
Arguably that's exactly the reason why they're lobbying for the corrupt implementation in the first place.
They hope to exert control over the de-facto authentication system so they can eventually build in circumventions to it,
and have justifications why the properly architected system doesn't need to get developed because *a* system already exists.

I will personally earmark the next maintenance period for my systems to finally migrate
away from systemd to a simpler paradigm. This was already on my agenda, but now is as good
a time as any to bump that priority up, and get some testing done on my Gentoo workstation.
I already do a lot of extra work just to get around some default behaviour of systemd and
its various components, so I'm looking forward to the opportunities this restructuring will yield.

*P.S. I support transparency, verifiability, and accountability; but, enforcing these facets
at this level of a personal computer is exactly nothing short of a gross violation of personal
freedoms and a massive security risk as the systems are expanded. It clearly displays an utter
lack of understanding or care for the effectiveness of the "architected" solution.*

