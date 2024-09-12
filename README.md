# gitops

## Terraform

```
# ----------------------------------------------------------------------------------------------------------------------
# TERRAFORM PROVIDERS
# ----------------------------------------------------------------------------------------------------------------------
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    github = {
      source  = "integrations/github"
      version = ">= 4.22.0"
    }

    aws = {
      source  = "hashicorp/aws"
      version = ">= 3.75.1"
    }

    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.0, != 2.6.0"
    }

    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.9.0"
    }

    tls = {
      source  = "hashicorp/tls"
      version = "4.0.5"
    }
  }
}

# ----------------------------------------------------------------------------------------------------------------------
# DEPLOY HELM CHART
# ----------------------------------------------------------------------------------------------------------------------
locals {
  helm_chart_values = yamldecode(file("${path.module}/values.yaml"))
  helm_chart_values_apps = yamldecode(file("${path.module}/values-apps.yaml"))
}

resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  version          = "7.5.2"
  namespace        = "argocd"
  create_namespace = true

  values  = [yamlencode(local.helm_chart_values)]
  wait    = true
  timeout = 300
}

output "app_version" {
  value = helm_release.argocd.version
}

# ----------------------------------------------------------------------------------------------------------------------
# ADD RESPOSITORIES
# ----------------------------------------------------------------------------------------------------------------------
# GitHub deploy key
resource "tls_private_key" "gitops" {
  algorithm   = "ECDSA"
  ecdsa_curve = "P256"
}

data "github_repository" "gitops" {
  name = "gitops"
}

resource "github_repository_deploy_key" "gitops" {
  title      = "argocd"
  repository = data.github_repository.gitops.name
  key        = tls_private_key.gitops.public_key_openssh
  read_only  = true
}

resource "kubernetes_secret" "gitops" {
  depends_on = [helm_release.argocd]

  metadata {
    name      = "argocd-repo-gitops"
    namespace = "argocd"
    labels = {
      "argocd.argoproj.io/secret-type" = "repository"
    }
  }

  data = {
    name          = "gitops"
    project       = "default"
    sshPrivateKey = tls_private_key.gitops.private_key_openssh
    type          = "git"
    url           = "git@github.com:ubajze/gitops.git"
  }
}

resource "helm_release" "argocd_apps" {
  depends_on = [helm_release.argocd]

  name             = "argocd-apps"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argocd-apps"
  version          = "2.0.1"
  namespace        = "argocd"
  create_namespace = true

  values  = [yamlencode(local.helm_chart_values_apps)]
  wait    = true
  timeout = 300
}
```

`values.yaml`:
```
crds:
  keep: false
dex:
  enabled: false
global:
  logging:
    level: "info"
```

`values-apps.yaml`:
```
---
applications:
  argocd:
    namespace: "argocd"
    additionalAnnotations:
      argocd.argoproj.io/sync-wave: "200"
    project: "default"
    sources:
      - repoURL: "https://argoproj.github.io/argo-helm"
        chart: "argo-cd"
        targetRevision: "7.5.2"
        helm:
          valueFiles:
            - $values/argocd/argocd-values.yaml
      - repoURL: "git@github.com:ubajze/gitops.git"
        targetRevision: "main"
        ref: "values"
    destination:
      server: "https://kubernetes.default.svc"
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
  core:
    namespace: "argocd"
    project: "default"
    source:
      repoURL: "git@github.com:ubajze/gitops.git"
      targetRevision: "main"
      path: "core/"
    destination:
      server: "https://kubernetes.default.svc"
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
```