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
in system and program architecture, and the practical realities of needing to get things done.

Unfortunately for the project and the people who have vested time and resources to systems
that depend on it... Systemd has now bowed down to malicious expectations of implementing
the first steps to an authoritarian system that attempts to limit the very basic usage
of personal computers by ordinary citizens. This will not be tolerated, and is very likely
going to be the catalyst to a much needed spring cleaning within the initialisation stack
for at least some Linux distributions. Work that I fully expect to propagate downstream in
due time.

I will personally earmark the next maintenance period for my systems to finally migrate
away from systemd to a simpler paradigm. This was already on my agenda, but now is as good
a time as any to bump that priority up, and get some testing done on my Gentoo workstation.
I already do a lot of extra work just to get around some default behaviour of systemd and
its various components, so I'm looking forward to the opportunities this restructuring will yield.

*P.S. I support transparency, verifiability, and accountability; but, enforcing these facets
at this level of a personal computer is exactly nothing short of a gross violation of personal
freedoms and a massive security risk as the systems are expanded. It clearly displays an utter
lack of understanding or care for the effectiveness of the "architected" solution.*
