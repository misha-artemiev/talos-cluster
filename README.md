# talos cluster

## secrets

### create secrets
```
talhelper gensecret > talsecret.sops.yaml
```
### encrypt secrets
```
sops -e -i talsecret.sops.yaml
```
