## Lab layout

~~~
                     +----------------+
                     |                |
                +----+  br-ex         +--------------+             +-----------------------------+
                |    |                |              |             |     +---------------------+ |
                |    +-+--------------+              |             |     |       loopback      | |
+---------------+--+   |    +------------------------+-----+       |     |    (fc00:124::1)    | |
|      POD         |   |    |                              |       |     |                     | |
|             eth0 +---+    |                              |       |     |                     | |
|                  |        |                              |       |     +---------------------+ |
|                  |        |                              |       |                             |
|                  |        |        br1                   |       |                             |
|             net1 +--------+        (fc00:123::10)        +---------eth0                        |
|   (fc00:123::20) |        |                              |       | (fc00:123::1)               |
|                  |        |                              |       |                             |
|                  |        |                              |       |                             |
+---------------+--+        +------------------------+-----+       |                             |
                |                                    |             |                             |
                +------------------------------------+             +-----------------------------+


                 pod routes:
                 fc00:124::/64 via fc00:123::10

                 node routes:
                 fc00:124::/64 via fc00:123::1
~~~

## Configuration steps

i) Install NMState with:
https://docs.openshift.com/container-platform/4.12/networking/k8s_nmstate/k8s-nmstate-about-the-k8s-nmstate-operator.html

ii) Create nmstate.yaml to configure bridge interface, in this case br1 with eno8403np1 and static networking:
~~~
oc apply -f nmstate.yaml
~~~

iii) Patch the network operator:
~~~
oc patch network.operator cluster --type merge --patch-file additional-network.yaml
~~~

iv) Create a new pod with the net-attach-def:
~~~
oc apply -f deployment.yaml
~~~

## Verification steps

The cluster version for this test is 4.12.1:
~~~
$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.12.1    True        False         2d20h   Cluster version is 4.12.1
~~~

Make sure that the node has a route out of br1 to fc00:124::/64 which will be our off cluster target (see `nmstate.yaml` for the static route configuration):
~~~
sh-4.4# ip -6 r | grep fc00:124
fc00:124::/64 via fc00:123::1 dev br1 proto static metric 150 pref medium
~~~

Create the pod with `deployment.yaml`. After pod creation, we can `oc debug node/<node name>` and then verify the MASQUERADE rule:
~~~
sh-4.4# chroot /host ip6tables-save | grep MASQ
:KUBE-MARK-MASQ - [0:0]
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE --random-fully
-A CNI-3511213dc514e5a96e8c93d3 ! -d ff00::/8 -m comment --comment "name: \"test-nad\" id: \"3945ddfe53a5e42a96e0fe6fc89555506d88110388f57db2d2402de9cdffd25a\"" -j MASQUERADE
~~~

Let's make sure that the default settings are configured and that the br_netfilter module is *not* loaded:
~~~
sh-4.4# chroot /host lsmod | grep br
bridge                282624  0
stp                    16384  1 bridge
llc                    16384  2 bridge,stp
sh-4.4# chroot /host sysctl -a | grep bridge
sysctl: unable to open directory "/proc/sys/fs/binfmt_misc/"
sh-4.4# 
~~~

Now, connect to the pod and list its IPv6 IP configuration:
~~~
[akaris@linux ipMasq]$ oc get pods
NAME                                  READY   STATUS    RESTARTS   AGE
netshoot-deployment-8cb7ccdb6-pms8h   1/1     Running   0          4m14s
[akaris@linux ipMasq]$ oc rsh netshoot-deployment-8cb7ccdb6-pms8h
~ # ip -6 a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0@if118: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 state UP
    inet6 fd01:0:0:1::f/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::858:aff:fe80:f/64 scope link
       valid_lft forever preferred_lft forever
3: net1@if119: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    inet6 fc00:123::20/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::c6:f9ff:fe66:653c/64 scope link
       valid_lft forever preferred_lft forever
~ # ip -6 r
2600:52:7:18::/64 dev net1 proto kernel metric 256 expires 42939sec pref medium
2600:52:7:18::/64 via fc00:123::10 dev net1 metric 1024 pref medium
fc00:123::/64 dev net1 proto kernel metric 256 pref medium
fc00:124::/64 via fc00:123::10 dev net1 metric 1024 pref medium
fd01:0:0:1::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev net1 proto kernel metric 256 pref medium
default via fd01:0:0:1::1 dev eth0 metric 1024 pref medium
~~~

Now, go to external host fc00:123::1 and run a tcpdump there. Then, ping from the pod to fc00:124::1
~~~
~ # ping fc00:124::1
PING fc00:124::1(fc00:124::1) 56 data bytes
64 bytes from fc00:124::1: icmp_seq=1 ttl=63 time=0.566 ms
64 bytes from fc00:124::1: icmp_seq=2 ttl=63 time=0.423 ms
^C
--- fc00:124::1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1031ms
rtt min/avg/max/mdev = 0.423/0.494/0.566/0.071 ms
~ # 
~~~

And the tcpdump shows that traffic is correctly masqueraded:
~~~
# tcpdump -nne -i eth0 -l icmp6 | egrep 'reply|request'
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
09:31:18.894164 30:d0:42:da:54:81 > 52:54:00:bd:6f:be, ethertype IPv6 (0x86dd), length 118: fc00:123::10 > fc00:124::1: ICMP6, echo request, id 7, seq 1, length 64
09:31:18.894187 52:54:00:bd:6f:be > 30:d0:42:da:54:81, ethertype IPv6 (0x86dd), length 118: fc00:124::1 > fc00:123::10: ICMP6, echo reply, id 7, seq 1, length 64
09:31:19.924701 30:d0:42:da:54:81 > 52:54:00:bd:6f:be, ethertype IPv6 (0x86dd), length 118: fc00:123::10 > fc00:124::1: ICMP6, echo request, id 7, seq 2, length 64
09:31:19.924725 52:54:00:bd:6f:be > 30:d0:42:da:54:81, ethertype IPv6 (0x86dd), length 118: fc00:124::1 > fc00:123::10: ICMP6, echo reply, id 7, seq 2, length 64
^C12 packets captured
12 packets received by filter
0 packets dropped by kernel
~~~

One can also do the opposite test. Delete the deployment and wait until the pod is gone. Make sure that the MASQUERADE rule is removed from the node. Then, update the net-attach-def by setting `"ipMasq": false` and by patching the cluster network operator CR. After that, verify the net-attach-def's content (make sure that the setting was propagated). Then, respawn the deployment. Make sure that the MASQUERADE ip6tables rule was *not* written this time. Then, rerun the same test - you will now see the pod IPv6 address as the source of the traffic.
