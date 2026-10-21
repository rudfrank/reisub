---
title: 'Fix missing file-picker on niri on Void'
description: "Using niri on wayland, no filepicker appears eg. in Chromium. Let's fix it!"
date: '2026-10-20T22:32:34+02:00'
tags: ["linux","void","wayland","tips"]
author: "rdf"
draft: false
showToc: false
---

If ran in terminal, when it should open a dialog, it drops a warning: ``...components/dbus/xdg/request.cc:173] Request ended (non-user cancelled)``.

I figured I need to install xdg-desktop-portal-gtk:
``xbps-install -S xdg-desktop-portal-gtk``

After relogin, still no luck. Grepping around configs, I stumbled upon ``/usr/share/xdg-desktop-portal/niri-portals.conf``:
```cfg
[preferred]
default=gnome;gtk;
org.freedesktop.impl.portal.Access=gtk;
org.freedesktop.impl.portal.Notification=gtk;
org.freedesktop.impl.portal.Secret=gnome-keyring;
```
I won't edit it as it is a package-supplied file; cp it to /etc/xdg-desktop-portal/niri-portals.conf, then add the line:
```cfg
org.freedesktop.impl.portal.FileChooser=gtk;
```
After relogin, dialogs work as intended.

