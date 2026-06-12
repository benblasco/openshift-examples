# Satellite KubeVirt integration (OpenShift Virtualization)

Manifests for provisioning RHEL VMs on SNO via Red Hat Satellite 6.19 OpenShift Virtualization compute resource.

## Apply (from toolbox `oc`)

```bash
toolbox enter oc
export PATH="$HOME/.local/bin:$PATH"
oc login --server=https://api.sno.openshift.blasco.id.au:6443 \
  --certificate-authority=$HOME/.kube/sno/api-ca.crt

oc apply -f satellite-kubevirt/namespace.yaml
oc apply -f satellite-kubevirt/serviceaccount.yaml
oc apply -f satellite-kubevirt/rbac.yaml
oc apply -f satellite-kubevirt/token-secret.yaml
oc apply -f satellite-kubevirt/nad-satellite-provisioning.yaml
```

## Service account token (for Satellite compute resource)

```bash
oc get secret satellite-provisioner-token -n satellite-vms \
  --template='{{.data.token | base64decode}}'
```

Store locally; paste into Satellite **Infrastructure → Compute Resources → Token**.

## API CA (self-signed cluster)

```bash
echo | openssl s_client -connect api.sno.openshift.blasco.id.au:6443 \
  -servername api.sno.openshift.blasco.id.au 2>/dev/null \
  | openssl x509 -outform PEM > ~/.kube/sno/satellite-sno-api-ca.crt
```

## NAD

Uses OVN-Kubernetes localnet topology via **`br-ex`** on the SNO node (192.168.1.0/24), mapped through the existing OVS bridge-mapping `physnet:br-ex`. VMs get DHCP from EdgeRouter `192.168.1.1` with per-MAC static mappings in `192.168.1.128/25`.

## Satellite provisioning template change

The "SOE Kickstart Default PXELinux" template must include `inst.noverifyssl` in the kernel append line. Satellite uses a self-signed CA (`EXAMPLE.LAB Certificate Authority`) which the Anaconda installer does not trust by default, causing the kickstart fetch to fail over HTTPS.

In **Hosts → Provisioning Templates → SOE Kickstart Default PXELinux**, add `inst.noverifyssl` to the `APPEND` line before `inst.ks.sendmac`.

### SOE Kickstart Default — `url` line (line 110)

The `url` kickstart command rendered from `<%= @mediapath %>` points at the HTTPS kickstart repo on Satellite. While `inst.noverifyssl` on the kernel command line covers the initial kickstart and stage2 (`install.img`) fetches, it does **not** cover Anaconda's dnf repo access for package installation. Without `--noverifyssl` on the `url` line itself, Anaconda fails SSL verification against the self-signed Satellite CA, crashes before installing any packages, and the VM reboots into PXE — creating an infinite install loop.

On **line 110**, change:

```erb
<%= @mediapath %><%= proxy_string %>
```

to:

```erb
<%= @mediapath %> --noverifyssl<%= proxy_string %>
```

### SOE Kickstart Default — `%packages` section (RHEL 10)

