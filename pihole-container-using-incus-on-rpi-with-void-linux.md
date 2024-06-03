## How to configure pihole w/ unbound in an incus container on the rasberry pi
-----------------------------------------------------------------------------

Note: i compile my own custom void linux image for the rpi that includes incus and some other tweaks. [1]

### Method 1: Default networking 
--------------------------------

1. create a new debian container - call it pihole
```
  incus launch images:debian/bookworm/cloud pihole
```

3. enter the container and install pihole 
```
  incus exec pihole -- /bin/bash
  dpkg-reconfigure tzdata
  sudo apt-get install -y wget
  wget -O basic-install.sh https://install.pi-hole.net
  chmod +x basic-install.sh:w
  ./basic-install.sh
  exit
```

4. Forward DNS traffic to the container
Note this assumes the default bridge incusbr0 is being used.  
```  
  # incus network forward create <network> <listen-address>
  incus network forward create incusbr0 192.168.22.25
  incus network forward port add incusbr0 192.168.22.25 tcp 53 10.86.130.228 53
  incus network forward port add incusbr0 192.168.22.25 udp 53 10.86.130.228 53
  incus network forward port add incusbr0 192.168.22.25 tcp 8080 10.86.130.228 80
```

5. That's it. create a snapshot
```
  incus snapshot create pihole snapshot00
  incus snapshot list pihole
```

---

### Method 2: Macvlan networking
--------------------------------

```
[void@rpi-74f50651 ~]$ incus launch images:ubuntu/noble pihole --network macvlan
[void@rpi-74f50651 ~]$ incus list
+------------+---------+-----------------------+------+-----------+-----------+
|    NAME    |  STATE  |         IPV4          | IPV6 |   TYPE    | SNAPSHOTS |
+------------+---------+-----------------------+------+-----------+-----------+
| pihole     | RUNNING | 192.168.20.144 (eth0) |      | CONTAINER | 0         |
+------------+---------+-----------------------+------+-----------+-----------+


[void@rpi-74f50651 ~]$ incus exec pihole -- bash
root@pihole:~#
root@pihole:~# apt-get install -y wget
root@pihole:~# wget -O basic-install.sh https://install.pi-hole.net
root@pihole:~# chmod +x basic-install.sh
root@pihole:~# ./basic-install.sh
root@pihole:~# exit
exit
[void@rpi-74f50651 ~]$

# that's it. network forwards are not necessary when using macvlan.
# just point the clients DNS to the pihole address
```

## Next, configure Pi-hole with unbound
---------------------------------------

Pi-hole is great, sure, but it's even better when paired with unbound for a
recusrive DNS solution. [2]

1. Install unbound 
```
  incus exec pihole -- apt install -y unbound
```

2. Configure unbound
```
incus file push conf/unbound.conf pihole/etc/unbound/unbound.conf.d/pi-hole.conf 

# create the file locally conf/unbound.conf with below contents:
  server:
	# If no logfile is specified, syslog is used
	# logfile: "/var/log/unbound/unbound.log"
	verbosity: 0

	interface: 127.0.0.1
	port: 5335
	do-ip4: yes
	do-udp: yes
	do-tcp: yes

	# May be set to yes if you have IPv6 connectivity
	do-ip6: no

	# You want to leave this to no unless you have *native* IPv6. With 6to4 and
	# Terredo tunnels your web browser should favor IPv4 for the same reasons
	prefer-ip6: no

	# Use this only when you downloaded the list of primary root servers!
	# If you use the default dns-root-data package, unbound will find it automatically
	#root-hints: "/var/lib/unbound/root.hints"

	# Trust glue only if it is within the server's authority
	harden-glue: yes

	# Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
	harden-dnssec-stripped: yes

	# Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
	# see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
	use-caps-for-id: no

	# Reduce EDNS reassembly buffer size.
	# IP fragmentation is unreliable on the Internet today, and can cause
	# transmission failures when large DNS messages are sent via UDP. Even
	# when fragmentation does work, it may not be secure; it is theoretically
	# possible to spoof parts of a fragmented DNS message, without easy
	# detection at the receiving end. Recently, there was an excellent study
	# >>> Defragmenting DNS - Determining the optimal maximum UDP response size for DNS <<<
	# by Axel Koolhaas, and Tjeerd Slokker (https://indico.dns-oarc.net/event/36/contributions/776/)
	# in collaboration with NLnet Labs explored DNS using real world data from the
	# the RIPE Atlas probes and the researchers suggested different values for
	# IPv4 and IPv6 and in different scenarios. They advise that servers should
	# be configured to limit DNS messages sent over UDP to a size that will not
	# trigger fragmentation on typical network links. DNS servers can switch
	# from UDP to TCP when a DNS response is too big to fit in this limited
	# buffer size. This value has also been suggested in DNS Flag Day 2020.
	edns-buffer-size: 1232

	# Perform prefetching of close to expired message cache entries
	# This only applies to domains that have been frequently queried
	prefetch: yes

	# One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
	num-threads: 1

	# Ensure kernel buffer is large enough to not lose messages in traffic spikes
	so-rcvbuf: 1m

	# Ensure privacy of local IP ranges
	private-address: 192.168.0.0/16
	private-address: 169.254.0.0/16
	private-address: 172.16.0.0/12
	private-address: 10.0.0.0/8
	private-address: fd00::/8
	private-address: fe80::/10
```

3. Start unbound and validate it is repsonding to queries
```

  incus exec pihole -- service unbound restart
  incus exec pihole -- service unbound status
  incus exec pihole -- dig pi-hole.net @127.0.0.1 -p 5335
```

4. Test DNSSEC validation 
```
  incus exec pihole -- dig fail01.dnssec.works @127.0.0.1 -p 5335 # will fail/timeout
  incus exec pihole -- dig dnssec.works @127.0.0.1 -p 5335 # will work
```

5. Configure Pi-hole to point to unbound

In the pihole webUI, go to "Upstream DNS Servers" option and specify a custom
DNS address `127.0.0.1#5335` to point to the local unbound instance. Deslect
all other upstream providers. Click 'Save'.

6. That's it. Celebrate.

[1]:
[2]: https://docs.pi-hole.net/guides/dns/unbound/
