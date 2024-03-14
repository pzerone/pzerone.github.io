+++
title = 'LTSP: Setting up auto login for clients without a Display Manager'
date = 2024-03-13T21:39:34+05:30
draft = false
+++

## Introduction
LTSP's default implementation of auto login is technically *not* auto login. It requires the user to press *Enter* on the dislay manager page. We can overcome this limitation by getting rid of the display manager altogether and use `agetty` to properly auto login the user to their shell.

Auto login can be implemented on the client to login to the TTY automatically without user interaction. From there we can start xorg or run custom scripts using our shell's profile file: ie `bash_profile`. For this to work, we need:
- Auto login with agetty using systemd drop-in file on the client OS
- Auto login for LTSP client, configured on the server OS
- A user for the clients to auto log in to, created on the server OS

>*The first one alone won't be able to run our shell profile file since without proper SSHFS/NFS mount, we will get dropped into `/` without proper home directory*

>*The second one alone won't technically "auto-login" the user, it just wait for the user to press enter on the login screen and hence we need user interaction.*

## The setup
First, start the VM and add a systemd drop-in file
- systemd drop-in file *(To be placed inside the client OS image)*:

`/etc/systemd/system/getty@tty1.service.d/autologin.conf`
```
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -f -- \\u' --noclear --autologin username %I $TERM
```
> *replace **username** with the username we want to auto-login*

- For LTSP auto login, create the `ltsp.conf` file at `/etc/ltsp/ltsp.conf` using
```bash
install -m 0660 -g sudo /usr/share/ltsp/common/ltsp/ltsp.conf /etc/ltsp/ltsp.conf
```
After that, add the following code to the `[client]` section of `ltsp.conf` 
```
[*]                           # Wild card to catch all clients
HOSTNAME=client               # host name for clients
PASSWORDS_client=ltc/cm9vdA== # username/base64 encoded password
AUTOLOGIN=ltc                 # enables autologin for the above user
```
> *replace **client** with a hostname for the clients and **ltc** with the user that we want to auto login.*
> *Password for the user should be converted to base64. Pipe it into `base64` from the commandline like this:*

```bash
echo -n password | base64
```
> *Replace **password** with your user password*

- Regenerate image and initrd
```bash
ltsp image <image name>
ltsp initrd
```
Done and dusted. Reboot your client and boot from your updated image

## Limitations

#### Limited to single user
Though this is a reliable approch to auto login to the client machine, we are limited to a single user since the systemd drop-in file can only have a single user.

Unless we create multiple image for multiple clients, change the usernames on those images and finally change the LTSP config to auto login based on the client MAC address instead of using wildcard method.

#### Works only on VM image method
If we try to add the drop-in file to our server's systemd config, our server will start auto logging into that user from the next boot onwards, which is not what we intend to do. Hence a VM image is required for this method to work