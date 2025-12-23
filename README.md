# talos cluster

## talos iso
### set talos version
```
export TALOS_VERSION=
```
### get arm64
```bash
wget -O "metal-arm64-${TALOS_VERSION//./-}.iso" \
  "https://github.com/siderolabs/talos/releases/download/$TALOS_VERSION/metal-arm64.iso"
```
### get amd64
```bash
wget -O "metal-amd64-${TALOS_VERSION//./-}.iso" \
  "https://github.com/siderolabs/talos/releases/download/$TALOS_VERSION/metal-amd64.iso"
```

## .gitignore
```gitignore
**/.DS_Store
**/.vscode
clusterconfig/
```
>[!TIP]
> just dump
>```bash
>cat >> .gitignore <<EOF
>**/.DS_Store
>**/.vscode
>clusterconfig/
>EOF
> ```

## encryption setup
> [!IMPORTANT]
> if you dont have an age key
>```bash
>mkdir -p $HOME/.config/sops/age
>age-keygen -o $HOME/.config/sops/age/keys.txt
>```
### get key id
```bash
export TALSECRETS_KEY=$(age-keygen -y ~/.config/sops/age/keys.txt)
```
### create .sops.yaml
```bash
cat >> .sops.yaml <<EOF
creation_rules:
  - age:
    - $TALSECRETS_KEY
EOF
```

## talsecret.sops.yaml
### generate and encrypt
```bash
talhelper gensecret > talsecret.sops.yaml
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
          extraArgs:
            rotate-server-certificates: true
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
## install
### talhelper
#### generate config
```bash
talhelper genconfig
```
#### apply config
```bash
talhelper gencommand apply --extra-flags --insecure
```
#### bootstrap etcd
```bash
talosctl bootstrap --talosconfig=clusterconfig/talosconfig --nodes {endpoint-address} # <- EDIT THIS
```
#### local kubeconfig
```bash
talosctl kubeconfig --talosconfig=clusterconfig/talosconfig --nodes {endpoint-address} # <- EDIT THIS
```
#### watch
``` bash
watch kubectl get nodes
```
#### roles
```bash
kubectl label node <node-name> node-role.kubernetes.io/worker=""
```
```bash
kubectl label node <node-name> node-role.kubernetes.io/edge=""
```
```bash
kubectl label node <node-name> node-role.kubernetes.io/control-plane=""
```
#### get additional serives
```bash
wget -O gateway-api-crds-standard.yaml https://github.com/kubernetes-sigs/gateway-api/releases/download/{version}/standard-install.yaml <- EDIT THIS
wget -O gateway-api-crds-experimental.yaml https://github.com/kubernetes-sigs/gateway-api/releases/download/{version}/experimental-install.yaml <- EDIT THIS
wget -O cert-approver.yaml https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
wget -O metrics-server.yaml https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
#### apply additional serives
```bash
kubectl apply --server-side -f gateway-api-crds-standard.yaml
kubectl apply --server-side -f gateway-api-crds-experimental.yaml
kubectl apply -f cert-approver.yaml
kubectl apply -f metrics-server.yaml
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
#### apply cilium and gateway crds
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
#### add helm repo
```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```
#### show versions
```bash
helm search repo longhorn/longhorn --versions | head
```
#### get values
```bash
helm show values longhorn/longhorn --version {version} > longhorn-values.yaml # <- EDIT THIS
```
#### an configuration
```yaml
persistence:
  defaultClass: false
global:
  tolerations:
    - key: node.kubernetes.io/edge
      operator: "Exists"
      effect:
```
#### create template
```bash
helm template \
    longhorn longhorn/longhorn \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace longhorn-system \
    --values longhorn-values.yaml \
    > longhorn.yaml
