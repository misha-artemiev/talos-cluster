# talos cluster

## talos
### arm64
```bash
wget -O metal-arm64.iso https://github.com/siderolabs/talos/releases/download/{version}/metal-arm64.iso # <- EDIT THIS
```
### amd64
```bash
wget -O metal-amd64.iso https://github.com/siderolabs/talos/releases/download/{version}/metal-amd64.iso # <- EDIT THIS
```

## .sops.yaml
### generate age if doesnt exist
```bash
age-keygen -o $HOME/.config/sops/age/keys.txt
```
### generate
```yaml
creation_rules:
  - age:
    - {key} # <- EDIT THIS
```

## talsecret.sops.yaml
### generate
```bash
talhelper gensecret > talsecret.sops.yaml
```
### encrypt
```bash
sops -e -i talsecret.sops.yaml
```

## talconfig.yaml
```yaml
clusterName: {cluster-name} # <- EDIT THIS
talosVersion: {version} # <- EDIT THIS
kubernetesVersion: {version} # <- EDIT THIS
endpoint: https://{endpoint-address}:6443 # <- EDIT THIS
domain: cluster.local
allowSchedulingOnControlPlanes: false
clusterPodNets:
  - 10.244.0.0/16
clusterSvcNets:
  - 10.96.0.0/12
cniConfig:
  name: none

patches:
  - |-
    cluster:
      proxy:
        disabled: true
  - |-
    machine:
      features:
        hostDNS:
          enabled: true
          forwardKubeDNSToHost: false

commonConfig: &common
  volumes:
    - name: EPHEMERAL
      encryption:
        provider: luks2
        keys:
          - slot: 0
            nodeID: {}
    - name: STATE
      encryption:
        provider: luks2
        keys:
          - slot: 0
            nodeID: {}
  machineSpec:
    mode: metal
    secureboot: false
  schematic:
    customization:
      systemExtensions:
        officialExtensions:
          - siderolabs/iscsi-tools
          - siderolabs/util-linux-tools
          - siderolabs/qemu-guest-agent
  patches:
    - |-
      machine:
        sysctls:
          vm.nr_hugepages: "1024"
        kernel:
          modules:
            - name: nvme_tcp
            - name: vfio_pci
        kubelet:
          extraMounts:
            - destination: /var/lib/longhorn
              type: bind
              source: /var/lib/longhorn
              options:
                - bind
                - rshared
                - rw

controlPlane:
  <<: *common
worker:
  <<: *common

nodes:
  - hostname: {node-name} # <- EDIT THIS
    ipAddress: {node-ip} # <- EDIT THIS
    installDisk: /dev/{node-install-disk} # <- EDIT THIS
    controlPlane: {is-control-panel}
    networkInterfaces:
      - interface: {network-interface-name} # <- EDIT THIS
        addresses:
          - {node-ip}/{node-ip-cidr} # <- EDIT THIS
        routes:
          - network: 0.0.0.0/0
            gateway: {node-ip-gateway} # <- EDIT THIS
        dhcp: false
  - ... # <- ADD MORE
```

## talhelper
### generate config
```bash
talhelper genconfig
```
### apply config
```bash
talhelper gencommand apply --extra-flags --insecure
```
### bootstrap etcd
```bash
talosctl bootstrap --talosconfig=clusterconfig/talosconfig --nodes {endpoint-address} # <- EDIT THIS
```
### local kubeconfig
```bash
talosctl kubeconfig --talosconfig=clusterconfig/talosconfig --nodes {endpoint-address} # <- EDIT THIS
```

## deployments
### cilium
```bash
helm template \
    cilium cilium/cilium \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace cilium-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost={endpoint-ip} \ # <- EDIT THIS
    --set k8sServicePort=6443 \
    --set=gatewayAPI.enabled=true \
    --set=gatewayAPI.enableAlpn=true \
    --set=gatewayAPI.enableAppProtocol=true \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set hostFirewall.enabled=true \
    > cilium.yaml
```
```bash
kubectl create namespace cilium-system
kubectl label namespace cilium-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply -f cilium.yaml
```
### longhorn
```bash
helm template \
    longhorn longhorn/longhorn \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace longhorn-system \
    > longhorn.yaml
```
```bash
kubectl create namespace longhorn-system
kubectl label namespace longhorn-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply -f longhorn.yaml
```
### cnpg
```bash
helm template \
    cnpg cnpg/cloudnative-pg \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace cnpg-system \
    > cnpg.yaml
```
```bash
kubectl create namespace cnpg-system
kubectl apply --server-side -f cnpg.yaml
```
### envoy gateway
```bash
helm template \
    envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace envoy-gateway-system \
    > envoy-gateway.yaml
```
```bash
kubectl create namespace envoy-gateway-system
kubectl label namespace envoy-gateway-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply -f envoy-gateway.yaml
```
