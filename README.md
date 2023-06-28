# ACM GitOps OpenShift BareMetal with Sushy

ACM, PolicyGenerator Kustomize Pugin, Helm, Sushy - to deploy a SNO bare metal cluster, simple lifecycle management (add worker node, update the cluster version). Uses a Quay transparent mirror to simulate disconnected installation.

# 🤠 For the impatient

- Install RHACM Operator 2.6/2.7+ into your OpenShift4 cluster
- Label your local hub cluster with `ztp-assisted-service=true` to setup provisioning
- Label your local hub cluster with `ztp-gitops-sno=true` to deploy a sno bm instance
- Label your cluster(s) with `update-cluster-version=true` to update the cluster version
- Bootstrap global ArgoCD for policy and Cluster installs
- Add a worker node to your SNO instance post install

```bash
oc apply -f bootstrap-acm-global-gitops/setup.yaml
```

- Adjust all `sno-gitops-policy/input` templates to suit your environment. I'm using:
  - a quay local mirror
  - sushy to emulate bare metal on libvirt
  - sealed secrets for bmc, pull secrets
  - nmstate for bm host configuration
  - cluster domain - dns setup in libvirt (for install) and externally (for access)

- Install Assisted Service Provisioning config

```bash
oc apply -f applicationsets/assisted-service-appset.yaml
```

- Install SNO Cluster

```bash
oc apply -f applicationsets/sno-cluster-appset.yaml
```

