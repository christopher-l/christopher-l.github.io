---
layout: post
title:  "Gnome's Alt-Tab Delay"
date:   2015-09-12 18:10:23
categories: gnome
---
I have now been waiting for about four years for Gnome to fix a particular bug. At least I thought it was a bug and yeah -- I have been to lazy to check. Until now.

The bug that has kept nagging me for so long is about Gnome's alt-tab behavior. No, I don't want the old one from Gnome 2 back. If anything I am glad that finally went away in Gnome 3. What I want back is the task switcher from Gnome 3.0. The one that would pop up instantly and not after a short delay of some 100 milliseconds. I always assumed that it was something about my setup -- a proprietary graphics card driver maybe -- but as it turns out, this is actually intentional.

As it does so often, Stack Exchange had the [first part](https://askubuntu.com/questions/90213/is-there-any-way-to-make-the-alt-tab-window-changer-pop-up-quicker-in-gnome-shel) of the puzzle for me:

> Well, you can change that in the .js file of the alt-tab.
>
> In
>
> ```
> /usr/share/gnome-shell/js/ui/altTab.js
> ```
>
> Almost at the beginning of the file, is this line
>
> ```
> const POPUP_DELAY_TIMEOUT = 150; // milliseconds
> ```
>
> Change it, save and restart the gnome-shell (Alt+F2 type 'r', or logout and login again).

Man felt I stupid.

But not as much as I felt disillusioned after having typed `cd /usr/share/gnome-shell/` and hit tab. There's no `js` folder there. Of course it isn't.

Luckily two posts further down, someone knew what's going on:

> Since gnome 3.12, the Javascript UI files are embedded inside libgnome-shell.so. See [this post](https://blogs.gnome.org/mclasen/2014/03/24/keeping-gnome-shell-approachable/).

So no more javascript files lying around for me to edit. The other end of the link, however, turned out to be quite instructive. It is a blog post about the change and it even gives a script that will extract all the files from `libgnome-shell.so`:

{% highlight bash %}
#! /bin/sh

gs=/usr/lib64/gnome-shell/libgnome-shell.so

cd $HOME/gnome-shell-js

mkdir -p ui/components ui/status misc perf extensionPrefs gdm

for r in `gresource list $gs`; do
  gresource extract $gs $r > ${r/#\/org\/gnome\/shell/.}
done
{% endhighlight %}

After that one should be able to start the shell with

```
GNOME_SHELL_JS=$HOME/gnome-shell-js gnome-shell
```

I didn't exactly know where I would start `gnome-shell` as usually GDM does that for me. Eventually `gnome-shell --replace` from within a running Gnome session did the trick.

So now there's only left to make it apply this automatically. After some fiddling I finally found the [answer](https://wiki.archlinux.org/index.php/GDM#Add_or_edit_GDM_sessions) in the Arch Wiki: In the `Exec` statement of a `.desktop` file describing an `xsession`, one can define environment variables.

# Here is what I did:

* Create a new file:

      $ $EDITOR extract-gnome-shell-js.sh

  and put the following in it (I changed it slightly):

      #!/bin/bash
      set -e

      gs=/usr/lib64/gnome-shell/libgnome-shell.so

      mkdir $HOME/gnome-shell-js
      cd $HOME/gnome-shell-js

      mkdir -p ui/components ui/status misc perf extensionPrefs gdm portalHelper

      for r in `gresource list $gs`; do
        gresource extract $gs $r > ${r/#\/org\/gnome\/shell/.}
      done


* Make it executable and run it:

      $ chmod +x extract-gnome-shell-js.sh
      $ ./extract-gnome-shell-js.sh

* Open the blameworthy file:

      $ $EDITOR gnome-shell-js/ui/switcherPopup.js

  and change the line

      const POPUP_DELAY_TIMEOUT = 150; // milliseconds

  to

      const POPUP_DELAY_TIMEOUT = 0; // milliseconds

* Move the folder somewhere system wide (as root):

      # mkdir -p /usr/local/share/gnome-shell/
      # mv gnome-shell-js /usr/local/share/gnome-shell/js
      # chown -R root:root /usr/local/share/gnome-shell/js

* Create a new `xsession` file:

      # $EDITOR /usr/share/xsessions/gnome-custom-js.desktop

  and change it to

      [Desktop Entry]
      Name=GNOME Custom JS
      Comment=Gnome session setting GNOME_SHELL_JS
      Exec=env GNOME_SHELL_JS=/usr/local/share/gnome-shell/js gnome-session
      TryExec=gnome-session
      Icon=
      Type=Application
      DesktopNames=GNOME

* Reboot the system (can probably be avoided but you know how things are).

* When GDM comes up, click your name, then click the settings icon and choose `GNOME Custom JS`.

* Punch in your password and hope for the best.
