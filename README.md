# talos cluster

> [!IMPORTANT]
> **CHANGE ALL INSTANCES OF {} WHERE PROMPTED**

## talos
### arm64
```bash
wget -O metal-arm64.iso https://github.com/siderolabs/talos/releases/download/{version}/metal-arm64.iso # <- EDIT THIS
```
### amd64
```bash
wget -O metal-amd64.iso https://github.com/siderolabs/talos/releases/download/{version}/metal-amd64.iso # <- EDIT THIS
```

## .gitignore
```gitignore
**/.DS_Store
**/.vscode
clusterconfig/
```

## .sops.yaml
> [!IMPORTANT]
> if you dont have an age key
>```bash
>mkdir -p $HOME/.config/sops/age
>age-keygen -o $HOME/.config/sops/age/keys.txt
>```

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
  - hostname: node-0
    controlPlane: false
    nodeAnnotations:
      machine: netcup-v22.....
    patches:
      - |-
        machine:
          kubelet:
            extraConfig:
              registerWithTaints:
                - key: node.kubernetes.io/edge
                  value: "true"
                  effect: NoSchedule
    ipAddress: 192.168.0.10
    installDisk: /dev/vda
    networkInterfaces:
      - interface: ens3
        addresses:
          - 192.168.0.10/22
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.0.1
        dhcp: false
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

### watch
``` bash
watch kubectl get nodes
```
### roles
```bash
kubectl label node <node-name> node-role.kubernetes.io/worker=""
```
```bash
kubectl label node <node-name> node-role.kubernetes.io/edge=""
```

## deployments
### cilium
#### add helm repo
```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```
#### show versions
```bash
helm search repo cilium/cilium --versions | head
```
#### get values
```bash
helm show values cilium/cilium --version {version} > cilium-values.yaml # <- EDIT THIS
```

#### an configuration
```yaml
ipam:
  mode: kubernetes
kubeProxyReplacement: true
securityContext:
  capabilities:
    ciliumAgent:
      - CHOWN
      - KILL
      - NET_ADMIN
      - NET_RAW
      - IPC_LOCK
      - SYS_ADMIN
      - SYS_RESOURCE
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID
    cleanCiliumState:
      - NET_ADMIN
      - SYS_ADMIN
      - SYS_RESOURCE
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
k8sServiceHost: {endpoint-ip} # <- EDIT THIS
k8sServicePort: 6443
gatewayAPI:
  enabled: false
hubble:
  relay:
    enabled: true
  ui:
    enabled: true
hostFirewall:
  enabled: true
```
#### create template
```bash
helm template \
    cilium cilium/cilium \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace cilium-system \
    --values cilium-values.yaml \
    > cilium.yaml
```
#### get kubernetes gateway crds
```bash
wget -O gateway-api-crds.yaml https://github.com/kubernetes-sigs/gateway-api/releases/download/{version}/standard-install.yaml # <- EDIT THIS
```
#### apply gateway crds
```bash
kubectl apply -f gateway-api-crds.yaml
```
#### apply cilium
```bash
kubectl create namespace cilium-system
kubectl label namespace cilium-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply -f cilium.yaml
```
#### watch cilium
```bash
watch kubectl get pods -n cilium-system
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
```bash
watch kubectl get pods -n longhorn-system
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
```bash
watch kubectl get pods -n cnpg-system
```
### cert-manager
```bash
helm template \
    cert-manager oci://quay.io/jetstack/charts/cert-manager \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace cert-manager-system \
    --set crds.enabled=true \
    > cert-manager.yaml
```
```bash
kubectl create namespace cert-manager-system
kubectl apply -f cert-manager.yaml
```
```bash
watch kubectl get pods -n cert-manager-system
```
### envoy gateway
```bash
helm template \
    envoy-gateway oci://docker.io/envoyproxy/gateway-crds-helm \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --set crds.gatewayAPI.enabled=true \
    --set crds.gatewayAPI.channel=standard \
    --set crds.envoyGateway.enabled=true \
    > envoy-gateway-crds.yaml
```
```bash
helm template \
    envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace envoy-gateway-system \
    --skip-crds \
    > envoy-gateway.yaml
```
```bash
kubectl create namespace envoy-gateway-system
kubectl label namespace envoy-gateway-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply --server-side -f envoy-gateway-crds.yaml
kubectl apply --server-side -f envoy-gateway.yaml
```
```bash
watch kubectl get pods -n envoy-gateway-system
```
### haproxy
```bash
helm template \
    haproxy-ingress haproxy-ingress/haproxy-ingress \
    --namespace haproxy-system \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --set controller.kind=DaemonSet \
    --set controller.daemonset.useHostPort=true \
    --set controller.service.type=ClusterIP \
    --set controller.ingressClassResource.enabled=true \
    --set controller.ingressClassResource.default=false \
    --set controller.tolerations[0].key="node-role.kubernetes.io/control-plane" \
    --set controller.tolerations[0].operator="Exists" \
    --set controller.tolerations[0].effect="NoSchedule" \
    > haproxy.yaml
```
```bash
kubectl create namespace haproxy-system
kubectl label namespace haproxy-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply -f haproxy.yaml
```
```bash
watch kubectl get pods -n haproxy-system
```
