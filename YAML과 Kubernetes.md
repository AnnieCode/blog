# **YAML 과 Kubernetes**


#### 목차

1. YAML 파일의 정의
2. 개발자들이 YAML을 사용하는 이유
3. YAML, JSON, XML 비교
4. YAML 파일의 기본 구조
5. Kubernetes에서 YAML 파일의 역할


#### 1. YAML 파일의 정의

YAML은 Yet Another Markup Language 의 약자로, www.yaml.org의 정의에 따르면 "Human-friendly, data serialization standard for all programming languages"라고 한다. 직독을 하자면 모든 프로그래밍 언어를 위한 사람이 이해하기 쉬운 데이터 직렬화(?) 표준이라고 합니다.

YAML은 그동안 아래와 같은 용도로 주로 사용이 되어 왔습니다.
- Configuration file
- Log file
- Inter-process messaging
- 프로그래밍 언어 간의 데이터 공유 목적
- Object persistence (?)
- 복잡한 데이터 구조

#### 2. 개발자들이 YAML을 사용하는 이유

- 가독성, 표현력, 확장성이 좋다.
- 사용하기 쉽다.
- 프로그래밍 언어 간 호환이 쉽다.
- 애자일(agile) 언어의 네이티브 데이터 구조에 잘 맞는다.
- 툴에서 지원하기에 좋은 일관된 모델로 되어 있다.
- One-pass 처리를 지원한다.
- 사용이 편리하다. (커맨드 라인에 파라미터를 추가할 필요가 없다)
- 유지보수(변경 사항 추적)가 가능하다.
- 유연성이 좋다. 커맨드 라인에서 처리하는 것보다 훨씬 복잡한 구조를 처리할 수 있다.

#### 3. YAML, JSON, XML 비교

![비교표](http://developer.ibm.com/developer/tutorials/yaml-basics-and-usage-in-kubernetes/images/table1.png "비교표")
