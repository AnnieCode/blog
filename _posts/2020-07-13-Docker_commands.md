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
$ docker login
```
또는
```
$ docker --u [USERNAME] --p [PASSWORD]
```
**Docker에 공유되어 있는 Repository에서 이미지를 가져오기**
```
$ docker pull [IMAGE_NAME]
$ docker pull debian
# 별도의 태그가 없이 이미지 이름만 명시하면, 자동으로 :latest 태그의 이미지를 가져옵니다.
```
```
$ docker pull [IMAGE_NAME]:[TAG_NAME]
$ docker pull ubuntu:18.04
# 태그를 붙이면, 해당 버전의 이미지를 가져옵니다.
```
```
$ docker pull [IMAGE_NAME] -a
# -a 옵션을 붙이면, 태그 구분 없이 모든 이미지를 가져옵니다.
```

**Docker 이미지에 태그 사용하기**

1. Public Docker registry로 이미지를 업로드할 경우
```
# docker tag [image_ID] [repository_name]/[image_name]:[version] 또는
# docker tage [local_image_name] [repository_name]/[image_name]:[version]
```
```
$ docker tag httpd fedora/httpd:1.0
```
```
$ docker tag httpd:test fedora/httpd:1.0.test
```

2. Private registry로 이미지를 업로드할 경우

```
 # docker tag [image_ID] [hostname]:[port_number]/[repository_name]/[image_name]:[version]

$ docker tag 0e5574283393 myregistryhost:5000/fedora/httpd:1.0
```

3. 태그를 붙인 정보를 삭제하기
태그로 붙인 이미지의 이름을 그대로 사용하여 rmi 명령어로 삭제합니다. 
원본 이미지는 삭제되지 않고, 태그 정보만 삭제됩니다. 

```
$ docker rmi [repository_name]:[version] 
```

[참고자료]

https://docs.docker.com/engine/reference/commandline/rmi/

https://docs.docker.com/engine/reference/commandline/tag/

