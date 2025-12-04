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