- Add an extra worker node to your SNO cluster (move this file into `clusters/sushi` folder when gitops'ing)

```bash
oc apply -f 10-extra-workers-baremetalhost.yaml
```

- Update Cluster Version

```bash
oc apply -f applicationsets/update-cluster-version-appset.yaml
```

- Post install registry mirrors for marketplace (needs `mirror-by-digest-only = false`)

```bash
butane 99-master-mirror-by-digest-registries.bu -o 99-master-mirror-by-digest-registries.yaml
oc apply -f 99-master-mirror-by-digest-registries.yaml
```

# WIP - Helm & PolicyGenerator

This uses the [PR to implement kustomizeOptions](https://github.com/open-cluster-management-io/policy-generator-plugin/pull/109) - we don't use helm yet in this repo, but it is enabled.

```yaml
policies:
  - name: team-gitops
    manifests:
      - path: helm-input/
        kustomizeOptions:
          enableAlphaPlugins: true
          enableHelm: true
```

# ZTP SNO Instance

## Sushy

BMC emulator integrated into libvirt.

```bash
# sushy on host running vms

sudo dnf install bind-utils libguestfs-tools cloud-init -yy
sudo dnf module install virt -yy
sudo dnf install virt-install -yy
sudo systemctl enable libvirtd --now
sudo dnf install podman -yy

sudo mkdir -p /etc/sushy/
cat << "EOF" | sudo tee /etc/sushy/sushy-emulator.conf
SUSHY_EMULATOR_LISTEN_IP = u'0.0.0.0'
SUSHY_EMULATOR_LISTEN_PORT = 8000
SUSHY_EMULATOR_SSL_CERT = None
SUSHY_EMULATOR_SSL_KEY = None
SUSHY_EMULATOR_OS_CLOUD = None
SUSHY_EMULATOR_LIBVIRT_URI = u'qemu:///system'
SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = True
SUSHY_EMULATOR_BOOT_LOADER_MAP = {
    u'UEFI': {
        u'x86_64': u'/usr/share/OVMF/OVMF_CODE.secboot.fd'
    },
    u'Legacy': {
        u'x86_64': None
    }
}
EOF

export SUSHY_TOOLS_IMAGE=${SUSHY_TOOLS_IMAGE:-"quay.io/metal3-io/sushy-tools"}
sudo podman create --net host --privileged --name sushy-emulator -v "/etc/sushy":/etc/sushy -v "/var/run/libvirt":/var/run/libvirt "${SUSHY_TOOLS_IMAGE}" sushy-emulator -i :: -p 8000 --config /etc/sushy/sushy-emulator.conf
sudo podman start sushy-emulator

# firewall
firewall-cmd --add-port=8000/tcp --permanent
firewall-cmd --reload

# or iptables
iptables -I INPUT -p tcp -m state --state NEW -m tcp --dport 8000 -j ACCEPT
iptables-save > /etc/sysconfig/iptables

# test
REDFISH_HOST="192.168.86.27"
REDFISH_PORT="8000"

curl -s http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/ | jq -r
curl http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Managers
curl -s http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/ | jq -r

REDFISH_SYSTEM="8ed24a85-d093-4cd8-a71f-df772d4f132a"

curl -s http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Systems/$REDFISH_SYSTEM/ | jq '[{hostname: .Name, manufacturer: .Manufacturer}, {"hardware": {cpu: .ProcessorSummary.Count, memory: .MemorySummary.TotalSystemMemoryGiB}}, {"health": {system: .Status.Health, cpu: .ProcessorSummary.Status.Health, memory: .MemorySummary.Status.Health}}]'

curl -s http://$REDFISH_HOST:$REDFISH_PORT/redfish/v1/Managers/$REDFISH_MANAGER/VirtualMedia/Cd/ | jq  '[{iso_connected: .Inserted}]'
```

## Define a libvirt vm for sno bm

BM SNO instance `sushi`

```bash
VM_NAME=sushi
NET_NAME=baz
OS_VARIANT="fedora-coreos-stable"
RAM_MB="16384"
DISK_GB="20"
CPU_CORE="8"
MAC=52:54:00:22:4d:4b

lvremove -f fedora/${VM_NAME}
lvcreate --virtualsize 120G --name ${VM_NAME} -T fedora/thin_pool
vgchange -ay -K fedora

tmpfile=$(mktemp /tmp/sushy-domain.XXXXXX)

virt-install \
    -n "${VM_NAME}" \
    -r "${RAM_MB}" \
    --vcpus "${CPU_CORE}" \
    --os-type linux \
    --os-variant="${OS_VARIANT}" \
    --cpu=host-passthrough,cache.mode=passthrough \
    --network=network:${NET_NAME},mac=${MAC},driver.queues=4 \
    --events on_reboot=restart \
    --disk path=/dev/fedora/${VM_NAME},io=io_uring,cache='writeback',discard='unmap' \
    --print-xml > $tmpfile

virsh define --file $tmpfile
rm $tmpfile
```

## Define a libvirt vm for sno bm worker node

Worker node we add to `sushi` SNO cluster. Use the same virt-install code as above but with these env.vars:

```bash
VM_NAME=sushi-worker
NET_NAME=baz
OS_VARIANT="fedora-coreos-stable"
RAM_MB="8192"
DISK_GB="20"
CPU_CORE="4"
MAC=52:54:00:22:4d:4c
```

## Networking

Using libvirt networking in a lab. Here's the setup.

- `baz` - this is a ACM hub instance (also a SNO)
- `sushi` - this is the BM SNO gitops deployed instance
- `sushi-worker` - this is the BM worker instance added to sushi SNO post cluster install

```bash
cat <<EOF > /etc/libvirt/qemu/networks/baz.xml
<network>
  <name>baz</name>
  <uuid>fc23191f-de21-4bf5-774b-98711b9f3d9e</uuid>
  <forward mode='nat' size='1500'/>
  <bridge name='tt0' stp='on' delay='0'/>
  <mac address='52:54:00:22:4d:3a'/>
  <domain name='eformat.me'/>
  <dns enable='yes'>
    <host ip='192.168.130.10'>
      <hostname>api.baz.eformat.me</hostname>
      <hostname>api-int.baz.eformat.me</hostname>
      <hostname>console-openshift-console.apps.baz.eformat.me</hostname>
      <hostname>oauth-openshift.apps.baz.eformat.me</hostname>
      <hostname>canary-openshift-ingress-canary.apps.baz.eformat.me</hostname>
    </host>
    <host ip='192.168.130.11'>
      <hostname>api.sushi.baz.eformat.me</hostname>
      <hostname>api-int.sushi.baz.eformat.me</hostname>
      <hostname>console-openshift-console.apps.sushi.baz.eformat.me</hostname>
      <hostname>oauth-openshift.apps.sushi.baz.eformat.me</hostname>
      <hostname>canary-openshift-ingress-canary.apps.sushi.baz.eformat.me</hostname>
    </host>
  </dns>
  <ip family='ipv4' address='192.168.130.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.130.20' end='192.168.130.254'/>
      <host mac='52:54:00:22:4d:4a' name='baz' ip='192.168.130.10'/>
      <host mac='52:54:00:22:4d:4b' name='sushi' ip='192.168.130.11'/>
      <host mac='52:54:00:22:4d:4c' name='sushi-worker' ip='192.168.130.12'/>
    </dhcp>
  </ip>
</network>
EOF

virsh net-define /etc/libvirt/qemu/networks/baz.xml
virsh net-start baz
virsh net-autostart baz
```

# VMC

VMware gitops cluster deployment.

Kustomize

```bash
kustomize build --enable-helm --enable-alpha-plugins gitops-vmc-policy/
```

Kustomize + Helm

```bash
kustomize build --enable-helm --enable-alpha-plugins gitops-vmc-policy-helm/
```

AppSet for Kustomize + Helm

```bash
oc apply -f applicationsets/cluster-appset-vmc.yaml
```

Sealed Secrets for VMC. Secret templates are in `secrets/` folder, adjust to suit.

```bash
kubeseal < secrets/01-vsphere-creds-secret.yaml > gitops-vmc-policy/input/clusters/mx76p/01-vsphere-creds-sealed-secret.yaml \
    -n cluster-mx76p \
    --controller-namespace sealed-secrets \
    --controller-name sealed-secrets \
    -o yaml

kubeseal < secrets/01-cluster-install-secret.yaml > gitops-vmc-policy/input/clusters/mx76p/01-cluster-install-sealed-secret.yaml \
    -n cluster-mx76p \
    --controller-namespace sealed-secrets \
    --controller-name sealed-secrets \
    -o yaml

kubeseal < secrets/01-cluster-pull-secret.yaml > gitops-vmc-policy/input/clusters/mx76p/01-cluster-pull-sealed-secret.yaml \
    -n cluster-mx76p \
    --controller-namespace sealed-secrets \
    --controller-name sealed-secrets \
    -o yaml

kubeseal < secrets/01-cluster-ssh-private-key-secret.yaml > gitops-vmc-policy/input/clusters/mx76p/01-cluster-ssh-private-key-sealed-secret.yaml \
    -n cluster-mx76p \
    --controller-namespace sealed-secrets \
    --controller-name sealed-secrets \
    -o yaml

kubeseal < secrets/01-cluster-vsphere-certs-secret.yaml > gitops-vmc-policy/input/clusters/mx76p/01-cluster-vsphere-certs-sealed-secret.yaml \
    -n cluster-mx76p \
    --controller-namespace sealed-secrets \
    --controller-name sealed-secrets \
    -o yaml
```

## Notes

Useful links and workarounds for mirror registry and disconnected installs. I am using a pull-secret for the quay transparent mirror only in this example.

- https://www.youtube.com/watch?v=oVlRDuCD6ic - Quay Transparent Proxy-Pull Through Cache - `FEATURE_PROXY_CACHE`
- https://github.com/openshift/assisted-service/blob/master/docs/operator.md#mirror-registry-configuration
- https://issues.redhat.com/browse/MGMT-8789 - `unqualified-search-registries` needed for install