```
#### apply longhorn
```bash
kubectl create namespace longhorn-system
kubectl label namespace longhorn-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply -f longhorn.yaml
```
#### watch longhorn
```bash
watch kubectl get pods -n longhorn-system
```
#### port forward
```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```
### cnpg
#### add helm repo
```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
```
#### show versions
```bash
helm search repo cnpg/cloudnative-pg --versions | head
```
#### get values
```bash
helm show values cnpg/cloudnative-pg --version {version} > cnpg-values.yaml # <- EDIT THIS
```
#### an configuration
```yaml
replicaCount: 3
```
#### create template
```bash
helm template \
    cnpg cnpg/cloudnative-pg \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace cnpg-system \
    --values cnpg-values.yaml \
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
#### show versions
```bash
crane ls quay.io/jetstack/charts/cert-manager | tail
```
#### show versions
```bash
crane ls quay.io/jetstack/charts/cert-manager-approver-policy | tail
```
#### get crds
```bash
wget -O cert-manager-crds.yaml https://github.com/cert-manager/cert-manager/releases/download/{version}/cert-manager.crds.yaml # <- EDIT THIS
```
#### get values
```bash
helm show values oci://quay.io/jetstack/charts/cert-manager --version {version} > cert-manager-values.yaml # <- EDIT THIS
```
#### an configuration
```yaml
installCRDs: false
replicaCount: 3
disableAutoApproval: true
```
#### create template
```bash
helm template \
    cert-manager oci://quay.io/jetstack/charts/cert-manager \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace cert-manager-system \
    --values cert-manager-values.yaml \
    > cert-manager.yaml
helm template \
    cert-manager-approver-policy oci://quay.io/jetstack/charts/cert-manager-approver-policy \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace cert-manager-system \
    > cert-manager-approver-policy.yaml
```
#### apply cert-manager and crds
```bash
kubectl create namespace cert-manager-system
kubectl label namespace cert-manager-system \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/audit=restricted
kubectl apply --namespace cert-manager-system -f cert-manager-crds.yaml
kubectl apply -f cert-manager.yaml
kubectl apply -f cert-manager-approver-policy.yaml
```
#### watch cert-manager
```bash
watch kubectl get pods -n cert-manager-system
```
### envoy-gateway
#### show versions
```bash
crane ls docker.io/envoyproxy/gateway-crds-helm | tail
```
#### get values
```bash
helm show values oci://docker.io/envoyproxy/gateway-crds-helm --version {version} > envoy-gateway-values.yaml # <- EDIT THIS
```
#### an configuration
```yaml
service:
  type: "ClusterIP"
deployment:
  replicas: 3
crds:
  gatewayAPI:
    enabled: true
    channel: standard
  envoyGateway:
    enabled: true
config:
  envoyGateway:
    extensionApis:
      enableBackend: true
```
#### get templates
```bash
helm template \
    envoy-gateway oci://docker.io/envoyproxy/gateway-crds-helm \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --values envoy-gateway-values.yaml \
    > envoy-gateway-crds.yaml
helm template \
    envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
    --kube-version {version} \ # <- EDIT THIS
    --version {version} \ # <- EDIT THIS
    --namespace envoy-gateway-system \
    --values envoy-gateway-values.yaml \
    --skip-crds \
    > envoy-gateway.yaml
```
#### apply envoy-gateway and crds
```bash
kubectl create namespace envoy-gateway-system
kubectl label namespace envoy-gateway-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged --overwrite
kubectl apply --server-side -f envoy-gateway-crds.yaml
kubectl apply --server-side -f envoy-gateway.yaml
```
#### watch envoy-gateway
```bash
watch kubectl get pods -n envoy-gateway-system
```
> [!IMPORTANT]
> its better to comment https part of an gateway until certificate is created
#### cloudflare dns issuer (cloudflare-dns-issuer.yaml)
> [!IMPORTANT]
> User Profile > API Tokens > API Tokens
> * Permissions:
>   * Zone - DNS - Edit
>   * Zone - Zone - Read
> * Zone Resources:
>   * Include - All Zones
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: envoy-gateway-system
type: Opaque
stringData:
  api-token: {token} # <- EDIT THIS
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: cloudflare-dns-issuer
  namespace: envoy-gateway-system
