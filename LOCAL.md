# Local environment

These are instructions for setting up a local test environment with ACM and a Kubernetes cluster managed by install.sh

## OpenShift local (CRC)

Tested with CRC version: 2.12.0+74565a6 (OpenShift version: 4.11.18).

Instructions on how to install OpenShift local are available in [Red Hat customer portal](https://access.redhat.com/documentation/en-us/red_hat_openshift_local/2.13/html/getting_started_guide/installation_gsg).

To satifsfy the PoC requirements in terms of resources it is recommended to allocate 6 cores and 16GB of memory to OpenShift local.
This can be done by using following commands to start it up:

~~~
$ export KUBECONFIG=/tmp/crc.kubeconfig
$ crc start -c 6 -m 16384
~~~

Use the credentials provided by the command to log in the Web Console as an administrator after OpenShift local has finished starting. From the command line:

~~~
$ oc login -u kubeadmin
~~~

## ACM

Tested with ACM 2.6.

ACM can be installed on OpenShift by using OLM. Simply go to Operators > OperatorHub in the Web Console and search for "Advanced Cluster Management". Select it and click "Install".

The next step is to create a pull secret, which will be used for installation of the ACM components on the Kubernetes cluster. The images for these components are indeed pulled from Red Hat registry, which requires authentication.

The definition for the pull secret can be downloaded from [Red Hat website](https://console.redhat.com/openshift/install/pull-secret)

~~~
$ oc create secret generic <secret> -n <namespace> --from-file=.dockerconfigjson=<path-to-pull-secret> --type=kubernetes.io/dockerconfigjson
~~~
*replace `<secret>` by the desired name for the secret, `<namespace>` by the name of the namespace, where the secret will get created and `<path-to-pull-secret>` with the path to the file stored in the previous step.*

To configure ACM a MultiClusterHub resource needs to be created. This can be done again through the Web Console in the ACM operator page. The default parameters can be used. For reducing resource consumption in the context of a PoC availabilityConfig can be set to "Basic".
The pull secret should be set to what has been configured in the previous step.

Finally we need to create a MultiClusterEngine. Using the Web Console a custom resource with defaults get created. To set the availability to "Basic" the following command can be used:

~~~
 $ oc patch MultiClusterEngine multiclusterengine --type merge -p '{"spec":{"availabilityConfig":"Basic"}}'
~~~

## Kubernetes cluster

In this PoC we use [kind](https://kind.sigs.k8s.io/) to create a minimal Kubernetes cluster, not consuming much resources. Refer to ["kind" instructions](https://kind.sigs.k8s.io/docs/user/quick-start/) for setting it up on your machine.

Here is the configuration file used for this PoC:

~~~
$ cat > /tmp/kind-1.cfg << EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # WARNING: It is _strongly_ recommended that you keep this the default
  # (127.0.0.1) for security reasons. However it is possible to change this.
  # 192.168.130.1 is the IP of my CRC network
  apiServerAddress: "192.168.130.1"
  ipFamily: ipv4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    protocol: TCP
  - containerPort: 443
    hostPort: 8443
    protocol: TCP
EOF
~~~

The configuration does the following:
- deviate from the default of binding the loopback IP and use the IP of the gateway of the CRC network instead.
- map the ports used by an ingress gateway to the host.
This is to allow connectivity from OpenShift local. There are multiple ways of discovering this IP: using the `ss` command or using a tool like Cockpit.

For people having systemd and which prefer creating rootless containers, the following command can be used to allow cgroups delegation required by kind:
~~~
$ systemd-run --scope --user -p Delegate=true bash
~~~

The Kubernetes cluster can then get created with:
~~~
$ kind create cluster --name kind-1 --kubeconfig /tmp/kind-1.kubeconfig --config /tmp/kind-1.cfg
~~~

Access to OpenShift local is also required from Kubernetes. Therefore name resolution needs to work from the Kubernetes cluster. Additional hosts can be configured in CoreDNS for the purpose.
~~~
$ cat <<EOF | kubectl --kubeconfig=/tmp/kind-1.kubeconfig replace -f -
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        hosts crc.hosts testing {
           192.168.130.11 api.crc.testing canary-openshift-ingress-canary.apps-crc.testing console-openshift-console.apps-crc.testing default-route-openshift-image-registry.apps-crc.testing downloads-openshift-console.apps-crc.testing oauth-openshift.apps-crc.testing cluster-proxy-anp.apps-crc.testing
        fallthrough
        }
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
EOF
~~~

CoreDNS need to be restared. They can just be deleted for the purpose and get automatically recreated.

The last bit is to open firewalls for the communication between OpenShift local and the kind cluster. Using iptables the following commands can be used. An equivalent would need to be run for firewalld or another firewall API:

~~~
 $ sudo iptables -I INPUT -p tcp --dst 192.168.130.1 --dport 37015 --src 192.168.130.0/24 -j ACCEPT
 $ sudo iptables -I INPUT -p tcp --dst 192.168.130.1 --dport 80 --src 192.168.130.0/24 -j ACCEPT
 $ sudo iptables -I INPUT -p tcp --dst 192.168.130.1 --dport 443 --src 192.168.130.0/24 -j ACCEPT
 ~~~
- *192.168.130.1 is the IP address of the CRC gateway used previously.*
- *37017 is the port used for the API of the kind cluster. This may differ in your installation.*