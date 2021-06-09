# OpenShift Baremetal UPI on vSphere (PoC)

## Background

If you install OpenShift with `platform: vsphere` or `platform: baremetal` in your `install-config.yaml` the cluster can run without an Loadbalancer and without DNS entries (for `api-int.*` and the nodes). Instead OpenShift runs Keepalived, MDNS and HAProxy (for the API) internally.

Because of a limitation in the installer this is currently not possible when using `platform: none`.

 - https://github.com/openshift/installer/blob/master/pkg/types/none/platform.go
 - https://github.com/openshift/installer/blob/master/pkg/types/vsphere/platform.go
 - https://github.com/openshift/installer/blob/master/pkg/types/baremetal/platform.go

## What is this?

I wan't to run OpenShift (or OKD) on VMs (vSphere) without a cloud provider but avoid an external Loadbalancer and DNS entries. So i tried to install the cluster using `platform: baremetal` with a UPI method.

This is the documentation of the required steps for this setup.

## Installation

### Environment

### install-config.yaml

The `replicas` field for the master and compute nodes do no matter, because we setup the nodes by our own.

Also we can omit `bootstrapProvisioningIP` parameter, because we will setup the nodes by our own (and not via the provisioning node method). 

The `apiVIP` is the floating ip address for the API, the `ingressVIP` is the floating ip address for the Ingress.

The `provisioningHostIP` is the floating ip address for the DHCP/TFTP service in the cluster. With the option `provisioningNetwork: Disabled` we disable the DHCP/TFTP service. Check [this](https://github.com/openshift/installer/blob/master/pkg/types/baremetal/platform.go#L54).

We also omit the `bmc` option (check [this](https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal_ipi/ipi-install-installation-workflow.html#ipi-install-bmc-addressing_ipi-install-configuration-files) for the nodes, because we setup everything on VMs (w/o IPMI/BMC/iDRAC).

```
---
apiVersion: v1
baseDomain: 'example.com'
metadata:
  name: 'cluster1'
compute:
  - architecture: amd64
    hyperthreading: Enabled
    name: worker
    platform: {}
    replicas: 3
controlPlane:
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
networking:
  networkType: OpenShiftSDN
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.1.0/24
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    apiVIP: 192.168.1.21
    ingressVIP: 192.168.1.22
    provisioningHostIP: 192.168.1.23
    provisioningNetwork: Disabled
    #bootstrapProvisioningIP: unused
    hosts:
      - name: master01.cluster1.example.com
        role: master
        bootMACAddress: 93:d2:cb:38:e9:ea
        hardwareProfile: default
      - name: master02.cluster1.example.com
        role: master
        bootMACAddress: 35:26:a9:b2:95:61
        hardwareProfile: default
      - name: master03.cluster1.example.com
        role: master
        bootMACAddress: 1f:42:30:e4:a8:0e
        hardwareProfile: default
      - name: worker01.cluster1.example.com
        role: worker
        bootMACAddress: d2:f0:df:6b:ef:0f
        hardwareProfile: unknown
      - name: worker02.cluster1.example.com
        role: worker
        bootMACAddress: 70:d2:07:9d:ea:88
        hardwareProfile: unknown
      - name: worker03.cluster1.example.com
        role: worker
        bootMACAddress: 00:0e:38:8d:9f:29
        hardwareProfile: unknown
pullSecret: '{"auths":{"quay.io":{"auth":"...","email":"..."},"registry.connect.redhat.com":{"auth":"...","email":"..."},"registry.redhat.io":{"auth":"...","email":"..."}}}'
sshKey: |
  ssh-rsa AAAA.... Peter_Fuerst_2021
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
```

Save this file to ./cluster1/install-config.yaml (and make a backup, because `openshift-install` will delete the file after the next step).

### Generate manifests

```
openshift-install create manifests --dir ./cluster1 --log-level=debug
```

### Delete Machine objects

Because we setup all machines via UPI we delete all Machine and BareMetalHost objects.

```
rm -f ./cluster1/openshift/99_openshift-cluster-api_hosts-*.yaml
rm -f ./cluster1/openshift/99_openshift-cluster-api_master-machines-*.yaml
rm -f ./cluster1/openshift/99_openshift-cluster-api_worker-machineset-*.yaml
```

If necessary we can create them again later.

### Disable scheduling on master nodes

Maybe you also wan't to disable the scheduling on the master nodes (only needed when you defined `replicas: 0` for worker pool in the `install-config.yaml`).

```
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/'  ./cluster1/manifests/cluster-scheduler-02-config.yml
```

## Create ignition files

```
openshift-install create ignition-configs --dir ./cluster1 --log-level=debug
```

## DNS

Each machine need to be able to resolve his own desired hostname via forward (A) and reverse (PTR) DNS records.

Example DNS zone for the cluster (dnsmasq config).

```
# api, ingress
address=/apps.api.cluster1.example.com/192.168.1.22
address=/.apps.api.cluster1.example.com/192.168.1.22
address=/api.cluster1.example.com/192.168.1.21

# nodes - a records
address=/bootstrap.cluster1.example.com/192.168.1.30
address=/master01.cluster1.example.com/192.168.1.31
address=/master02.cluster1.example.com/192.168.1.32
address=/master03.cluster1.example.com/192.168.1.33
address=/worker01.cluster1.example.com/192.168.1.41
address=/worker02.cluster1.example.com/192.168.1.42
address=/worker03.cluster1.example.com/192.168.1.43

# nodes - ptr records
ptr-record=30.1.168.192.in-addr.arpa,bootstrap.cluster1.example.com
ptr-record=31.1.168.192.in-addr.arpa,master01.cluster1.example.com
ptr-record=32.1.168.192.in-addr.arpa,master02.cluster1.example.com
ptr-record=33.1.168.192.in-addr.arpa,master03.cluster1.example.com
ptr-record=41.1.168.192.in-addr.arpa,worker01.cluster1.example.com
ptr-record=42.1.168.192.in-addr.arpa,worker02.cluster1.example.com
ptr-record=43.1.168.192.in-addr.arpa,worker03.cluster1.example.com
```

## Setup machines

I used ansible and the [RHCOS OVA](https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.6/4.6.8/rhcos-vmware.x86_64.ova).

Because i wan't static IPs instead of DHCP i used the `guestinfo.afterburn.initrd.network-kargs` 

I've set the following VM attributes:

  - `guestinfo.afterburn.initrd.network-kargs` to `ip=192.168.1.30::192.168.1.1:255.255.255.0:::none nameserver=192.168.1.1 nameserver=192.168.1.2`
 - `guestinfo.ignition.config.data` to the ignition data (base64 encoded)
 - `guestinfo.ignition.config.data.encoding` to `base64`
 - `disk.EnableUUID` to `TRUE`
```

