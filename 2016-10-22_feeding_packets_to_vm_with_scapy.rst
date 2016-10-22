====================================
 Feeding packets to a VM with Scapy
====================================

:Date: 2016-10-22
:Authors: Jakub Sitnicki <jkbs@redhat.com>
:Version: 1

Want to send hand crafted packets to a VM? `Scapy`_ can be used for
that. It can sniff on a `TAP`_ device, a virtual link between the host
and the VM, and send packets in reaction to what it sees coming from
the VM.

.. _Scapy: http://www.secdev.org/projects/scapy/doc/
.. _TAP: https://en.wikipedia.org/wiki/TUN/TAP

In this case, I needed to simulate a quirky switch which is notorious
for tagging every packet forwarded to the host with VLAN ID 0.

This was confusing the network stack in the VM and a simple ping test
from the VM to the outside was failing.

To reproduce this scenario, I've extended a bit the `Simplistic ARP
Monitor`_ example from Scapy's excellent documentation.

.. _Simplistic ARP Monitor: http://www.secdev.org/projects/scapy/doc/usage.html#simplistic-arp-monitor

What we want to do is:

1) for every ARP request from the VM, send a VLAN 0 tagged ARP reply,
2) for every ICMP Echo request from the VM, send a VLAN 0 tagged ICMP
   Echo reply,
3) ignore everything else.

My Scapy script to do just that is `here`_. Now all that is left is to
attach the script to a TAP device linking to the VM.

.. _here: https://github.com/jsitnicki/tools/blob/master/net/scapy/arp_and_icmp_reply_with_vlan0.py

I like to use Andrew Lutomirski's `virtme`_ tool to spin up toy VMs
but it's doesn't matter what you use - qemu, virsh,
virt-manager... What is important is the vNIC model you choose. For
example, I had problems with virtio_net v1.0.0 driver
(qemu-system-x86-2.6.2-2.fc24.x86_64), which seems to be filtering
VLAN 0 tagged frames when the device is not in promiscuous mode. A
bug?

.. _virtme: https://github.com/amluto/virtme

Let's start up a VM::

  $ sudo virtme-run \
      --installed-kernel \
      --qemu-opts -net nic,model=e1000 -net tap,script=no,downscript=no
  [    0.000000] Linux version 4.7.7-200.fc24.x86_64 (mockbuild@bkernel01.phx2.fedoraproject.org) (gcc version 6.2.1 20160916 (Red Hat 6.2.1-2) (GCC) ) #1 SMP Sat Oct 8 00:21:59 UTC 2016
  …
  virtme-init: console is ttyS0
  bash-4.3# ip link show
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  2: ens2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
  bash-4.3# ethtool -i ens2
  driver: e1000
  version: 7.3.21-k8-NAPI
  …

The ``e1000`` virtual network device is there. Let's assign it an
address and bring it up::

  bash-4.3# ip address add dev ens2 10.1.1.1/24
  bash-4.3# ip link set dev ens2 up

While on the host a TAP device has showed up (you might find it under
another name, like ``vnetX``). Let's bring it up. We don't need to do
anything else with it for this test, like enslave it to a bridge. ::

  $ ip link show
  ...
  12: tap0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
      link/ether 6a:fb:0b:61:63:94 brd ff:ff:ff:ff:ff:ff
  $ sudo ip link set dev tap0 up

We are ready to carry out the test. Let's put Scapy to work::

  $ sudo ~/tools/net/scapy/arp_and_icmp_reply_with_vlan0.py tap0
  WARNING: No route found for IPv6 destination :: (no default route?)

It will be useful to monitor the traffic exchanged with the VM to
confirm that what is happening is what we expect. On another terminal
run ``tcpdump -n -nn -ei tap0 -t``.

Now let's ping from the VM a fake address on the same subnet as VM
thinks its on::

  bash-4.3# ping -c 3 10.1.1.2
  PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
  64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=20.3 ms
  64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=8.60 ms
  64 bytes from 10.1.1.2: icmp_seq=3 ttl=64 time=7.30 ms

  --- 10.1.1.2 ping statistics ---
  3 packets transmitted, 3 received, 0% packet loss, time 2003ms
  rtt min/avg/max/mdev = 7.305/12.100/20.394/5.889 ms

Scapy script is reporting what it has sniffed on the TAP interface and
that replies were sent. ::

  .
  Sent 1 packets.
  Ether / ARP who has 10.1.1.2 says 10.1.1.1 / Padding
  Ether / Dot1Q / ARP is at 0a:e0:c5:28:0c:a7 says 10.1.1.2
  .
  Sent 1 packets.
  Ether / IP / ICMP 10.1.1.1 > 10.1.1.2 echo-request 0 / Raw
  Ether / Dot1Q / IP / ICMP 10.1.1.2 > 10.1.1.1 echo-reply 0 / Raw
  .
  Sent 1 packets.
  Ether / IP / ICMP 10.1.1.1 > 10.1.1.2 echo-request 0 / Raw
  Ether / Dot1Q / IP / ICMP 10.1.1.2 > 10.1.1.1 echo-reply 0 / Raw
  .
  Sent 1 packets.
  Ether / IP / ICMP 10.1.1.1 > 10.1.1.2 echo-request 0 / Raw
  Ether / Dot1Q / IP / ICMP 10.1.1.2 > 10.1.1.1 echo-reply 0 / Raw

And the traffic capture confirms it::

  52:54:00:12:34:56 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.1.1.2 tell 10.1.1.1, length 46
  0a:e0:c5:28:0c:a7 > 52:54:00:12:34:56, ethertype 802.1Q (0x8100), length 46: vlan 0, p 0, ethertype ARP, Reply 10.1.1.2 is-at 0a:e0:c5:28:0c:a7, length 28
  52:54:00:12:34:56 > 0a:e0:c5:28:0c:a7, ethertype IPv4 (0x0800), length 98: 10.1.1.1 > 10.1.1.2: ICMP echo request, id 246, seq 1, length 64
  0a:e0:c5:28:0c:a7 > 52:54:00:12:34:56, ethertype 802.1Q (0x8100), length 102: vlan 0, p 0, ethertype IPv4, 10.1.1.2 > 10.1.1.1: ICMP echo reply, id 246, seq 1, length 64
  52:54:00:12:34:56 > 0a:e0:c5:28:0c:a7, ethertype IPv4 (0x0800), length 98: 10.1.1.1 > 10.1.1.2: ICMP echo request, id 246, seq 2, length 64
  0a:e0:c5:28:0c:a7 > 52:54:00:12:34:56, ethertype 802.1Q (0x8100), length 102: vlan 0, p 0, ethertype IPv4, 10.1.1.2 > 10.1.1.1: ICMP echo reply, id 246, seq 2, length 64
  52:54:00:12:34:56 > 0a:e0:c5:28:0c:a7, ethertype IPv4 (0x0800), length 98: 10.1.1.1 > 10.1.1.2: ICMP echo request, id 246, seq 3, length 64
  0a:e0:c5:28:0c:a7 > 52:54:00:12:34:56, ethertype 802.1Q (0x8100), length 102: vlan 0, p 0, ethertype IPv4, 10.1.1.2 > 10.1.1.1: ICMP echo reply, id 246, seq 3, length 64
