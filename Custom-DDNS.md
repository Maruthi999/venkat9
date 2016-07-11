### Introduction
If you set the DDNS (dynamic DNS) service to "Custom", then you can fully control the update process through a `ddns-start` user script (which could launch a custom update client, or run a simple "wget" on a provider's update URL). The ddns-start script is passed the WAN IP as an argument.

Note that your custom script is responsible for notifying the firmware on the success or failure of the process.  To do this your script must execute:

```
/sbin/ddns_custom_updated 0 or 1
```
(where 0 = failure, 1 = successful update)

If you can't determine the success or failure, then report it as a success to ensure that the firmware won't continuously try to force an update.

Finally, like all [[user scripts|User-scripts]], the option to support custom scripts and config files must be enabled under Administration -> System.

After enabling custom scripts, place the contents of your update script in `/jffs/scripts/ddns-start` 


# Sample Scripts


### afraid.org

Here is a working example, for afraid.org's free DDNS (you must update the URL to use your private API key from afraid.org). 

```
----- HTTPS -----                                                                                                           
#!/bin/sh

curl -k "https://freedns.afraid.org/dynamic/update.php?PASTE_YOUR_KEY_HERE" >/dev/null 2>&1 &

if [ $? -eq 0 ]; then
    /sbin/ddns_custom_updated 1
else
    /sbin/ddns_custom_updated 0
fi

--- OR non-HTTPS --- (not recommended as your password is in plain hash).

#!/bin/sh

wget -q http://freedns.afraid.org/dynamic/update.php?your-private-key-goes-here -O - >/dev/null 2>&1 &

if [ $? -eq 0 ]; then
    /sbin/ddns_custom_updated 1
else
    /sbin/ddns_custom_updated 0
fi
```

### [dnsExit.com](http://www.dnsexit.com/Direct.sv?cmd=dynDns)
* **_NOTE: the example below uses non-HTTPS which isn't recommended.  dnsExit.com doesn't have HTTPS method available._**

Free DNS server that also offers DDNS services.
```
#!/bin/sh
USER=
PASS=
DOMAIN=
wget -qO - "http://update.dnsexit.com/RemoteUpdate.sv?login=$USER&password=$PASS&host=$DOMAIN"
if [ $? -eq 0 ]; then
  /sbin/ddns_custom_updated 1
else
  /sbin/ddns_custom_updated 0
fi
```

### Google Domains
Transfer your domain to Google and enjoy free DDNS and other features.
```
#!/bin/sh

set -u

U=xxxx
P=xxxx
H=xxxx

# args: username password hostname
google_dns_update() {
  CMD=$(curl -s https://$1:$2@domains.google.com/nic/update?hostname=$3)
  logger "google-ddns-updated: $CMD"
  case "$CMD" in
    good*|nochg*) /sbin/ddns_custom_updated 1 ;;
    abuse) /sbin/ddns_custom_updated 1 ;;
    *) /sbin/ddns_custom_updated 0 ;;
  esac
}

google_dns_update $U $P $H

exit 0

```

### [DyNS](http://dyns.cx)
* **_NOTE: the example below uses non-HTTPS which isn't recommended.  See example for afraid above._**

provide a number of free and premium DNS related services for home or office use.
```
#!/bin/sh
#
# http://dyns.cx/documentation/technical/protocol/v1.1.php
                
USERNAME=   
PASSWORD=   
HOSTNAME=
DOMAIN=  # optional                       
IP=${1}                                                                                                        
DEBUG= # set to true while testing                                                                                          
                                                                                                               
URL="http://www.dyns.net/postscript011.php?username=${USERNAME}&password=${PASSWORD}&host=${HOSTNAME}&ip=${IP}"
if [ -n "${DOMAIN}" ] ; then   
  URL="${URL}&domain=${DOMAIN}"
fi                         
if [ -n "${DEBUG}" ] ; then
  URL="${URL}&devel=1"     
fi                           
                             
wget -q -O - "$URL"          
if [ $? -eq 0 ]; then        
  /sbin/ddns_custom_updated 1
else                         
  /sbin/ddns_custom_updated 0
fi                           
```

### CloudFlare
If you use CloudFlare for your domains, this script can update any A record on your account.
```
#!/bin/sh

EMAIL= # Your CloudFlare E-mail address
ZONE= # The zone where your desired A record resides
RECORDID= # ID of the A record
RECORDNAME= # Name of the A record
API= # Your CloudFlare API Key
IP=${1}

curl -fs -o /dev/null https://www.cloudflare.com/api_json.html \
  -d "a=rec_edit" \
  -d "tkn=$API" \
  -d "email=$EMAIL" \
  -d "z=$ZONE" \
  -d "id=$RECORDID" \
  -d "type=A" \
  -d "name=$RECORDNAME" \
  -d "ttl=1" \
  -d "content=$IP"

if [ $? -eq 0 ]; then
	/sbin/ddns_custom_updated 1
else
	/sbin/ddns_custom_updated 0
fi
```

### [pdd.yandex.ru](https://domain.yandex.com)
If you use domain.yandex.com for your domains, this script can update any A/AAAA record on your account. Replace `router.yourdomain.com`, `token` and `id` with your own values.
```
#!/bin/sh
# Get token at https://pddimp.yandex.ru/token/index.xml?domain=yourdomain.com
token=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Get record ID from https://pddimp.yandex.ru/nsapi/get_domain_records.xml?token=$token&domain=yourdomain.com
# <record domain="home.yourdomain.com" priority="" ttl="21600" subdomain="home" type="A" id="yyyyyyyy">...</record>
id=yyyyyyyy

/usr/sbin/curl --silent "https://pddimp.yandex.ru/nsapi/edit_a_record.xml?token=$token&domain=yourdomain.com&subdomain=home&record_id=$id&ttl=900&content=${1}" > /dev/null 2>&1
if [ $? -eq 0 ];
then
    /sbin/ddns_custom_updated 1
else
    /sbin/ddns_custom_updated 0
fi
```

### [Namecheap](https://www.namecheap.com)
If you use Namecheap for your domains, this script can update any A record on your account. The script is currently (as of Aug 1 2015) required because the built-in script uses HTTP, while Namecheap requires HTTPS. To use this, replace `HOSTNAME`, `DOMAIN` and `PASSWORD` with your own values. You can refer to the [DDNS FAQ at Namecheap](https://www.namecheap.com/support/knowledgebase/article.aspx/36/11/how-do-i-start-using-dynamic-dns) for steps required. 
```
#!/bin/sh
# Update the following variables:
HOSTNAME=[hostname]
DOMAIN=[domain.com]
PASSWORD=XXXXXXXXXXXXXXXXXXXXXXXX

# Should be no need to modify anything beyond this point
/usr/sbin/wget --no-check-certificate -qO - "https://dynamicdns.park-your-domain.com/update?host=$HOSTNAME&domain=$DOMAIN&password=$PASSWORD&ip="
if [ $? -eq 0 ]; then
  /sbin/ddns_custom_updated 1
else
  /sbin/ddns_custom_updated 0
fi
```

### [DNS-O-Matic](https://www.dnsomatic.com)
If you use DNS-O-Matic to update your domains, this script can update all or a single host record on your account.  To use this, replace `dnsomatic_username`, `dnsomatic_password` with your own values. You can refer to the [DNS-O-Matic API Documentation](https://www.dnsomatic.com/wiki/api#sample_updates) for additional info.

Note: the HOSTNAME specified in the script below will update all records setup in your DNS-O-Matic account to have it only update a single host you will need to modify it accordingly.  In some cases this may require you to specify the host entry, sometimes the domain entry.  

To troubleshoot update issues you can run the curl command directly from the command line by passing in your details and removing the --silent option.  If you get back good and your IP address back you've got it setup correctly.  If you get back nohost, you're not passing in the correct hostname value.
 
```
#!/bin/sh
# Update the following variables:
USERNAME=dnsomatic_username
PASSWORD=dnsomatic_password
HOSTNAME=all.dnsomatic.com

# Should be no need to modify anything beyond this point
/usr/sbin/curl -k --silent "https://$USERNAME:$PASSWORD@updates.dnsomatic.com/nic/update?hostname=$HOSTNAME&wildcard=NOCHG&mx=NOCHG&backmx=NOCHG&myip=" > /dev/null 
if [ $? -eq 0 ]; then
  /sbin/ddns_custom_updated 1
else
  /sbin/ddns_custom_updated 0
fi
```
**Note:** It seems that the DNS-O-Matic API (at least when using a single https command) does _not_ like an email address as the user name and will fail. DNS-O-Matic no longer allows the creation of a separate user name. However there is a workaround: Your DNS-O-Matic account is the same as your OpenDNS account. If you go to _my account_ at [opendns.com](opendns.com) and choose _display name_ (purportedly for forum use), this will also work in this script for user name. The suggestion above about running the _curl_ command directly from the command line to test is really useful!

### [Duck DNS](https://www.duckdns.org)

Just replace ``yoursubdomain`` and ``your-token`` with the values you got from duckdns. The hostname you set up in the GUI doesn't matter, but I recommend setting it to your subdomain anyway.

```shell
#!/bin/sh

# register a subdomain at https://www.duckdns.org/ to get your token
SUBDOMAIN="yoursubdomain"
TOKEN="your-token"

# no modification below needed
curl --silent "https://www.duckdns.org/update?domains=$SUBDOMAIN&token=$TOKEN&ip=" >/dev/null 2>&1
if [ $? -eq 0 ];
then
    /sbin/ddns_custom_updated 1
else
    /sbin/ddns_custom_updated 0
fi
```

### [EasyDNS] (https://www.easydns.com/)
```
#!/bin/sh
#
# This script provides dynamic DNS update support for the EasyDNS service on
# the Merlin asuswrt router firmware.
#
#  
#	Command Line examples you can try in your web browser or CLI
# wget -qO - "http://api.cp.easydns.com/dyn/tomato.php?login=EDIT-ME&password=EDIT-ME&wildcard=no&hostname=EDIT.ME.EM&0ED.IT0.0ME.TOO"
#
# curl -k "http://EDIT-USER:EDIT-PASSWORD@api.cp.easydns.com/dyn/tomato.php?&wildcard=no&hostname=EDIT-ME&myip=0ED.IT0.0ME.TOO"


date >> /tmp/ddns-start.log
echo "$#: $*" >> /tmp/ddns-start.log

# This should be the domain (or hostname) to be updated.
# Seems as you can add more DDNS with this method, This works for me very well
# as I need two A records to be updated from DDNS.
# 	You should be able to add a C, D, etc if needed. 
DOMAIN_A=ADD DOMAIN HERE
DOMAIN_B=ADD 2nd DOMAIN HERE

# This is where your EasyDNS user name and the update token obtained from
# EasyDNS needs to be modified.
EASYDNS_USERNAME=Change to your login name.
EASYDNS_PASSWORD=Change to your taken.

# Set wildcard "on" if you want this to map any host under your domain
# to the new IP address otherwise "off".
WILDCARD=off

# This is set directly from http://helpwiki.easydns.com/index.php/Dynamic_DNS#Setting_up_your_system_to_use_Dynamic_DNS
# Their possibly may be another URI_BASE='https://members.easydns.com/dyn/dyndns.php' 
# I have had no luck with this other URI so far, but the one currently set works great. 
URI_BASE="http://api.cp.easydns.com/dyn/tomato.php"

# This is where your wan IP comes from.
WAN_IP=$1

# This is curl, update to DOMAIN_A
curl --silent -k -u "$EASYDNS_USERNAME:$EASYDNS_PASSWORD" \
        "$URI_BASE?wildcard=$WILDCARD&hostname=$DOMAIN_A&myip=$WAN_IP"

# This is curl update to DOMAIN_B Remove the comment from the last 
# two lines from this section to activate the secound DDNS updater.  
# If you need more updaters you should be able to copy the curl lines, and change
# DOMAIN_B to DOMAIN_X if you are on the same account and server. If not you will 
# Need to make a few other changes for each. 
#curl --silent -k -u "$EASYDNS_USERNAME:$EASYDNS_PASSWORD" \
#        "$URI_BASE?wildcard=$WILDCARD&hostname=$DOMAIN_B&myip=$WAN_IP"

# The last lines tell the web gui that we have or have not updated. 
if [ $? -eq 0 ]; then
        /sbin/ddns_custom_updated 1
else
        /sbin/ddns_custom_updated 0
fi
```

### [Dy.fi] (http://www.dy.fi/)
Just edit USERNAME, PASSWORD and HOSTNAME according to your setup, and you should be good to go. Dy.fi drops hosts after 7 days of inactivity, so I'd also recommend setting the "Forced refresh interval (in days)" setting in the web ui to 7.

```
#!/bin/sh
# http://www.dy.fi/page/specification

USERNAME="yourusername@whatever.com"
PASSWORD="yourtopsecretpassword"
HOSTNAME="yourhostname.dy.fi"

curl -D - --user $USERNAME:$PASSWORD https://www.dy.fi/nic/update?hostname=$HOSTNAME >/dev/null 2>&1

if [ $? -eq 0 ]; then
        /sbin/ddns_custom_updated 1
else
        /sbin/ddns_custom_updated 0
fi

```
### CloudFlare IMPROVED and working as of Dec 30th 2015
If you use CloudFlare for your domains, this script can update any A record on your account. Something was broke in the old script above. If you are having trouble, use this one. To get your record ID use:
```
curl https://www.cloudflare.com/api_json.html -d 'a=rec_load_all' \
  -d 'tkn=YOUR_API_KEY’ \
  -d 'email=YOUR_EMAIL_ADDRESS’ \
  -d 'z=YOUR_DOMAIN_NAME’
```
/jffs/scripts/ddns-start
```
#!/bin/sh

NEW_IP=`wget http://ipinfo.io/ip -qO -`

        curl https://www.cloudflare.com/api_json.html \
          -d 'a=rec_edit' \
          -d 'tkn=YOUR_API_KEY_HERE’ \
          -d 'email=YOUR_ACCOUNT_EMAIL_HERE’ \
          -d 'z=ZONE_OR_ROOT_DOMAIN_NAME’ \
          -d 'id=RECORD_ID’ \
          -d 'type=A' \
          -d 'name=DOMAIN_NAME’ \
          -d 'ttl=1' \
          -d "content=$NEW_IP"
        echo $NEW_IP > /var/tmp/current_ip.txt

if [ $? -eq 0 ]; then
    /sbin/ddns_custom_updated 1
else
    /sbin/ddns_custom_updated 0
fi
```
### BIND9 DDNS using nsupdate
If you run your own DNS server with BIND9, this script uses nsupdate to update an A record.  This requires that you are updating a zone configured for use with dynamic updates rather than the standard zone config files.
```
#!/opt/bin/bash
# A bash script to update BIND9 DDNS using nsupdate and tsig key
# Tested with bash and bind-client to be installed from entware-ng

#User variables - replace with your variables
NS="ns1.example.com"
ZONE="dynamic.example.com"
DHOST="dhost.dynamic.example.com"
TSIGFILE="/tmp/sda1/mykey.tsig"

NSUPDATE=$(which nsupdate)
IP=$1

echo "server $NS" > /tmp/nsupdate
echo "debug yes" >> /tmp/nsupdate
echo "zone $ZONE." >> /tmp/nsupdate
echo "update delete $DHOST A" >> /tmp/nsupdate
echo "update add $DHOST 600 A $IP" >> /tmp/nsupdate
echo "send" >> /tmp/nsupdate

$NSUPDATE -k $TSIGFILE /tmp/nsupdate 2>&1 &
wait $!
echo $?

if [ $?==0 ]; then
	/sbin/ddns_custom_updated 1
else
	/sbin/ddns_custom_updated 0
fi
```

### Loopia 
This scripts add Loopia support using curl just edit hostname and cred.
```
#!/bin/sh
# https://support.loopia.com/wiki/CURL
# Add your credentials here
cred=username:password
hostname=domain.com
# Don't edit anything beyond this point
burl=https://dns.loopia.se/XDynDNSServer/XDynDNS.php
wanip=$(curl -s ipecho.net/plain)
url="$burl"'?hostname='"$hostname"'&'myip="$wanip&wildcard=NOCHG"

loopia_dns_update() {
  CMD=$(curl -s --user "$cred" "$url")
  logger "ddns status: $CMD"
  case "$CMD" in
    good*|nochg*) /sbin/ddns_custom_updated 1 ;;
    abuse) /sbin/ddns_custom_updated 1 ;;
    *) /sbin/ddns_custom_updated 0 ;;
  esac
}
loopia_dns_update
exit 0
```

### DNSimple
This script adds DNSimple support, get token and record_id from the site and edit all the variables.
```
#!/bin/bash
 
