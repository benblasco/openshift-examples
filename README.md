# openshift-examples
My own OpenShift learning

# 2026 work
## Local authentication (htpasswd) on Single Node OpenShift

See [README.oauth.md](README.oauth.md) for configuring username/password local users on a fresh SNO cluster via the CLI (Fedora Toolbox workflow). YAML manifests are in the [`oauth/`](oauth/) directory.

## LVM Storage for OpenShift Virtualization on Single Node OpenShift

See [README.lvm-storage.md](README.lvm-storage.md) for configuring a dedicated local NVMe as VM backing storage via the LVM Storage operator on SNO (Fedora Toolbox workflow).


# Old work
## Persistent storage in OpenShift Local (CodeReady Containers)

Based on the great work here:
https://developers.redhat.com/articles/2022/04/20/create-and-manage-local-persistent-volumes-codeready-containers#

See yaml files in the `nfsprovisioner` directory
