## Lab layout

~~~

                              WORKER NODE                                 EXTERNAL TARGET

                     +----------------+
                     |                |
                +----+  br-ex         +--------------+             +-----------------------------+
                |    |                |              |             |     +---------------------+ |
                |    +-+--------------+              |             |     |       loopback      | |
+---------------+--+   |    +------------------------+-----+       |     |    (9.0.0.1)        | |
|      POD         |   |    |                              |       |     |                     | |
|             eth0 +---+    |                              |       |     |                     | |
|                  |        |                              |       |     +---------------------+ |
|                  |        |                              |       |                             |
|                  |        |        br1                   |       |                             |
|             net1 +--------+        (192.168.123.10)      +-------+ eth0                        |
| (192.168.123.20) |        |                              |       | (192.168.123.1)             |
|                  |        |                              |       |                             |
|                  |        |                              |       |                             |
+---------------+--+        +------------------------+-----+       |                             |
                |                                    |             |                             |
                +------------------------------------+             +-----------------------------+


                 pod routes:
                 9.0.0.0/24 via 192.168.123.1

                 node routes:
                 9.0.0.0/24 via 192.168.123.1
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

See master branch.
