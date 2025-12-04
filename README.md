# talos cluster

## sops
in `.sops.yaml`
```
creation_rules:
  - age:
    - {key}
```

## secrets
### create secrets
```
talhelper gensecret > talsecret.sops.yaml
```
### encrypt secrets
```
sops -e -i talsecret.sops.yaml
```

## talconfig.yaml
### default
```
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
