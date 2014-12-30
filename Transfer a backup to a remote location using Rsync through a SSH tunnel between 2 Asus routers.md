In this tutorial, we will learn how to setup a remote backup channel between 2 ASUS routers using Rsync as transporter.

Rsync is a "classic" linux program used to synchronize efficiently data between two locations. The 2 locations could be "local", even on the same server/router, but our goal being to secure a backup, we will do it to a remote location. We will also secure the transfer itself by using SSH. Secure Shell (SSH) is a cryptographic network protocol for secure data communication, remote command-line login, remote command execution, etc. To make it even more secure, we will use a private/public key schema (instead of id/password).

The routers will be named:
* RT-1080
* RT-8075

Both need Rsync installed. RT-1080 will be the "master", i.e. the one responsible for all actions going on between the 2 routers and RT-8075 will be the "slave", just reacting to RT-1080 requests. Each router has a 4TB USB3 disk (on the USB3 port, indeed). Each disk has 2 partitions: 
* the "data partition" (~4TB)
* the "router partition" (~5GB)

In order to run Rsync, we will need a swap space because the router memory could be too limited for the process, depending on the size of the backup. The swap space will be set on a file located on the router partition, and optware/rsync will also be installed on that partition. Simply said, Optware is a mechanism that simplifies the installation of some programs on the router.

The data partition is used to receive:
* the local backup: bckp-1080
* the remote backup: bckp-8075

In my specific case, Windows file story is the tool used to backup two local PC, and same thing on the remote side.  The goal is therefore to send bckp-1080 to RT-18075, and bckp-8075 to RT-1080. The backups will be scheduled to run daily at night using CRON. CRON is the standard Linux tool used to schedule processes on a periodic way.

We will also use JFFS, fstab and the event triggered scripts of RMerlin version.

JFSS is a permanent space used to store some user's data/program/scripts. Intrinsically, the router is a device with no permanent memory, memory being erased on each shutdown, with the exception of the JFFS space. Please note that the JFFS space has to be also copied because it could be overwritten by a new firmware version.

Fstab is a special configuration file used to mount the disks. The scripts are event-triggered and will be used to initiate some actions on the router reboot, and on the post-mount of the disk/partitions. Two scripts will be setup:
* init-start
* post-mount

Let's start the procedure:
* Enable ssh on both routers
* Create an Asus ddns on both routers (xxx.asuscomm.com)
* Install a terminal to access the router through ssh (Putty or Xshell)
* Connect to the router
* Enable JFFS
* Format JFFS
* Install Optware (by installing, and then removing, Download Master)
* Install Rsync
* Create and install private and public rsa keys for both routers (using Easy-RSA)
* Test the keys to make sure it's possible to login without a password
* Create a script to run at boot time to schedule the backup
* Create the Rsync script
* Schedule backup
* 
chmod a+rx /jffs/scripts/services-start
/jffs/scripts/services-start