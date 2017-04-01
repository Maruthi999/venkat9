> These scripts are considered legacy scripts, they have no maintainers and getting support on them might get tricky, they also only supports ipset version 4 so if you have a new router these scripts will not work, please consult the chart on [here](https://github.com/RMerl/asuswrt-merlin/wiki/Using-ipset)

# Peer Guardian

Another example is a [PeerGuardian](http://en.wikipedia.org/wiki/PeerGuardian) functionality right on router.

Please do not add this script to `/jffs/scripts/firewall-start` because it executes too long (~25 min on RT-N66U). Place following content to `/jffs/scripts/peerguardian.sh`
```
#!/bin/sh

# Loading ipset modules
lsmod | grep "ipt_set" > /dev/null 2>&1 || \
for module in ip_set ip_set_iptreemap ipt_set
do
    insmod $module
done

# Different routers got different iptables syntax
case $(uname -m) in
  armv7l)
    MATCH_SET='--match-set'
    ;;
  mips)
    MATCH_SET='--set'
    ;;
esac

# PeerGuardian rules
if [ "$(ipset --swap BluetackLevel1 BluetackLevel1 2>&1 | grep 'Unknown set')" != "" ]
then
    ipset --create BluetackLevel1 iptreemap
    [ -e /tmp/bluetack_lev1.lst ] || wget -q -O - "http://list.iblocklist.com/?list=bt_level1&fileformat=p2p&archiveformat=gz" | \
        gunzip | cut -d: -f2 | grep -E "^[-0-9.]+$" > /tmp/bluetack_lev1.lst
    for IP in $(cat /tmp/bluetack_lev1.lst)
    do
        ipset -A BluetackLevel1 $IP
    done
fi
iptables -I FORWARD -m set $MATCH_SET BluetackLevel1 src,dst -j DROP
```
and run it:
```
sh /jffs/scripts/peerguardian.sh
```
Please don't close SSH-session until it finishes. Script will blocks [over 8 000 000 IP's addresses](http://www.iblocklist.com/list.php?list=bt_level1) which anti-p2p activity has been seen from.

***
# Peer Guardian V2
Supports only IPSET 4 

Below is a speed optimized version of the peerguardian.sh script above. It does the same thing, but takes less than 30 seconds to run (the shortest run took 20 seconds on my RT-N66U). It might now be possible to run it from `/jffs/scripts/firewall-start`.

The script utilizes two sets: primary "BluetackLevel1" and temporary "BluetackLevel2". The IPs are bulk loaded into the temporary one and then swapped into the primary. Because of this approach it can also be run periodically on a running router to refresh the active set.
```
#!/bin/sh

# PeerGuardian rules

# Loading ipset modules
lsmod | grep "ipt_set" > /dev/null 2>&1 || \
for module in ip_set ip_set_iptreemap ipt_set; do
    insmod $module
done

# Different routers got different iptables syntax
case $(uname -m) in
  armv7l)
    MATCH_SET='--match-set'
    ;;
  mips)
    MATCH_SET='--set'
    ;;
esac

# Create the BluetackLevel1 (primary) if does not exists
if [ "$(ipset --swap BluetackLevel1 BluetackLevel1 2>&1 | grep 'Unknown set')" != "" ]; then
  ipset --create BluetackLevel1 iptreemap && \
  iptables -I FORWARD -m set $MATCH_SET BluetackLevel1 src,dst -j DROP
fi
# Destroy this transient set just in case
ipset --destroy BluetackLevel2 > /dev/null 2>&1
    
# Load the latest rule(s)

(echo -e "-N BluetackLevel2 iptreemap\n" && \
 nice wget -q -O - "http://list.iblocklist.com/?list=bt_level1&fileformat=p2p&archiveformat=gz" | \
    nice gunzip | nice cut -d: -f2 | nice grep -E "^[-0-9.]+$" | \
    nice sed 's/^/-A BluetackLevel2 /' && \
 echo -e "\nCOMMIT\n" \
) | \
nice ipset --restore && \
nice ipset --swap BluetackLevel2 BluetackLevel1 && \
nice ipset --destroy BluetackLevel2

exit $?
```
***
# Peer Guardian V3
Supports only IPSET 4 

If you want to have different blocklist, grouped by one, then this is a variant, where you can add multiple blocklists in one script.

```
#!/bin/sh

logger "PeerGuardian rules"

logger "Loading ipset modules"
lsmod | grep "ipt_set" > /dev/null 2>&1 || \
for module in ip_set ip_set_iptreemap ipt_set; do
    insmod $module
done

case $(uname -m) in
  armv7l)
    MATCH_SET='--match-set'
    ;;
  mips)
    MATCH_SET='--set'
    ;;
esac

logger "Create the BluetackLevel1 (primary) if does not exists"
if [ "$(ipset --swap BluetackLevel1 BluetackLevel1 2>&1 | grep 'Unknown set')" != "" ]; then
  ipset --create BluetackLevel1 iptreemap && \
  iptables -I FORWARD -m set $MATCH_SET BluetackLevel1 src,dst -j DROP
fi
logger "Destroy this transient set just in case"
ipset --destroy BluetackLevel2 > /dev/null 2>&1

logger "Load the latest rule(s)"

(
	 
	(
		(
		 nice wget -q -O - "http://list.iblocklist.com/?list=dgxtneitpuvgqqcpfulq&fileformat=p2p&archiveformat=gz" | \
	         nice gunzip | nice cut -d: -f2 | nice grep -E "^[-0-9.]+$"  \
		) && \
		(
	 	 nice wget -q -O - "http://list.iblocklist.com/?list=llvtlsjyoyiczbkjsxpf&fileformat=p2p&archiveformat=gz" | \
        	 nice gunzip | nice cut -d: -f2 | nice grep -E "^[-0-9.]+$"  \
		) && \
		(
		 nice wget -q -O - "http://list.iblocklist.com/?list=ydxerpxkpcfqjaybcssw&fileformat=p2p&archiveformat=gz" | \
		 nice gunzip | nice cut -d: -f2 | nice grep -E "^[-0-9.]+$"  \
		)
	) | \
	(  
      		nice sed '/^$/d' | \
	        nice sed 's/^/-A BluetackLevel2 /' | \
		nice sed '1s/^/-N BluetackLevel2 iptreemap\n/' && \
		echo -e "\nCOMMIT\n" \
	)
#) > output
) | \
nice ipset --restore && \
nice ipset --swap BluetackLevel2 BluetackLevel1 && \
nice ipset --destroy BluetackLevel2

logger "exiting Peerguarding rules"
exit $?
```
The output will be in router logs:
```
May 29 09:03:21 admin: PeerGuardian rules
May 29 09:03:21 admin: Loading ipset modules
May 29 09:03:21 admin: Create the BluetackLevel1 (primary) if does not exists
May 29 09:03:22 admin: Destroy this transient set just in case
May 29 09:03:22 admin: Load the latest rule(s)
May 29 09:04:04 admin: exiting Peerguarding rules
```