LOGIN="your@email"
TOKEN="your-api-token"
DOMAIN_ID="yourdomain.com"
RECORD_ID="12345" # Replace with the Record ID
IP=`curl -s http://icanhazip.com/`
 
curl --silent \
     -H "Accept: application/json" \
     -H "Content-Type: application/json" \
     -H "X-DNSimple-Token: $LOGIN:$TOKEN" \
     -X "PUT" \
     -i "https://api.dnsimple.com/v1/domains/$DOMAIN_ID/records/$RECORD_ID" \
     -d "{\"record\":{\"content\":\"$IP\"}}" > /dev/null

if [ $? -eq 0 ]; then
    /sbin/ddns_custom_updated 1
else
    /sbin/ddns_custom_updated 0
fi
``` 

### Double NAT - External IP Example
This script first retrieves the external IP rather than using the one passed to it from the Custom DDNS settings.  This may be necessary if your ASUS router is double NATed behind your ISP's router.  Example makes use of DNS-O-Matic but could be modified to work with other DDNS providers.
```
#!/bin/sh
USER="YourEmail%40domain.com" # replace @ symbol with URL safe %40
PASS="YourPassword"
HOST="all.dnsomatic.com"

# Should be no need to modify anything beyond this point
IP=$(wget -O - -q http://myip.dnsomatic.com/)
logger "Retrieved External IP: $IP"

RESULT=$(/usr/sbin/curl -k --silent "https://$USER:$PASS@updates.dnsomatic.com/nic/update?hostname=$HOST&wildcard=NOCHG&mx=NOCHG&backmx=NOCHG&myip=$IP")

logger "Results: $RESULT"

if [[ ${RESULT:0:4} == 'good' ]]
then
  /sbin/ddns_custom_updated 1
else
  /sbin/ddns_custom_updated 0
fi
``` 