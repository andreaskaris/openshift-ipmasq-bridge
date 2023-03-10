## Lab layout

~~~
                                     WORKER NODE                                                                EXTERNAL TARGET

                              +----------------------------+
                              |                            |
                 +------------+       br-ex                +----------+
                 |            |                            |          |
                 |            |                            |          |
                 |            +-------+-------+------------+          |
                 |                    |       |                       |
                 |                    |       |                       |
+------------------------------+      |       |                       |
|                              |      |       |                       |
|                              |      |       |                       |
|                      eth0    +------+       |         +-------------------------------+                   +-------------------+
|                              |                        |             |                 |                   |                   |
|                              |   (192.168.123.0/24)   |           br123               |  (9.0.0.0/24)     |                   |
|   192.168.123.20     net1    +------------------------+     (192.168.123.10/24)       +-------------------+ eth0 9.0.0.1:8000 |   (vrf 123 red)
|(9.0.0.0/24 via 192.168.123.1)|                        | (9.0.0.0/24 ^ia 192.168.123.1)|                   |                   |
|                              |              |         +-------------------------------+                   +-------------------+
|                              |              |                       |
|                              |              |                       |
|                              |              |                       |
+------------------------------+              |                       |         
                 |                            |                       |
                 |                            |                       |
                 |                            |                       |
+------------------------------+              |                       |
|                              |              |                       |
|                              |              |                       |
|                      eth0    +--------------+        +--------------------------------+                   +-------------------+
|                              |                       |              |                 |                   |                   |
|                              |    (192.168.124.0/24) |            br124               |  (9.0.0.0/24)     |                   |
|   192.168.124.20     net1    +-----------------------+     (192.168.124.10/24)        +-------------------+ eth0 9.0.0.1:8000 |   (vrf 124 blue)
|(9.0.0.0/24 via 192.168.124.1)|                       | (9.0.0.0/24 ^ia 192.168.124.1) |                   |                   |
|                              |                       +--------------+-----------------+                   +-------------------+
|                              |                                      |
|                              |                                      |
|                              |                                      |
+------------------------------+                                      |
                 |                                                    |
                 |                                                    |
                 |                                                    |
                 +----------------------------------------------------+
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
version   4.12.0    True        False         9d      Cluster version is 4.12.0
~~~

Check the VRF setup on the worker node:
~~~
$ oc debug node/<node name>
(...)
sh-4.4# ip vrf ls
Name              Table
-----------------------
vrf123             123
vrf124             124
sh-4.4# ip route ls vrf vrf123
9.0.0.0/24 via 192.168.123.1 dev br123 proto static metric 150
192.168.123.0/24 dev br123 proto kernel scope link src 192.168.123.10 metric 427
sh-4.4# ip route ls vrf vrf124
9.0.0.0/24 via 192.168.124.1 dev br124 proto static metric 150
192.168.124.0/24 dev br124 proto kernel scope link src 192.168.124.10 metric 428
sh-4.4# ip a ls dev br123
5042: br123: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master vrf123 state UP group default qlen 1000
    link/ether 30:d0:42:d9:b8:7b brd ff:ff:ff:ff:ff:ff
    inet 192.168.123.10/24 brd 192.168.123.255 scope global noprefixroute br123
       valid_lft forever preferred_lft forever
sh-4.4# ip a ls dev br124
5043: br124: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master vrf124 state UP group default qlen 1000
    link/ether 2e:19:d9:5b:80:fd brd ff:ff:ff:ff:ff:ff
    inet 192.168.124.10/24 brd 192.168.124.255 scope global noprefixroute br124
       valid_lft forever preferred_lft forever
~~~

The deployed pods are:
~~~
$ oc get pods
NAME                                      READY   STATUS    RESTARTS   AGE
netshoot123-deployment-67c55c9f4f-d9mps   1/1     Running   0          4m39s
netshoot124-deployment-7465b55b6f-gs5vp   1/1     Running   0          4m39s
~~~

Check the IP address setup:
~~~
$ oc exec -it netshoot123-deployment-67c55c9f4f-d9mps -- /bin/bash -c "ip a;ip r"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if5090: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default 
    link/ether 0a:58:0a:80:00:b8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.128.0.184/23 brd 10.128.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::858:aff:fe80:b8/64 scope link 
       valid_lft forever preferred_lft forever
3: net1@if5091: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether da:18:44:01:c6:6f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.123.20/24 brd 192.168.123.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::d818:44ff:fe01:c66f/64 scope link 
       valid_lft forever preferred_lft forever
default via 10.128.0.1 dev eth0 
9.0.0.0/24 via 192.168.123.1 dev net1 
10.128.0.0/23 dev eth0 proto kernel scope link src 10.128.0.184 
192.168.123.0/24 dev net1 proto kernel scope link src 192.168.123.20 
~~~

~~~
$ oc exec -it netshoot124-deployment-7465b55b6f-gs5vp -- /bin/bash -c "ip a;ip r"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if5092: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default 
    link/ether 0a:58:0a:80:00:b9 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.128.0.185/23 brd 10.128.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::858:aff:fe80:b9/64 scope link 
       valid_lft forever preferred_lft forever
3: net1@if5093: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 92:64:88:e5:4c:7e brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.124.20/24 brd 192.168.124.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::9064:88ff:fee5:4c7e/64 scope link 
       valid_lft forever preferred_lft forever
default via 10.128.0.1 dev eth0 
9.0.0.0/24 via 192.168.124.1 dev net1 
10.128.0.0/23 dev eth0 proto kernel scope link src 10.128.0.185 
192.168.124.0/24 dev net1 proto kernel scope link src 192.168.124.20
~~~

Now, curl IP address 9.0.0.1 from both pods. In each VRF, the subnet 9.0.0.0/24 is part of a different routing domain.
There are 2 servers with the same IP 9.0.0.1, and we can verify that we reach each of them with:
~~~
$ oc exec -it netshoot123-deployment-67c55c9f4f-d9mps -- /bin/bash -c "curl 9.0.0.1:8000"
vrf 123 red
~~~

~~~
$ oc exec -it netshoot124-deployment-7465b55b6f-gs5vp -- /bin/bash -c "curl 9.0.0.1:8000"
vrf 124 blue
~~~
