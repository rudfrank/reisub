---
title: 'Setting up fingerprint login on Void Linux'
description: "My ThinkPad has a fingerprint sensor, let's make use of it."
date: '2026-10-28T11:08:52+01:00'
tags: ["linux","void","pam","fingerprint","login","tips"]
author: "rudfrank"
draft: false
showToc: false
---
## Install and enroll

First we need to install the fingerprint daemon and the udev rules for the device:
`xbps-install -S fprintd libfprint-udev-rules`

Now we enroll the fingeprint: `fprintd-enroll `
It defaults to right index finger first, you can choose with `fprintd-enroll --finger`

Let's verify whether the enrolling worked: `fprintd-verify`

## Configure PAM

Listing the files under `/etc/pam.d`, you can see different configs for given auth methods and services. \
I will edit `/etc/pam.d/system-local-login`, for my dankshell config it works on login and on the lock screen, but your mileage can vary (you might need to edit your loginmanager's file).

I won't use it with sudo, as it is a security breach: https://nvd.nist.gov/vuln/detail/cve-2024-37408

All guides I've found says that you should put `auth sufficient pam_fprintd.so` as the first line, for fingerprint auth to work. \
Indeed, this works, but adding it as *sufficient* first method, when a fingeprint is enrolled, it will **only** prompt for fingerprint auth.

I want to be able to use a password as normally. After some trial and error, now my system-local-login looks like this -- notice the first two lines:
```pam
auth      sufficient 	pam_unix.so try_first_pass likeauth nullok
auth      sufficient pam_fprintd.so
auth      include   system-login
account   include   system-login
password  include   system-login
session   include   system-login
```

Now I am able to type my password, but if I press enter (without any password), it asks for my fingerprint.
