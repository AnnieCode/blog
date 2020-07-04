---
title: "YAML 과 Kubernetes"
date: 2020-07-03 10:00:00 +0900
categories: Kubernetes
---
YAML 파일의 기본적인 내용과 쿠버네티스에서의 역할을 살펴봅니다.

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

- 가독성, 표현력, 확장성이 좋습니다.
- 사용하기 쉽습니다.
- 프로그래밍 언어 간 호환이 잘 됩니다.
- 애자일(agile) 언어의 네이티브 데이터 구조에 잘 맞습니다.
- 툴에서 지원하기에 좋은 일관된 모델로 되어 있습니다.
- One-pass 처리를 지원합니다.
- 사용이 편리합니다. (커맨드 라인에 파라미터를 추가할 필요가 없습니다)
- 유지보수(변경 사항 추적)가 가능합니다.
- 유연성이 좋습니다. 커맨드 라인에서 처리하는 것보다 훨씬 복잡한 구조를 처리할 수 있습니다.


#### 3. YAML, JSON, XML 비교

![비교표](https://developer.ibm.com/developer/tutorials/yaml-basics-and-usage-in-kubernetes/images/table1.png "비교표")

셋중에 가장 먼저 활성화된 것은 XML입니다. 구조화된 문서 작성을 위해 고안되었고 많이 사용되어 왔지만, 설계상 제약사항이 많이 있습니다.

JSON은 작성/파싱을 위한 단순함과 일관성이 목적으로 설계되었습니다. 하지만 가독성이 떨어지긴 하지만, 프로그래밍 환경에서 데이터 처리에 용이합니다.

YAML의 설계 목적은 가독성과 보다 완벽한 정보 모델입니다. 위 표에서 볼 수 있듯이 YAML은 JSON과 유사한 면이 많습니다.  


#### 4. YAML 파일의 기본 구조

YAML 파일의 빌딩 블록 삼총사
1. Key Value Pair - 가장 기본이 되는 입력 유형은 키-값 쌍입니다. 키(Key)와 콜론(:)은 붙여서 쓰고, 한칸 띄운 다음 값(Value)을 입력해야 합니다.
2. Array/List - 리스트(List)는 리스트의 이름과 리스트에 속한 아이템들로 이루어집니다. 리스트의 각 아이템들은 - 로 시작합니다. 여러 개의 리스트를 작성할 수 있지만, 들여쓰기(indentation)를 주의해야 합니다.
3. Dictionary/Map - YAML 파일의 좀 더 복잡한 형태입니다.

![YAML 파일 구조](https://developer.ibm.com/developer/tutorials/yaml-basics-and-usage-in-kubernetes/images/table2.png "YAML 파일 구조")

##### YAML 파일 작성할 때 지켜야 할 규칙

**-- 들여쓰기의 중요성 --**

예를 들어 아래의 다이어그램의 구조로 YAML 을 작성한다고 생각해보겠습니다.
![Diagram](https://developer.ibm.com/developer/tutorials/yaml-basics-and-usage-in-kubernetes/images/3banana.png "잘된 YAML 예제")

"Banana"에 대한 정보로 3가지 속성을 정의했습니다.
1. Calories = 200
2. Fat = 0.5g
3. Carbs = 30g

그런데 여기에서 들여쓰기나 탭을 잘못 사용하면, YAML 오브젝트의 의미가 다음과 같이 변해 버리니 주의하셔야 합니다.

![Diagram](https://developer.ibm.com/developer/tutorials/yaml-basics-and-usage-in-kubernetes/images/4banana.png "들여쓰기 잘못하면?")

##### YAML 기본에서 중요한 점

![YAML기본](https://developer.ibm.com/developer/tutorials/yaml-basics-and-usage-in-kubernetes/images/5yamlbasics.png "YAML 기본")

- SCALAR - `date` 를 쓸 때는 "YYYY-MM-DD" 형식으로 작성하면 `date` 변수에 지정할 수 있습니다.
- COLLECTIONS - Billing Address를 입력했고, Shipping Address를 Billing Address와 동일하게 사용하고 싶다고 가정해보겠습니다. 이렇게 재사용을 하고 싶은 경우에는, Billing Address 앞에 &와 ID 이름을 적어줍니다. 그런 다음, 동일한 내용을 쓰고 싶은 Shipping Address 자리에 \*표와 함께 ID 이름을 쓰면 됩니다. 데이터 복사/붙여넣기에 유용한 기술입니다.
- MULTI-LINE COLLECTIONS - 내용을 여러 줄로 써야 할 경우에는 `|`표시를 사용하면 됩니다.
- LISTS/DICTIONARIES - 앞에 내용 참조
- MULTI-LINE FORMATTING - 긴 문자열을 입력하면서 형식화를 하고 싶은 경우에는, `>` 표시를 사용하면 됩니다.



#### 5. Kubernetes 에서 YAML 파일의 역할

POD, 서비스, 배포와 같은 Kubernetes의 리소스는 YAML 파일을 사용하여 선언적 방식으로 만들어집니다.
아래 예제는 배포 리소스를 생성하는 내용이 담긴 YAML 입니다.

![배포YAML](https://developer.ibm.com/developer/tutorials/yaml-basics-and-usage-in-kubernetes/images/6yamlex.png "배포 리스스 YAML")

각 필드가 의미하는 바는 다음과 같습니다

- `replicas` - 배포(deployment)시 생성할 POD의 개수. 컨테이너 어플리케이션을 확장하고 싶을 때 이 값을 늘려주면 됩니다.
- `spec.strategy.type` - 배포할 어플리케이션이 여러 버전이 있는 경우, 어플리케이션 중단 없이 업데이트하고 싶을 때 "롤링 업데이트 전략(Rolling Update strategy)"을 사용하여 한번에 하나씩 POD를 업데이트할 수 있습니다.
- `spec.strategy.rollingUpdate.maxUnavailable` - 롤링 업데이트하는 동안 unavailable 상태가 될 수 있는 최대 POD 개수
- `spec.strategy.rollingUpdate.maxSurge` - 의도한 POD(파드)의 수에 대해 생성할 수 있는 최대 POD의 수를 지정하는 선택적 필드. 기본값은 25%.
- `spec.minReadySeconds` - 최소 대기 시간(초). 새롭게 생성된 POD(파드)의 컨테이너가 어떤 것과도 충돌하지 않고 사용할 수 있도록 준비되어야 하는 최소 시간(초)를 지정하는 선택적 필드. 기본값은 0 (파드가 준비되는 즉시 사용할 수 있는 것으로 간주).
- `spec.revisionHistoryLimit` - 롤백을 허용하기 위해 보존할 이전 ReplicaSet의 수를 지정하는 선택적 필드. 이 이전 ReplicaSet은 `etcd`의 리소스를 소비하고, `kubectl get rs`의 결과를 가득차게 만듬.
- `spec.template.metadata.labels` - 배포 스펙에 라벨 추가
- `spec.selector` - 쿠버네티스 배포(deployment) 컨트롤러가 관리할 POD를 찾는 방법 정의. 예제에서는 "app"과 "deployer"라는 라벨을 가진 POD만을 찾습니다.

사용 가능한 모든 필드에 대한 자세한 설명은 [쿠버네티스 홈페이지]를 참고하십시오.

[쿠버네티스 홈페이지]: https://kubernetes.io
