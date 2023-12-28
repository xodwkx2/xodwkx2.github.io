---
layout: posts
title:  "쿠버네티스 클러스트 프로비저닝 with RKE2"
date:   2023-12-23 00:00:00 +0900
categories: blog
---
# 1. 개요

[공식 사이트](https://docs.rke2.io/install/quickstart)

Ubuntu 20.04를 기준으로 RKE2 설치를 통해 Kubernetes cluster를 provisioning하여 lab을 구성하는 과정을 설명한다.

### TL;DR
```
curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service

export PATH=$PATH:/var/lib/rancher/rke2/bin

mkdir ~/.kube
  
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
```

# 2. 사전 준비
Kubernetes를 설치하기 전 필요한 linux 설정을 완료한다.
단일 노드 쿠버네티스 클러스터를 구성 할 생각이라면 넘겨도 되지만 멀티 노드 쿠버네티스 클러스터를 구성할 생각이라면 아래 내용을 확인하고 해당 값들이 중복이 없는지 확인해야 한다.

- hostname
```bash
hostname
sudo hostnamectl set-hostname [UNIQUE HOSTNAME]
```
- product_uuid
```bash
sudo cat /sys/devices/virtual/dmi/id/product_uuid
```
- mac address
```bash
ip link
```

다음으로 swap을 비활성화한다.
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

# 3. RKE2 설치
아래 명령으로 RKE2 설치 스크립트를 실행한다.
```bash
curl -sfL https://get.rke2.io | sudo sh -
```
[설정 옵션](https://docs.rke2.io/install/configuration)을 통해 특정 버전을 설치하거나 master node 용도로 설치할지 worker node 용으로 설치할지 등 다양한 설정이 가능하다.


# 4. 설치 확인
RKE2는 서비스 형태로 실행되며 설치가 완료된 후에는 자동으로 실행되지 않고 멈춰있는 상태로 있게된다. systemctl status 명령으로 상태를 확인할 수 있다.
```bash
sudo systemctl status rke2-server.service
```
다음으로 시스템 재시작 후에도 자동으로 서비스가 올라올 수 있게 한다.
```bash
sudo systemctl enable rke2-server.service
```
서비스를 시작하기 전에 /etc/rancher/rke2/config.yaml 파일을 생성 후 [서버 설정](https://docs.rke2.io/reference/server_config)이 가능하다.
마지막으로 systemctl start 명령으로 서비스를 시작한다.
```bash
sudo systemctl start rke2-server.service
```
systemctl status 명령으로 상태를 확인한다.

# 5. 설정
kubectl 및 각종 바이너리는 /var/lib/rancher/rke2/bin에서 찾을 수 있다. 
export 명령으로 PATH에 추가하는 내용을 쉘 rc 파일에 넣어놓는다.
```bash
export PATH=$PATH:/var/lib/rancher/rke2/bin
```

config 파일은 /etc/rancher/rke2/rke2.yaml 파일에서 확인이 가능하다. 나머지 설정은 일반적인 쿠버네티스 기본 사용 설정과 동일하다.
```bash
mkdir ~/.kube
  
sudo cp /etc/rancher/rke2/rke2.yaml  ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
```

# 6. 최종 확인

```bash
kubectl version -o yaml
kubectl get ns
kubectl get pods --all-namespaces
```