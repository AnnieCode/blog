---
title: "Docker 명령어 정리"
date: 2020-07-13 21:00:00 +0900
categories: Kubernetes
tags: Docker
sidebar:
  nav: "main"
---

### Docker 명령어 정리

**Docker Hub에 가입한 계정으로 로그인하기**  
Docker Hub Repository 에 내가 만든 이미지를 업로드하려면 로그인해야 한다.
```
docker login
```
또는
```
docker --u [USERNAME] --p [PASSWORD]
```
**Docker에 공유되어 있는 Repository에서 이미지를 가져오기**
```
docker pull [IMAGE_NAME]
docker pull debian
# 별도의 태그가 없이 이미지 이름만 명시하면, 자동으로 :latest 태그의 이미지를 가져옵니다.
```
```
docker pull [IMAGE_NAME]:[TAG_NAME]
docker pull ubuntu:18.04
# 태그를 붙이면, 해당 버전의 이미지를 가져옵니다.
```
```
docker pull [IMAGE_NAME] -a
# -a 옵션을 붙이면, 태그 구분 없이 모든 이미지를 가져옵니다.
```