The `dhclient` package was removed in RHEL 10 (replaced by NetworkManager's internal DHCP client). If `dhclient` is listed in the `%packages` section, Anaconda raises a `NonCriticalInstallationError` and pops a YesNoDialog. In automated text-mode kickstart, this dialog causes the installation to abort and the VM reboots into PXE -- creating an install loop with zero RPMs ever downloaded.

Conditionalize `dhclient` (and any other RHEL < 10 only packages) in the template:

```erb
<% unless rhel_compatible && os_major >= 10 -%>
dhclient
<% end -%>
```

### SOE Kickstart Default — `%post` built snippet (lines 380, 383)

The `SOE Kickstart Default` template wraps the `snippet('built', ...)` calls in `indent(2, skip1: true) { ... }`. This indents the snippet output, including the `EOF` terminator of a bash heredoc inside the snippet. Bash requires `EOF` at column 0 for `<< EOF` heredocs, so the indented terminator causes `syntax error: unexpected end of file` during the `%post` phase and the installation fails.

Remove the `indent()` wrapper from both `snippet('built', ...)` calls on **lines 380 and 383**. Change:

```erb
<%= indent(2, skip1: true) { snippet('built', :variables => { ... }) } -%>
```

to:

```erb
<%= snippet('built', :variables => { ... }) -%>
```

### KubeVirt VM memory

The RHEL 10 installer requires at least 3-4 GiB RAM for network-based kickstart installations. The `install.img` stage2 (~656 MB) is loaded into RAM alongside the kernel, Anaconda, and dnf. The Satellite compute profile should allocate at least **4 GiB** for provisioning.

## Network routing changes (required for cross-subnet PXE/TFTP)

VMs PXE boot from Satellite at `192.168.5.100` (hosted on hypervisor `hex.lan`). Because OVN localnet VMs aren't directly ARP-reachable from all hosts on the LAN, the following routing workarounds are needed for the TFTP return path.

### EdgeRouter (192.168.1.1) — persistent

DHCP static mapping so the VM always gets the same IP (required before PXE boot):

```bash
configure
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 static-mapping rhel10-kubevirt-test01 ip-address 192.168.1.201
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 static-mapping rhel10-kubevirt-test01 mac-address 02:1a:d8:a6:e2:e8
commit ; save
```

Static ARP so the EdgeRouter can deliver to VMs without ARP (OVN localnet VMs don't respond to ARP from outside the OVS bridge):

```bash
configure
set protocols static arp 192.168.1.201 hwaddr 02:1a:d8:a6:e2:e8
commit ; save
```

To revert:

```bash
configure
delete service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 static-mapping rhel10-kubevirt-test01
delete protocols static arp 192.168.1.201
commit ; save
```

### hex.lan (192.168.1.5 / 192.168.5.254) — non-persistent, lost on reboot

Host route so TFTP responses go via the EdgeRouter (which has the static ARP) instead of hex.lan trying to ARP directly:

```bash
sudo ip route add 192.168.1.201/32 via 192.168.1.1
```

Disable ICMP redirect acceptance (the EdgeRouter sends redirects because src and dst are on the same interface, which causes hex.lan to bypass the EdgeRouter):

```bash
sudo sysctl -w net.ipv4.conf.eno2.accept_redirects=0
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0
sudo sysctl -w net.ipv4.conf.eno2.secure_redirects=0
sudo sysctl -w net.ipv4.conf.all.secure_redirects=0
```

To make persistent, add to `/etc/sysctl.d/99-no-redirects.conf`:

```
net.ipv4.conf.eno2.accept_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.eno2.secure_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
```

And add the route to `/etc/NetworkManager/dispatcher.d/` or a systemd unit.

To revert all hex.lan changes:

```bash
sudo ip route del 192.168.1.201/32 via 192.168.1.1
sudo sysctl -w net.ipv4.conf.eno2.accept_redirects=1
sudo sysctl -w net.ipv4.conf.all.accept_redirects=1
sudo sysctl -w net.ipv4.conf.eno2.secure_redirects=1
sudo sysctl -w net.ipv4.conf.all.secure_redirects=1
```

## Post-install: KubeVirt VM boot order

After installation completes, the VM will still PXE boot first (network `bootOrder: 1`). Satellite deploys a "local boot" PXE config that tries to chainload `hd0`, but this doesn't work reliably with KubeVirt virtio disks. Patch the VM to boot from disk first:

```bash
oc patch vm <vm-name> -n satellite-vms --type json -p '[
  {"op": "add", "path": "/spec/template/spec/domain/devices/disks/0/bootOrder", "value": 1},
  {"op": "replace", "path": "/spec/template/spec/domain/devices/interfaces/0/bootOrder", "value": 2}
]'
```

## Troubleshooting journey

The following issues were encountered and resolved while provisioning `rhel10-kubevirt-test01.example.lab` (RHEL 10.1 on OpenShift 4.21 SNO with Satellite 6.19.1). Listed in the order they were hit.

| # | Symptom | Root cause | Fix |
|---|---------|-----------|-----|
| 1 | `hammer host create` → `Medium can't be blank` | No installation medium for RHEL 10.1 | Created `RHEL 10.1 Kickstart` medium and associated it with the OS |
| 2 | `hammer host create` → `undefined method 'interfaces=' for nil` | Satellite couldn't connect to SNO API (401 — wrong CA cert) | Extracted the correct CA signer cert and updated the compute resource |
| 3 | `hammer host create` → `persistentvolumeclaims is forbidden` | ServiceAccount missing RBAC for PVC creation | Added `edit` ClusterRole binding in `rbac.yaml` |
| 4 | PXE boot failed — `no_dhcp`, stale `inst.ks` URL | Host record had old Satellite hostname | Deleted and recreated the host to regenerate PXE config |
| 5 | NAD failed — `macvlan hairpin problem`, then `failed bridge mapping validation` | macvlan incompatible with OVS `br-ex`; NAD config name wrong | Switched to `ovn-k8s-cni-overlay` localnet with `physicalNetworkName: "physnet"` |
| 6 | VM not getting DHCP (cross-subnet routing) | OVN localnet VMs not ARP-reachable from hex.lan/Satellite | Static ARP on EdgeRouter + host route on hex.lan + disabled ICMP redirects |
| 7 | PXE/TFTP timeout — "connection timed out", "no bootable device" | MTU mismatch: OVN defaulted to 1400, physical network uses 1500 | Set `"mtu": 1500` explicitly in the NAD |
| 8 | Installer fetched but "failed to fetch kickstart file" | Anaconda didn't trust Satellite's self-signed CA over HTTPS | Added `inst.noverifyssl` to PXELinux template kernel params |
| 9 | Install loop — VM PXE reboots, no RPMs ever downloaded | `url` line in kickstart lacked `--noverifyssl`; dnf failed SSL verification for repo access | Added `--noverifyssl` to `url` line (line 110) in kickstart template |
| 10 | Install loop persisted after SSL fix | `dhclient` missing in RHEL 10 → `NonCriticalInstallationError` → silent abort in text-mode | Conditionalized `dhclient` in `%packages` for RHEL < 10 only |
| 11 | `%post` failed — `syntax error: unexpected end of file` | `indent(2, skip1: true)` wrapper indented heredoc `EOF` terminator | Removed `indent()` from `snippet('built')` calls (lines 380, 383) |
| 12 | Installer OOM / sluggish | 2 GiB too small for RHEL 10 network kickstart (stage2 ~656 MB + kernel + Anaconda + dnf) | Increased VM memory to 4 GiB |
| 13 | Post-install stuck at PXE "Booting local disk" menu | Network `bootOrder: 1`, disk had no boot order; `chain.c32 hd0` failed with virtio | Patched VM: disk `bootOrder: 1`, network `bootOrder: 2` |
