# talos cluster

## .sops.yaml
### generate age if doesnt exist
```
age-keygen -o $HOME/.config/sops/age/keys.txt
```
### generate
```
creation_rules:
  - age:
    - {key}
```

## talsecret.sops.yaml
### generate
```
talhelper gensecret > talsecret.sops.yaml
```
### encrypt
```
sops -e -i talsecret.sops.yaml
```

## talconfig.yaml
### default
```bash
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
```
### additional
```
clusterName: {cluster-name}
talosVersion: v{version}
kubernetesVersion: v{version}
endpoint: https://{endpoint-address}:6443
nodes:
  - hostname: {node-name}
    ipAddress: {node-ip}
    installDisk: /dev/{node-install-disk}
    controlPlane: {is-control-panel}
    networkInterfaces:
      - interface: {network-interface-name}
        addresses:
          - {node-ip}/{node-ip-cidr}
        routes:
          - network: 0.0.0.0/0
            gateway: {node-ip-gateway}
        dhcp: false
```

## talhelper
### generate config
```
talhelper genconfig
```
### apply config
```
talhelper gencommand apply --extra-flags --insecure
```
### bootstrap etcd
```
talosctl bootstrap --talosconfig=clusterconfig/talosconfig --nodes {endpoint-address}
```
### local kubeconfig
```
talosctl kubeconfig --talosconfig=clusterconfig/talosconfig --nodes {endpoint-address}
```

## deployments
### cilium
```
helm template \
    cilium cilium/cilium \
    --kube-version {version} \
    --version {version} \
    --namespace cilium-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost={endpoint-ip} \
    --set k8sServicePort=6443 \
    --set=gatewayAPI.enabled=true \
    --set=gatewayAPI.enableAlpn=true \
    --set=gatewayAPI.enableAppProtocol=true \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set hostFirewall.enabled=true \
    > cilium-$VERSION.yaml
```
```
kubectl create namespace cilium-system
kubectl label namespace cilium-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply -f cilium-{version}.yaml
```
### longhorn
```
helm template \
    longhorn longhorn/longhorn \
    --kube-version {version} \
    --version {version} \
    --namespace longhorn-system \
    > longhorn-{version}.yaml
```
```
kubectl create namespace longhorn-system
kubectl label namespace longhorn-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply -f longhorn-{version}.yaml
```
### cnpg
```
helm template \
    cnpg cnpg/cloudnative-pg \
    --kube-version {version} \
    --version {version} \
    --namespace cnpg-system \
    > cnpg-{version}.yaml
```
```
kubectl create namespace cnpg-system
kubectl apply --server-side -f cnpg-{version}.yaml
```
