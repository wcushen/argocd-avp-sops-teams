# argocd-avp-sops-teams

Prerequisites

- OpenShift Cluster 4.14+
- Download Helm, Kustomize, Age go binaries

## Bootstrap

As admin user (cluster-admin)

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

As admin user (cluster-admin)

## SOPS key setup

Generate SOPS public/private key using age

```bash
mkdir -p secrets
age-keygen -o secrets/age-key-admin.txt
echo 'secrets/age-key-admin.txt' >> .gitignore
```

Load the private key into `openshft-gitops` project (could also be from Hashi Vault, Any Cloud KMS)

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

`FIXME` - Deploy cluster priv-apps

## Team ArgoCD

As a-user user (part of a-team group). A Normal User.

### A Team

Create tenant ArgoCD for A Team

```bash
oc create -k applications/argocd/overlay/a-team
```

A Team - SOPS key setup

```bash
mkdir -p secrets
age-keygen -o secrets/age-key-a-team.txt
echo 'secrets/age-key-a-team.txt' >> .gitignore
```

Load the private key into `a-team` project (could also be from Hashi Vault, Any Cloud KMS)

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

SOPS Secret cli

```bash
export SOPS_AGE_KEY_FILE=$(pwd)/secrets/age-key-a-team.txt
export AGE_RECIPIENTS=$(grep public secrets/age-key-a-team.txt | awk '{print $4}')
```

Top level config to help with encoding SOPS secrets.

```bash
cat <<EOF > .sops.yaml
creation_rules:
  - path_regex: \.enc$
    age: '${AGE_RECIPIENTS}'
EOF
```

There are x2 applications without sops for demo purposes - `welcome-helm-novault`, `welcome-kustomize-novault`

Mount a k8s secret with SOPS - `sops-secret-message`

```bash
sops --input-type yaml --output-type yaml secrets/a-team/secret.enc
# message: Hello from the A-Team!
```

Kustomize application secret with SOPS - `welcome-sops-kustomize`

```bash
sops --input-type yaml --output-type yaml applications/welcome/overlay/a-team/welcome-sops-kustomize/deployment.enc
# message: from Kustomize-Sops!
```

Helm from Kustomize application secret with SOPS - `welcome-sops-kustomize-helm`

```bash
sops --input-type yaml --output-type yaml applications/welcome/overlay/a-team/welcome-sops-kustomize-helm/values/values.enc
# message: from Kustomize-Helm-Sops!
```

Helm application secret with SOPS - `welcome-sops-helm`

```bash
sops --input-type yaml --output-type yaml applications/welcome/chart/a-team-welcome-sops-helm/values.enc
# message: from Helm-Sops!
```

Check these ^^ `*.enc` encoded files into git.

Create all A-Team Applications using app-of-apps pattern.

```bash
oc apply -f app-of-apps/a-team-app-of-apps.yaml
```
