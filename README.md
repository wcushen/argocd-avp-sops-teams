# argocd-avp-sops-teams

Example to demonstrate multi-tenant ArgoCD for teams using SOPS secrets with applications. Various application deployment methods using yaml, helm and kustomize.

![images/sre-cluster-argo-team-namespaced.png](images/sre-cluster-argo-team-namespaced.png)

- The RedHat GitOps Operator (cluster scoped)
- A Ops-SRE (cluster scoped) ArgoCD instance
- Team (namespace scoped) ArgoCD instances

We use [SOPS](https://github.com/getsops/sops), [AGE](https://github.com/FiloSottile/age) and [ArgoCD Vault Plugin](https://github.com/argoproj-labs/argocd-vault-plugin) to encrypt secret values at rest and inject them into resources.

👓 Watch this [SOPS VIDEO](https://www.youtube.com/watch?v=V2PRhxphH2w) if you are new to sops. 👓

## Prerequisites

- OpenShift Cluster 4.14+
- Download Helm, Kustomize, SOPS, Age - clients

```bash
# helm
HELM_VERSION=3.16.1
curl -skL -o /tmp/helm.tar.gz https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz && \
    tar -C /tmp -xzf /tmp/helm.tar.gz && \
    mv -v /tmp/linux-amd64/helm /usr/local/bin && \
    chmod -R 775 /usr/local/bin/helm && \
    rm -rf /tmp/linux-amd64
```

```bash
# kustomize
KUSTOMIZE_VERSION=5.4.3
curl -skL -o /tmp/kustomize.tar.gz https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv${KUSTOMIZE_VERSION}/kustomize_v${KUSTOMIZE_VERSION}_linux_amd64.tar.gz && \
    tar -C /tmp -xzf /tmp/kustomize.tar.gz && \
    mv -v /tmp/kustomize /usr/local/bin && \
    chmod -R 775 /usr/local/bin/kustomize && \
    rm -rf /tmp/linux-amd64
```

```bash
# age
AGE_VERSGION=1.2.0
curl -skL -o /tmp/age.tar.gz https://github.com/FiloSottile/age/releases/download/v${AGE_VERSGION}/age-v${AGE_VERSGION}-linux-amd64.tar.gz && \
    tar -C /tmp -xzf /tmp/age.tar.gz && \
    mv -v /tmp/age/age /usr/local/bin && \
    mv -v /tmp/age/age-keygen /usr/local/bin && \
    chmod -R 775 /usr/local/bin/age && \
    chmod -R 775 /usr/local/bin/age-keygen && \
    rm -rf /tmp/age
```

```bash
# sops
SOPS_VERSION=3.9.0
curl -skL -o /usr/local/bin/sops https://github.com/getsops/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.amd64 && \
    chmod -R 775 /usr/local/bin/sops
```

Clone or fork this repo and cd into it.

## Bootstrap

As admin user (cluster-admin).

Install GitOps Subscriptions.

```bash
oc apply -f bootstrap/setup-subs.yaml
```

Install CR.

```bash
oc apply -f bootstrap/setup-cr.yaml
```

Setup users with htpassd.

```bash
# create htpasswd file
htpasswd -bBc /tmp/htpasswd admin password
htpasswd -bB /tmp/htpasswd a-user password
htpasswd -bB /tmp/htpasswd b-user password

# set cluster-admin
oc adm policy add-cluster-role-to-user cluster-admin admin

# create htpasswd k8s secret
oc delete secret htpasswdidp-secret -n openshift-config
oc create secret generic htpasswdidp-secret -n openshift-config --from-file=/tmp/htpasswd

# add oauth config
cat << EOF > /tmp/htpasswd.yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswdidp-secret
EOF
oc apply -f /tmp/htpasswd.yaml -n openshift-config
watch oc get co/authentication
```

## Cluster ArgoCD

As `admin` user (cluster-admin)

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

`FIXME` - Deploy cluster priv-apps into this ArgoCD.

## Team ArgoCD

As `a-user` user (part of `a-team` group). A Normal User.

### A Team

#### Setup

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
cat <<'EOF' > .sops.yaml
creation_rules:
  - path_regex: .*/a-team/.*\.enc$
    age: '${AGE_RECIPIENTS}'
EOF
```

#### Apps

Create secrets, check then into git, deploy apps with ArgoCD.

1. There are x2 applications without sops for demo purposes. You can mount the k8s secret from sops just like normal (trivial, not shown).

  - [`welcome-helm-novault`](app-of-apps/a-team/welcome-helm-novault.yaml)
  - [`welcome-kustomize-novault`](app-of-apps/a-team/welcome-kustomize-novault.yaml)

2. Mount a k8s secret with SOPS - [`sops-secret-message`](app-of-apps/a-team/sops-secret-message.yaml). It creates a normal k8s secret called `sops-secret-message`.

```bash
sops --input-type yaml --output-type yaml secrets/a-team/secret.enc
# message: Hello from the A-Team!
```

3. Kustomize application secret with SOPS - [`welcome-sops-kustomize`](app-of-apps/a-team/welcome-sops-kustomize.yaml). It shows how to inject a sops secret into the deployment yaml with kustomize.

```bash
sops --input-type yaml --output-type yaml applications/welcome/overlay/a-team/welcome-sops-kustomize/deployment.enc
# message: from Kustomize-Sops!
```

4. Helm from Kustomize application secret with SOPS - [`welcome-sops-kustomize-helm`](app-of-apps/a-team/welcome-sops-kustomize-helm.yaml). It shows how to create a helm chart deployment from kustomize and injects a sops secret into the inline helm value.

```bash
sops --input-type yaml --output-type yaml applications/welcome/overlay/a-team/welcome-sops-kustomize-helm/values/values.enc
# message: from Kustomize-Helm-Sops!
```

5. Helm application secret with SOPS - [`welcome-sops-helm`](app-of-apps/a-team/welcome-sops-helm.yaml). It shows how to create a helm chart based application using just an argocd application and injects a sops secret into a helm value.

```bash
sops --input-type yaml --output-type yaml applications/welcome/chart/a-team-welcome-sops-helm/values.enc
# message: from Helm-Sops!
```

Check all of the ^^ `*.enc` encoded files into git, they are AES256 encrypted using AGE.

Create all A-Team Applications using app-of-apps pattern.

```bash
oc apply -f app-of-apps/a-team-app-of-apps.yaml
```

#### Testing

A Team test the urls.

```bash
export BASE_DOMAIN=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')

# welcome
curl -s https://welcome-a-team.apps.${BASE_DOMAIN}

# welcome-helm-novault
curl -s https://welcome-helm-novault-a-team.apps.${BASE_DOMAIN}

# welcome-sops-helm
curl -s https://welcome-sops-helm-a-team.apps.${BASE_DOMAIN}

# welcome-sops-kustomize
curl -s https://welcome-sops-kustomize-a-team.apps.${BASE_DOMAIN}

# welcome-sops-kustomize-helm
curl -s https://welcome-sops-kustomize-helm-a-team.apps.${BASE_DOMAIN}
```

Should result in something like this (the `message` sops secret is in the Hello!).

```bash
Hello from Bob ! Welcome to OpenShift from welcome-7685d4794b-v9z88:10.128.0.137
Hello World ! Welcome to OpenShift from welcome-helm-novault-5fcb9d7ff4-4pdrk:10.128.0.78
Hello from Helm-Sops! ! Welcome to OpenShift from welcome-sops-helm-796dfd845b-7qrjm:10.128.0.193
Hello from Kustomize-Sops! ! Welcome to OpenShift from welcome-sops-kustomize-64d4585c6c-x9f4w:10.128.0.158
Hello from Kustomize-Helm! ! Welcome to OpenShift from welcome-sops-kustomize-helm-59d5dbf4b6-tdwss:10.128.0.125
```

### B Team

Rinse and repeat the a-team instructions - but for B Team with the `b-user` 🙂
