---
title: "DHCP-less Provisioning with Redfish Virtual Media and Preprovisioning Images"
date: 2026-09-21
draft: false
categories: ["metal3", "baremetal", "ironic", "networking", "virtual media"]
author: Kashif Khan, Adam Rozman
---

Most Metal3 deployments rely on DHCP. It is the default path: connect the
machines to a provisioning network, let Ironic hand out an address over DHCP,
boot the deploy ramdisk over iPXE. But DHCP is not always available. Some data
centers do not run a DHCP server, some environments cannot dedicate an untagged
provisioning VLAN, and some operators prefer static addressing for
predictability and auditability.

Metal3 supports a *DHCP-less* provisioning flow built on two features that work
together: **Redfish virtual media** boot and **preprovisioning network data**
delivered through a **`PreprovisioningImage`**. This post covers why DHCP is
normally needed, how virtual media removes that requirement, what is in the
shared image versus configured per host, and how to configure a `BareMetalHost`
to use all of it, with a step-by-step walkthrough at the end.

## Why DHCP is normally required

To see what we are removing, recall what happens in a standard Metal3
deployment. Provisioning is performed by the *Ironic Python Agent* (IPA), a
small Linux ramdisk that runs in memory on the target host and talks back to
Ironic. Before IPA can do anything, the host has to:

1. Boot the IPA kernel and initramfs, and
1. Obtain an IP address and a route so the agent can reach the Ironic API.

With network boot (PXE/iPXE), both steps depend on DHCP. The firmware broadcasts
a DHCP request to locate the boot server and fetch the iPXE binary over TFTP,
and once IPA is running it again relies on DHCP to configure its network
interface. This is why network boot requires a *provisioning network*: an
isolated L2 segment, usually an untagged VLAN, where Ironic can safely run a
DHCP server.

That model is simple and reliable, but it has constraints:

- You must run a DHCP server on the provisioning network.
- The host must reach Ironic over L2, which limits how the network can be
  segmented.
- NIC bonding, tagged VLANs, and static addressing are difficult or impossible
  during the provisioning phase.

Virtual media removes these constraints.

## How virtual media removes the DHCP requirement

Redfish is an HTTP- and JSON-based protocol for out-of-band hardware management.
One of its capabilities is *virtual media*: attaching an ISO 9660 image to the
server as a virtual CD-ROM device over HTTP(s) and booting from it. Instead of
pulling a kernel over TFTP after a DHCP handshake, the BMC mounts the boot ISO
directly and the machine boots from it.

Because the boot image is delivered *out of band* through the BMC rather than
over the provisioning network, the host no longer needs DHCP to start the deploy
ramdisk. This makes it possible to boot hardware over L3 networks, with no DHCP
at all.

To use virtual media, your hardware needs a BMC that supports the Redfish
virtual media feature, and you select it through the BMC address scheme of the
`BareMetalHost`. The most common formats are:

<!-- markdownlint-disable MD013 -->

| Vendor          | BMC address format                                | Notes                                     |
|-----------------|---------------------------------------------------|-------------------------------------------|
| Generic Redfish | `redfish-virtualmedia://<host>:<port>/<systemID>` | **Must not** be used for Dell machines.   |
| Dell iDRAC      | `idrac-virtualmedia://<host>:<port>/<systemID>`   | Use this variant for Dell hardware.       |
| HPE iLO 5+      | `redfish-virtualmedia://<host>:<port>/<systemID>` | Requires recent iLO firmware.             |

<!-- markdownlint-enable MD013 -->

The `<systemID>` is the Redfish path to the particular server, since one Redfish
endpoint can manage several systems. For Dell machines it usually looks like
`/redfish/v1/Systems/System.Embedded.1`, while other vendors often use
`/redfish/v1/Systems/1`. Check your hardware documentation for the correct
value. You can also pin the carrier protocol with a `+http` or `+https` suffix,
for example `redfish-virtualmedia+https://...`; HTTPS is used by default.

