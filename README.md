# ACM GitOps - Helm Version

ACM, PolicyGenerator Kustomize Pugin, Helm

# 🤠 For the impatient

- Install RHACM Operator 2.6/2.7+ into your OpenShift4 cluster
- Label you local cluster with `useglobal=true` and `teamargo=true`
- Boostrap global ArgoCD for policy

```bash
oc apply -f bootstrap-acm-global-gitops/setup.yaml
```

- Adjust all `sno-gitops-policy/input` templates to suit your environment. I'm using:
  - a quay local mirror
  - sushy to emulate bare metal on libvirt
  - sealed secrets for bmc, pull secrets
  - nmstate for bm host configuration
  - cluster domain - dns setup externally

- Install Team based ArgoCD's

```bash
oc apply -f applicationsets/sno-cluster-appset.yaml
```

# WIP

 This uses the [PR to implement kustomizeOptions](https://github.com/open-cluster-management-io/policy-generator-plugin/pull/109)

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
    --disk target=sda,device=cdrom \
    --print-xml > $tmpfile

virsh define --file $tmpfile
rm $tmpfile
```
