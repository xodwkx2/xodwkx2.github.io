---
layout: posts
title:  "Docker-Compose on OMV6"
date:   2023-12-24 00:00:00 +0900
categories: blog
---
# 1. 개요

[공식 사이트](https://www.openmediavault.org/)

openmediavault(OMV)는 자체 NAS 구축에서 빠지지 않고 등장하는 솔루션으로 많은 설치 가이드 문서나 영상이 있어 설치하는 것은 어렵지 않게 찾을 수 있다. 다만 지난 omv-extras 6.3 업데이트 이후 기존에 System > omv-extras에 있던 docker 관련 항목들(docker, portainer, yacht 등)이 사라지고 Services > Compose에 새로운 서비스로 들어가면서 달라진 부분이 있어 이 서비스에 대한 설치와 설정 방법을 기술한다.

# 2. docker_compose Plugin 설치 

omv-extras 설치에 관한 내용은 [omv-extras.org](https://wiki.omv-extras.org/doku.php?id=misc_docs:omv_extras)에서
docker_compose 설치에 관한 내용은 [imv-extras.org](https://wiki.omv-extras.org/doku.php?id=omv6:omv6_plugins:docker_compose)에서 확인 가능하다.

먼저 omv서버에 ssh접속 후 omv-extras 패키지를 설치한다. 설치에는 root 계정을 사용한다.
```
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
```

이 후 GUI에서 System > Plugins에서 돋보기 아이콘을 클릭하고 openmediavault-omvextasorg, openmediavault-compose plugin을 찾아서 설치한다.

openmediavault-omvextrasorg plugin 설치 후 Docker repository를 활성화한다. 

System > omv-extras에서 Docker repo 체크박스 선택 후 Save 버튼을 클릭한다. 


# 3. compose 설정
Compose > Settings에 들어가면 4가지 설정해야 할 부분이 있다.

- Compose Files
    - appData(compose.yaml 파일과 관련 env 파일)가 저장되는 곳
- Data(optional)
    - container의 영구 보존 데이터를 저장하는 곳
- Backup(optional)
    - appData를 백업하는 곳
- Docker
    - 임시 데이터를 저장하는 곳
    - container가 생성될 때마다 해당 위치/volumes/ 에 container 이름으로 폴더가 생성된다

# 4. compose 추가
다른 컨테이너 관리 도구처럼 OMV의 compose도 다양한 template을 제공한다. 

Services > Compose > Files 에서 +(Add) 버튼을 누르고 팝업 메뉴에서 Add from Example을 클릭한다.

Example에서 원하는 example을 찾아 선택 후 이름을 입력한 뒤 Save 버튼을 클릭한다.

해당 compose file이 추가된 것을 확인 후 선택하면 상단 메뉴가 활성화 되는 것을 볼 수 있는데 edit 버튼을 클릭하면 해당 compose file을 수정할 수 있는 페이지로 이동하게된다.

여기에서 주의해야 할 것은 ports:에 host에서 이미 열려있는 port와 겹치는 port가 있는지 확인하고 volumes:를 Compose > Settings에서 등록한 위치로 적절하게 변경한다. 

~~혹시라도 volume path를 수정하는 것을 잊어버리면
❯ ls /path/to/docker/docker-data/portainer-data/
와 같이 멍청한 곳에 파일을 저장해두고 찾지 못하는 일이 발생하게 된다.~~