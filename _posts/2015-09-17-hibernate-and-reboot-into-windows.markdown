---
layout: post
title:  "Hibernate and Reboot into Windows"
date:   2015-09-17 15:40:00
categories: systemd
---
Sadly, gaming on Linux is still a rather improvable story.
That means that I still boot into Windows sometimes.
Usually, however, I hibernate my Linux, so just pushing the reboot button is
kind of inconvenient -- especially if I just want Windows for a short break.
But of course it wouldn't be Linux if there was no solution.
There are actually two solutions for two aspects of the problem: rebooting after
hibernating and rebooting into windows.

## Rebooting after hibernating

I found the first information in `man 5 systemd-sleep.conf`.
It says under "OPTIONS":

{% highlight groff %}
The following options can be configured in the "[Sleep]" section of
/etc/systemd/sleep.conf or a sleep.conf.d file:

SuspendMode=, HibernateMode=, HybridSleepMode=
    The string to be written to /sys/power/disk by, respectively,
    systemd-suspend.service(8), systemd-hibernate.service(8), or
    systemd-hybrid-sleep.service(8). [...]
{% endhighlight %}

Just issuing `cat /sys/power/disk` shows some possible values that can go there:

    [platform] shutdown reboot suspend

We want `reboot`.

However, we want it to reboot only once.
The next time we hibernate the system it should shutdown as usual.
As I couldn't figure out a command that does just this, I wrote two scripts:
one that configures Systemd to reboot after hibernation and one that undoes that
when the system is resumed after hibernation.

The first one goes somewhere in our `PATH` e.g.
`/usr/local/bin/hibernate_and_reboot.sh`

{% highlight bash %}
#!/usr/bin/bash

set -e

tee /etc/systemd/sleep.conf.d/50-reboot-on-hibernate.conf << EOF
[Sleep]
HibernateMode=reboot
EOF

systemctl hibernate
{% endhighlight %}

It will create the file `/etc/systemd/sleep.conf.d/50-reboot-on-hibernate.conf`
and fill in the `HibernateMode=reboot` option.

The second one goes into `/usr/lib/systemd/system-sleep` e.g.
`/usr/lib/systemd/system-sleep/remove-reboot-conf.sh`

{% highlight bash %}
#!/bin/sh
case $1/$2 in
	post/hibernate)
		rm /etc/systemd/sleep.conf.d/50-reboot-on-hibernate.conf
		;;
esac
{% endhighlight %}

It will -- once made executable -- delete the file created by the last script as
soon as the system resumes ([related Arch Wiki section](https://wiki.archlinux.org/index.php/Power_management#Hooks_in_.2Fusr.2Flib.2Fsystemd.2Fsystem-sleep)).

## Rebooting into Windows

There are two ways of doing this: using UEFI and using GRUB.

### The UEFI way
In theory, the UEFI way is the more elegant one.
In practice, however, this one often fails me.
That might be because of my old firest-generation-UEFI mainboard.

Here is how it works, anyway:
The command `efibootmgr` will give you a list of all boot entries stored in your
UEFI firmware.[^1]
One entry should look like this:

    Boot0003* Windows Boot Manager

The `0003` is the boot number (in hex).
You can use the following command to tell your firmware what to boot the next
time and the next time only:

    efibootmgr -n 0003

Just add this line with your boot number for Windows to the first script of the
two above (for example just under `set -e`) and you're done.

### The GRUB way
If you run in trouble with your UEFI firmware or even have an old BIOS firmware,
GRUB is there to help you out.

The command `grub-reboot` works very much like `efibootmgr -n`.
It tells GRUB to boot the given entry just one time.
Now the number should match the position of the Windows boot entry in the list
that comes up when you boot into GRUB.
Remember to start counting at 0.
For me the line is:

    grub-reboot 2

Just like above: add this to your reboot script and everything should be set up.

[^1]: If you want to read more about it: [here](https://www.happyassassin.net/2014/01/25/uefi-boot-how-does-that-actually-work-then/) is a quite instructive article that I enjoyed.