spec:
  acme:
    email: {email} # <- EDIT THIS
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: issuer-account-key-secret
    solvers:
    - dns01:
        cloudflare:
          email: {email} # <- EDIT THIS
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```
#### apply cloudflare issuer
```bash
kubectl apply -f cloudflare-dns-issuer.yaml
```
#### gateway yaml (envoy-gateway-default.yaml)
```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: envoy-gateway-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: ClusterIP
      envoyDeployment:
        pod:
          annotations:
             gateway.envoyproxy.io/enable-proxy-protocol: "true"
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway-class
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: envoy-gateway-config
    namespace: envoy-gateway-system
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: envoy-gateway
  namespace: envoy-gateway-system
spec:
  gatewayClassName: envoy-gateway-class
  listeners:
    - name: domain-http
      protocol: HTTP
      port: 80
      hostname: "domain"
      allowedRoutes:
        namespaces:
          from: Same
    - name: domain-https
      protocol: HTTPS
      port: 443
      hostname: "domain"
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: domain-tls
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: v1
kind: Service
metadata:
  name: envoy-gateway-static
  namespace: envoy-gateway-system
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 10080
      protocol: TCP
    - name: https
      port: 443
      targetPort: 10443
      protocol: TCP
    - name: ssh
      port: 22
      targetPort: 10022
      protocol: TCP
  selector:
    gateway.envoyproxy.io/owning-gateway-name: envoy-gateway
    gateway.envoyproxy.io/owning-gateway-namespace: envoy-gateway-system
```
#### apply gateway
```bash
kubectl apply -f envoy-gateway-default.yaml
```
#### proxy deployment yaml (envoy-gateway-haproxy.yaml)
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: envoy-gateway-haproxy
  namespace: envoy-gateway-system
spec:
  selector:
    matchLabels:
      app: envoy-gateway-haproxy
  template:
    metadata:
      labels:
        app: envoy-gateway-haproxy
    spec:
      nodeSelector:
        node-role.kubernetes.io/edge: ""
      tolerations:
        - key: "node.kubernetes.io/edge"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
        - name: haproxy
          image: haproxy:3.3-alpine
          ports:
            - name: tcp
              containerPort: 80
              hostPort: 80
              protocol: TCP
            - name: tcp
              containerPort: 443
              hostPort: 443
              protocol: TCP
            - name: tcp
              containerPort: 22
              hostPort: 22
              protocol: TCP
          volumeMounts:
            - name: envoy-gateway-haproxy-config
              mountPath: /usr/local/etc/haproxy/haproxy.cfg
              subPath: haproxy.cfg
      volumes:
        - name: envoy-gateway-haproxy-config
          configMap:
            name: envoy-gateway-haproxy-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-gateway-haproxy-config
  namespace: envoy-gateway-system
data:
  haproxy.cfg: |
    global
        log stdout format raw local0

    defaults
        log global
        mode tcp
        option tcplog
        timeout connect 10s
        timeout client 1m
        timeout server 1m

    resolvers k8s_dns
        parse-resolv-conf
        hold valid 10s

    frontend http_front
        bind *:80
        mode tcp
        default_backend envoy_gateway_http

    backend envoy_gateway_http
        mode tcp
        server envoy envoy-gateway-static.envoy-gateway-system.svc.cluster.local:80 check resolvers k8s_dns init-addr none send-proxy-v2

    frontend https_front
        bind *:443
        mode tcp
        default_backend envoy_gateway_https

    backend envoy_gateway_https
        mode tcp
        server envoy envoy-gateway-static.envoy-gateway-system.svc.cluster.local:443 check resolvers k8s_dns init-addr none send-proxy-v2

    frontend ssh_front
        bind *:22
        mode tcp
        timeout client 2h
        default_backend envoy_gateway_ssh

    backend envoy_gateway_ssh
        mode tcp
        timeout server 2h
        server envoy envoy-gateway-static.envoy-gateway-system.svc.cluster.local:22 check resolvers k8s_dns init-addr none send-proxy-v2
```
#### apply proxy deployment
```bash
kubectl apply -f envoy-gateway-haproxy.yaml
```
