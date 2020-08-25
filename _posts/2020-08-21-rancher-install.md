---
title: "Rancher설치(helm활용)"
date: 2020-08-21 08:26:28 +0800
categories: [kubernetes, rancher]
---

# Helm

## Rancher Install(with helm)

### prerequisite
Helm 설치(https://github.com/jeonwoosung/k8s-install/tree/master/helm 참고)
### Rancher Install

Cert-Manager설치 필수(미 설치 시 rancher설치 불가)
※ Cert-Manager 설치 중 Rancher 삭제 시 Hang발생 가능

    kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.yaml
    helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
    kubectl create namespace cattle-system
    helm install rancher rancher-stable/rancher \
      --namespace cattle-system \
      --set hostname=rancher.oplab.co.kr

### Rancher Uninstall
 helm uninstall  rancher rancher-stable/rancher  --namespace cattle-system