Note that virtual media is an optional Redfish feature, and on some hardware it
requires an additional license. Check the
[supported hardware][supported-hardware] documentation before relying on this
approach.

## Configuring the ramdisk network with preprovisioning network data

Booting the ramdisk without DHCP is only half the problem. Once IPA is running,
it still needs a working network configuration to reach Ironic. In a DHCP-less
setup, you supply that configuration statically.

Metal3 does this with *preprovisioning network data*. This describes the
networking for the deploy ramdisk (addresses, routes, bonds, VLANs and DNS), in
the [OpenStack `network_data.json`][network_data] format. It is the same format
used for the deployed operating system's `networkData`, so if you already write
network data for your instances, it will be familiar.

The data is applied inside the ramdisk by a first-boot tool. The default
community-supported IPA ramdisk uses [Glean][glean], a lightweight alternative
to *cloud-init*. If you build your own ramdisk with *cloud-init* instead, that
works too, as long as the tool can consume the `network_data.json` format from a
configuration drive.

Virtual media and network data connect here: the virtual media ISO that boots
the ramdisk also serves as a configuration drive. The same ISO the BMC mounts to
boot IPA carries the network configuration that IPA reads on start-up. There is
no DHCP and no separate metadata service; the network config travels with the
boot image.

You supply this data as a Kubernetes secret whose key is `networkData`, and
point the `BareMetalHost` at it with the `preprovisioningNetworkDataName` field.
The full commands and manifests are in the
[step-by-step walkthrough](#step-by-step-provisioning-a-host-without-dhcp) below.

Preprovisioning network data is often the same configuration you want for the
deployed operating system. If you set `preprovisioningNetworkDataName` but do not
specify a separate `networkData` secret, Metal3 reuses the preprovisioning secret
for the deployed instance as well, so you only configure the network once.

## The PreprovisioningImage resource

So far we have described the result we want: a boot ISO that carries a per-host
network configuration. Something has to build that image and tell Ironic where
to find it. That is the `PreprovisioningImage` custom resource, and you mostly
do not manage it by hand; it is worth knowing about because it shows up when you
watch a host come up and when you troubleshoot.

When the Bare Metal Operator (BMO) registers a host that uses a virtual media
driver, it automatically creates a `PreprovisioningImage` with the same name and
namespace as the `BareMetalHost`. A controller reconciles it through a pluggable
*image provider*, records the resulting `imageUrl` in the status, and marks the
image `Ready`. In the default Metal3 deployment the provider does not build a
customized image; it returns a preconfigured image URL supplied through
environment variables, the same for every host. The per-host network data is
attached later, by Ironic, when it templates the boot ISO.

The one fact to remember is that this `Ready` condition gates registration: BMO
will not finish registering the host until its `PreprovisioningImage` reports
`Ready`. So if a host is stuck in `registering`, the `PreprovisioningImage`
status is the first place to look (Step 6 and Step 7 below show exactly how).

The image provider is pluggable. The default provider shipped with Metal3
publishes one fixed image URL for every host and leaves the per-host templating
to Ironic. An operator can instead supply a custom provider that builds a unique
image per host, for example baking the network data straight into the ISO.
Metal3 passes the network data through as opaque bytes, so a custom provider can
interpret and embed it however it likes. Either way the `BareMetalHost` API you
write is identical.

Putting the pieces together, the end-to-end flow is:

1. You create a `BareMetalHost` with a virtual media BMC address and a
   `preprovisioningNetworkDataName`.
1. BMO creates a `PreprovisioningImage` referencing that network data secret.
1. The image provider returns (or builds) a ramdisk image URL and it is marked
   `Ready`.
1. BMO tells Ironic to boot the host from that image via Redfish virtual media.
1. The BMC mounts the ISO as a virtual CD-ROM; the host boots IPA with no DHCP.
1. IPA reads the static network configuration Metal3 supplied and brings up its
   interfaces.
1. IPA reaches Ironic over the configured L3 network, and inspection or
   provisioning proceeds as usual.

## What is in the image and what is per host

This is the most common question from users who do not want one identical image
for every host. The short answer: the base ramdisk image is shared across all
hosts, and the per-host settings are supplied separately at boot. Three layers
come together, prepared at different times and with different scope.

The base ramdisk image is shared. The IPA kernel, the initramfs or root
filesystem, the agent itself, and the first-boot tool that applies your
configuration (Glean by default, or cloud-init) are built into the image ahead
of time. In the default flow this base image is the same for every host: the
default image provider publishes one fixed `imageUrl` and hands it to every host
that asks. Anything that must be code running inside the
ramdisk, such as a custom IPA plugin or an extra service, has to be part of this
image. You cannot supply code through network data.

The network configuration is per host and delivered at boot. Your
`network_data.json` is not compiled into the shared base image. When Ironic
boots the host over virtual media, it builds a per-host ISO and attaches your
network data to it as a configuration drive. The first-boot tool inside the
ramdisk reads that config drive on start-up and applies the addresses, routes,
bonds and VLANs.
Every host boots the same base ramdisk, but each one gets its own network
configuration at boot. This is why a hundred hosts can share one image URL and
still each get a unique static IP.

A few other settings are per host in the same way. Ironic's per-host ISO build
also branches on settings it must get right for that specific machine:

- Boot mode. UEFI versus BIOS changes what goes into the ISO, so Ironic builds
  the ISO differently per host based on the host's boot mode.
- Kernel arguments (upcoming). Per-host kernel command-line arguments through a
  `preprovisioningExtraKernelParams` field on the `BareMetalHost` are proposed
  in [BMO PR #2576][ppi-kernel-params] and are not in a released version yet, so
  the field below will be rejected until it ships (targeted for BMO v0.15).
  When it lands, it is useful for a mixed hardware fleet or for troubleshooting
  a single machine:

  ```yaml
  apiVersion: metal3.io/v1alpha1
  kind: BareMetalHost
  metadata:
    name: host-0
    namespace: my-cluster
  spec:
    preprovisioningExtraKernelParams: "console=ttyS0,115200 intel_iommu=on"
    # ... bmc, preprovisioningNetworkDataName, and the rest
  ```

  The proposed behavior depends on the image: with a BMO-generated image the
  arguments come only from the `BareMetalHost`; with an external ISO image the
  ISO's own parameters win and the `BareMetalHost` value is ignored; with an
  external initrd image the two sets are combined.

The rule of thumb: software and behavior are part of the image; the network
configuration and boot mode are per host (with per-host kernel arguments
upcoming). If it is code that has to run, it belongs in the image. If it is
which addresses, VLANs or boot mode a specific server should use, it is supplied
at boot and you do not need a new image for it.

Where the line moves: custom image providers. The split above is the default
provider, where the base image is fixed and Ironic does the per-host ISO
templating. The image provider is pluggable, so an operator can replace the
default one with a provider that builds a per-host image and bakes in whatever
it likes, including the network data itself, publishing a unique `imageUrl` for
each host. In that model more is in the image and less is templated by Ironic,
but the `BareMetalHost` API is identical: you still set
`preprovisioningNetworkDataName` and let the provider decide how to honor it.
Metal3 passes the network data through as opaque bytes, so a custom provider can
interpret and embed it however it wants.

## Step-by-step: provisioning a host without DHCP

This section walks through the process end to end, from an empty namespace to a
`BareMetalHost` that reaches `available` without a DHCP server. Follow it in
order to get a first host working.

### Step 0: Check the prerequisites

Before you start, confirm the following:

- Your server has a BMC that supports Redfish **virtual media**. Generic
  Redfish, Dell iDRAC (9, and 8 with recent firmware) and HPE iLO 5+ are the
  regularly tested options. On some hardware virtual media needs an extra
  license, so verify it in the BMC web UI first.
- The Bare Metal Operator and Ironic are running in your management cluster.
  You can check with `kubectl get pods -n baremetal-operator-system` (the
  namespace may differ in your install).
- The `PreprovisioningImage` integration is enabled in BMO. This is what makes
  BMO build the per-host ramdisk image. It is controlled by a BMO command-line
  flag and is off by default in a plain build, though most real deployments
  (and the Ironic Standalone Operator based installs) turn it on for you. If
  your hosts never get past `registering` and you see no `PreprovisioningImage`
  objects appearing, this flag is the first thing to check.
- You know, on paper, the static network you want the ramdisk to use: the IP
  address, netmask, default gateway and DNS server that let the host reach the
  Ironic API over L3.

Pick a namespace to work in. This guide uses `my-cluster`:

```bash
kubectl create namespace my-cluster
```

### Step 1: Find out the NIC name and MAC address

The `network_data.json` format binds configuration to a specific NIC by its MAC
address, and names the interface for you. If you already know the MAC of the NIC
that is cabled to the network Ironic lives on, you can skip ahead.

If you do not know it yet, the easiest way is to let Metal3 tell you. Enroll the
host once with just the BMC details (no network data), let it inspect, and read
the discovered NICs from the resulting `HardwareData` resource:

```bash
kubectl get hardwaredata -n my-cluster host-0 \
  -o jsonpath='{.spec.hardware.nics}' | jq .
```

```json
[
  {
    "ip": "192.168.111.25",
    "mac": "00:f8:a8:a0:d0:d2",
    "model": "0x1af4 0x0001",
    "name": "enp2s0"
  },
  {
    "mac": "00:f8:a8:a0:d0:d0",
    "model": "0x1af4 0x0001",
    "name": "enp1s0"
  }
]
```

Note the `mac` of the interface you want to configure. You will reference it in
the next step. In a strict DHCP-less environment the first inspection also needs
virtual media plus network data, so this step assumes you can inspect once with
DHCP available, or that you already know the MAC from your cabling records.

### Step 2: Write the network_data.json

Create a file named `host-0-network.json`. Start with the simplest case: one
NIC, a static IPv4 address, a default route and a DNS server. Use the MAC you
found in the previous step.

```json
{
  "links": [
    {
      "id": "enp1s0",
      "type": "phy",
      "ethernet_mac_address": "00:f8:a8:a0:d0:d0"
    }
  ],
  "networks": [
    {
      "id": "network0",
      "link": "enp1s0",
      "type": "ipv4",
      "ip_address": "192.168.1.20",
      "netmask": "255.255.255.0",
      "routes": [
        {
          "network": "0.0.0.0",
          "netmask": "0.0.0.0",
          "gateway": "192.168.1.1"
        }
      ]
    }
  ],
  "services": [
    {
      "type": "dns",
      "address": "8.8.8.8"
    }
  ]
}
```

Three things to keep straight in this format:

- `links` describe physical or virtual devices. Each `link` is matched to real
  hardware through `ethernet_mac_address`, and its `id` is the name the device
  gets inside the ramdisk.
- `networks` describe L3 configuration and attach to a `link` through the `link`
  field. Use `"type": "ipv4"` for a static address (as above) or
  `"type": "ipv4_dhcp"` if that particular interface should still use DHCP.
- `services` are global helpers such as DNS servers.

If your provisioning traffic runs over a **bonded** pair of NICs on a **tagged
VLAN**, a common reason to use this feature, the same file grows into something
like this:

```json
{
  "links": [
    {
      "id": "eth0",
      "type": "phy",
      "ethernet_mac_address": "00:f8:a8:a0:d0:d0"
    },
    {
      "id": "eth1",
      "type": "phy",
      "ethernet_mac_address": "00:f8:a8:a0:d0:d2"
    },
    {
      "id": "bond0",
      "type": "bond",
      "ethernet_mac_address": "00:f8:a8:a0:d0:d0",
      "bond_links": ["eth0", "eth1"],
      "bond_mode": "802.3ad",
      "bond_xmit_hash_policy": "layer3+4"
    },
    {
      "id": "vlan100",
      "type": "vlan",
      "vlan_link": "bond0",
      "vlan_id": 100
    }
  ],
  "networks": [
    {
      "id": "provisioning",
      "link": "vlan100",
      "type": "ipv4",
      "ip_address": "192.168.1.20",
      "netmask": "255.255.255.0",
      "routes": [
        {
          "network": "0.0.0.0",
          "netmask": "0.0.0.0",
          "gateway": "192.168.1.1"
        }
      ]
    }
  ],
  "services": [
    {
      "type": "dns",
      "address": "8.8.8.8"
    }
  ]
}
```

This is exactly the kind of setup that plain PXE plus DHCP cannot handle during
provisioning, because PXE needs an untagged VLAN and supplies its address over
DHCP.

Two things to note about bonds. First, the `bond` link needs an
`ethernet_mac_address` of its own (Glean uses it as the bond's address); set it
to one of the member MACs. Second, the default Glean ramdisk brings the bond up
but does not apply `bond_mode` or `bond_xmit_hash_policy`; those keys are
honored by cloud-init, so include them if you build a cloud-init based ramdisk,
and configure the switch side accordingly for the mode you expect.

### Step 3: Create the network data secret

Turn the file into a secret. The key inside the secret **must** be `networkData`
or Metal3 will not find it:

```bash
kubectl create secret generic host-0-preprov-networkdata \
  -n my-cluster \
  --from-file=networkData=host-0-network.json
```

Verify it was created with the right key:

```bash
kubectl get secret host-0-preprov-networkdata -n my-cluster \
  -o jsonpath='{.data.networkData}' | base64 -d | jq .
```

### Step 4: Create the BMC credentials secret

Each `BareMetalHost` needs a secret holding the BMC username and password.
Create it with `kubectl`:

```bash
kubectl create secret generic host-0-bmc \
  -n my-cluster \
  --from-literal=username='admin' \
  --from-literal=password='<your-bmc-password>'
```

Or as a manifest, if you prefer to keep everything in YAML (use `stringData` so
you do not have to base64-encode by hand):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: host-0-bmc
  namespace: my-cluster
type: Opaque
stringData:
  username: admin
  password: <your-bmc-password>
```

### Step 5: Write and apply the BareMetalHost

The two fields that make this a DHCP-less flow are the virtual media BMC
`address` and `preprovisioningNetworkDataName`:

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: host-0
  namespace: my-cluster
spec:
  online: true
  bootMode: UEFI
  bootMACAddress: 80:c1:6e:7a:e8:10
  bmc:
    address: redfish-virtualmedia://192.168.1.13/redfish/v1/Systems/1
    credentialsName: host-0-bmc
    disableCertificateVerification: true
  preprovisioningNetworkDataName: host-0-preprov-networkdata
```

A few notes on the fields:

- Use `idrac-virtualmedia://` instead of `redfish-virtualmedia://` for Dell
  hardware, and set the correct `<systemID>` for your vendor
  (`/redfish/v1/Systems/System.Embedded.1` on Dell, `/redfish/v1/Systems/1` on
  many others).
- `bootMACAddress` is the MAC used to identify the host during boot. It is not
  necessarily the same NIC you configured in `network_data.json`.
- `disableCertificateVerification: true` is convenient in a lab where the BMC
  uses a self-signed certificate. In production, provision a trusted certificate
  and drop this line.
- Because we set `preprovisioningNetworkDataName` but no separate `networkData`,
  Metal3 will reuse this same secret to configure the deployed operating system
  later. If you want a different network for the final OS, add a `networkData`
  secret reference as well.

Save the two secrets and this manifest into one file (separated by `---`) and
apply it:

```bash
kubectl apply -f host-0.yaml
```

### Step 6: Watch it come up

Watch the host move through its states. With virtual media and preprovisioning
network data enabled, you should see it go through `registering` and
`inspecting` and land on `available`, all without a DHCP server:

```bash
kubectl get bmh -n my-cluster -w
```

```text
NAME     STATE          CONSUMER   ONLINE   ERROR   AGE
host-0   registering               true             20s
host-0   inspecting                true             45s
host-0   available                 true             3m
```

While it is registering, confirm that BMO actually created the
`PreprovisioningImage` for this host. It shares the host's name and namespace:

```bash
kubectl get preprovisioningimage host-0 -n my-cluster
kubectl get preprovisioningimage host-0 -n my-cluster \
  -o jsonpath='{.status}' | jq .
```

```json
{
  "architecture": "x86_64",
  "format": "iso",
  "imageUrl": "http://172.22.0.2/images/ironic-python-agent.iso",
  "networkData": {
    "name": "host-0-preprov-networkdata",
    "version": "12345"
  },
  "conditions": [
    { "type": "Ready", "status": "True" },
    { "type": "Error", "status": "False" }
  ]
}
```

The line to check is the `Ready` condition. BMO pauses registration until this
image reports `Ready`, so if the host is stuck in `registering`, the
`PreprovisioningImage` status is the first place to look.

### Step 7: Troubleshoot if it gets stuck

If the host does not reach `available`, work through these in order. Most
DHCP-less problems fall into one of these categories:

- **No `PreprovisioningImage` appears.** The integration is probably not enabled
  in BMO, or the BMC address is not a virtual media variant (a plain
  `redfish://` or `ipmi://` address will not trigger it). Check the driver in
  the address and the BMO flag.
- **`PreprovisioningImage` exists but is not `Ready`.** Read its `Error`
  condition and the image controller logs. The image build or the fixed URL it
  points at is the problem, not your network data yet.
- **Image is `Ready` but the host never talks to Ironic.** This is usually the
  `network_data.json`. Check that `ethernet_mac_address` matches the NIC cabled
  to the Ironic network, that the gateway is reachable, and that DNS resolves. A
  wrong MAC means the config applies to no interface.
- **BMC errors during registration.** Inspect the host events and status:

  ```bash
  kubectl describe bmh host-0 -n my-cluster
  kubectl get bmh host-0 -n my-cluster -o jsonpath='{.status.errorMessage}'
  ```

  Certificate and licensing errors from the BMC show up here. On a lab BMC with
  a self-signed certificate, make sure `disableCertificateVerification: true` is
  set.

Once the host is `available`, it has inspected and is ready to be provisioned
like any other Metal3 host, with no DHCP involved. Provisioning it with an
operating system image, and driving it through Cluster API, works the same as in
the standard flow.

## Summary

DHCP-less provisioning comes down to two features working together. Redfish
virtual media delivers the boot image out of band through the BMC, so you no
longer need DHCP or an untagged provisioning VLAN to start the deploy ramdisk.
Preprovisioning network data, delivered through a `PreprovisioningImage` and
applied by Glean or cloud-init from the ISO's configuration drive, gives the
ramdisk a static network configuration so it can reach Ironic. The base ramdisk
image is shared across hosts; the network configuration and boot mode are
supplied per host. Set a virtual media BMC address and a
`preprovisioningNetworkDataName` on your `BareMetalHost`, and Metal3 handles the
rest.

This flow fits environments where DHCP is not available: regulated data centers,
L3-segmented fabrics, or setups that require static addressing. If you try it,
share your experience with the community. You can find us on the
[Kubernetes Slack][slack] in the `#cluster-api-baremetal` channel, or join the
weekly community meetings.

For reference documentation, see
[advanced instance customization][advanced-instance-customization] and
[supported hardware][supported-hardware] in the Metal3 user guide.

[network_data]: https://docs.openstack.org/nova/latest/user/metadata.html#openstack-format-metadata
[glean]: https://opendev.org/opendev/glean/
[supported-hardware]: https://book.metal3.io/bmo/supported_hardware
[advanced-instance-customization]: https://book.metal3.io/bmo/advanced_instance_customization
[ppi-kernel-params]: https://github.com/metal3-io/baremetal-operator/pull/2576
[slack]: https://kubernetes.slack.com/archives/CHD49TLE7
