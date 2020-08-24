---
title: "Kubernetes 이중화 아키텍처3"
date: 2020-08-24 08:26:28 +0800
categories: kubernetes
---

# Kubernetes Master 노드 아키텍처

- 쿠버네티스는 etcd, APIServer, Scheduler, Controller로 구성
- APIServer는 설정 저장소인 ETCD에 접근하여 데이터를 조회하거나 저장
- Master Node의  Scheduler, Controller는 APIServer를 통해 Kubernetes Orchestration 수행
- Worker Node의 Kubelet, Kubeproxy은 Master 노드의 APIServer를 통해 데이터 조회 및 Worker Node 반영

[그림1. Kubernetes아키텍처(출처: kubernetes in action chapter11)]
![Kubernetes 아키텍처 이중화-2](0002.png)


# Kubernetes Master Node 클러스터 구성
[그림2. MasterNode 클러스터 구성(출처: kubernetes in action chapter11)]
![Kubernetes 아키텍처 이중화-3](0003.png)

- [그림2]와 같이 Masternode 클러스터 구성 가능
- API서버는 Stateless로 동작하므로 앞에 LB를 구성하여 이중화 가능
- Controller Manager, Scheduler의 경우 Active/Stand-by로 구성(지속적인 클러스터 상태 감시와 대응으로 특정 시점 원하지 않는 형상이 발생 가능)

![Kubernetes 이중화-3](0001.png)
[그림3. MasterNode 클러스터 구성(출처: https://skysoo1111.tistory.com/47)]

- 때문에 [그림3]과 같이 동작

# 결론
- 마스터노드를 이중화 할 경우 API서버는 Active/Active로, Controller, Scheduler는 Active/Stand-by로 구성 가능
- 마스터 노드의 결함에 대해서는 충분히 대응이 가능할 것으로 예상
- 하지만 Kubernetes 형상 변경 시 발생할 수 있는 클러스터 내부에서 발생 가능한 오류 대응을 고려하면, Cluster Federation도 고려 필요(Kubernetes in Action, Appendix D참고)
