# argocd-avp-sops-teams

Prerequisites

- OpenShift Cluster 4.14+
- Download Helm, Kustomize, Age go binaries

## Bootstrap

Install Subscriptions

```bash
oc apply -f bootstrap/setup-subs.yaml
```

Install CR

```bash
oc apply -f bootstrap/setup-cr.yaml
```

`FIXME` - Setup Users & Groups

```bash
# Add user admin to `admin` group, cluser-admin
# Add users a-user, b-user
```

## Cluster ArgoCD

## SOPS key setup

```bash
mkdir -p secrets
age-keygen -o secrets/age-key-admin.txt
echo 'secrets/age-key-admin.txt' >> .gitignore
```

```bash
oc apply -f - <<EOF
apiVersion: v1
stringData:
  age-key.txt: |
    $(grep AGE-SECRET-KEY secrets/age-key-admin.txt)
kind: Secret
metadata:
  name: sops-age-key
  namespace: openshift-gitops
type: Opaque
EOF
```

## Team ArgoCD

A Team

```bash
oc create -k applications/argocd/overlay/a-team
```

```bash
mkdir -p secrets
age-keygen -o secrets/age-key-a-team.txt
echo 'secrets/age-key-a-team.txt' >> .gitignore
```

```bash
oc apply -f - <<EOF
apiVersion: v1
stringData:
  age-key.txt: |
    $(grep AGE-SECRET-KEY secrets/age-key-a-team.txt)
kind: Secret
metadata:
  name: sops-age-key
  namespace: a-team
type: Opaque
EOF
```

SOPS Secret

```bash
export SOPS_AGE_KEY_FILE=$(pwd)/secrets/age-key-a-team.txt
export AGE_RECIPIENTS=$(grep public secrets/age-key-a-team.txt | awk '{print $4}')
```

```bash
cat <<EOF > .sops.yaml
creation_rules:
  - path_regex: \.enc$
    age: '${AGE_RECIPIENTS}'
EOF
```

```bash
sops --input-type yaml --output-type yaml secrets/a-team/secret.enc
# message: Hello from the A-Team!
```

```bash
sops --input-type yaml --output-type yaml applications/welcome/overlay/a-team/welcome-sops-kustomize/deployment.enc
# message: from the A-Team!
```

```bash
sops --input-type yaml --output-type yaml applications/welcome/overlay/a-team/welcome-sops-kustomize-helm/values/values.enc
# message: from Kustomize-Helm!
```

```bash
oc apply -f app-of-apps/a-team-app-of-apps.yaml
```
