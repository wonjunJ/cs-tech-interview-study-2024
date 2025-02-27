---
created: 2024-09-14
tags:
  - kubernetes
  - workload
  - deployment
dg-publish: true
분류: 개념
aliases:
  - 디플로이먼트
---
# 개요
> A Deployment manages a set of Pods to run an application workload, usually one that doesn't maintain state.

상태를 유지하지 않는 애플리케이션을 실행하는 파드를 관리하는 객체라고 정의된다.
디플로이먼트.
배포라고 부르면.. 너무 직관적이지 않긴 하다.
이전에 어떤 분이 init을 초기화라고 번역해버리면 오히려 잘 와닿지 않는 것 같다고 말씀하셨는데, 확실히 이 리소스는 번역을 안하는 게 나은 것 같다.

웹 환경에서 대부분의 통신은 stateless하게 이뤄지기에, 이 리소스는 매우 흔하게 사용된다.
가령 초당 100개의 요청을 처리할 수 있는 어플리케이션 로직이 있는데 트래픽이 초당 300개가 들어온다면, 스케일아웃을 하는 것이 유효한 선택지다.
이때 사용하는 게 바로 디플로이먼트!
# 양식
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80

```
양식으로 보면 조금 더 설명이 쉬울 듯.
- kind 
	- 종류를 Deployment로 선언
- metadata
	- 그냥 흔히 하는 설정 정도.
- spec
	- replicas
		- 관리할 파드의 복제본이 몇 개 일지 개수를 정한다.
	- selector
		- 관리할 파드가 무엇인지 포착하는 조건
		- 사용 방식은 [[셀렉터]]를 참조하자.
	- template
		- 관리할 파드의 양식
		- [[Pod]]에 들어갈 법한 양식이 담겨있다.

다시 해석해보자면, template에 적힌 파드를 replicas에 적힌 개수로 유지시키고 관리하는 것이 바로 디플로이먼트라는 것이다.
위의 예시에서는 nginx라는 이름의 파드를 3개 유지시키는 nginx-deployment라는 디플로이먼트가 만들어질 것이다.

왜 템플릿을 여기에서 명시하는데 굳이 또 셀렉터를 지정하는 것일까?
어차피 디플은 자신의 템플릿으로 써진 파드를 자신의 관리 하에 있는 것으로 삼으면 안 되는 걸까?
결국 지금 이 파일은 http 요청을 파일화시킨 것에 불과하다는 것을 상기하자.
아래의 다양한 기능을 행하기 위해서 디플로이먼트는 자신이 관리하는 파드가 정확히 무엇인지 포착할 조건이 명시될 필요가 있다.
이후 내용, [[#동작 구조]]와 이후 조작 방법을 보면서 조금씩 구체화될 것이다.

근데 이러면 또 한가지 궁금점이 생긴다.
템플릿에 명시되지 않는 다른 것까지 포착되도록 셀렉터를 쓴다면 무슨 일이 발생할까?
[[디플로이먼트 템플릿에 없는 파드를 포착한다면]]
# 기능
[[워크로드]]로서 파드를 관리한다.
디플로이먼트가 관리하는 파드는 다음의 특징을 가진다.
- stateless
	- 통신이 유지되지 않는 파드가 적합하다.
- 지속 실행
	- 한번만 실행되어 끝나는 것이 아니라 지속적으로 실행되는 애플리케이션에 적합하다.	

그러니까 딱 좋은 예시가 웹서버인 것이다.
stateful이 중요하다면 [[StatefulSet]], 한번만 실행되면 [[Job]]을 사용하라.
## 용례
디플로이먼트를 가지고 조작을 하는 다양한 유즈케이스를 고려해보자.
- 위에서 말한 파드 여러 개 띄우기
- 파드 스펙을 업데이트하여 새로 띄우기
	- 이 경우 이전에 만들어진 파드들과 다른 새로운 파드들이 새로운 버전으로 만들어진다.
	- 각 파드 묶음은 [[ReplicaSet]]으로 관리되는데, 이건 [[#동작 구조]]에서 이야기하겠다.
- 이전 파드 스펙 버전으로 돌아가기
	- 무려 버전 롤백도 가능하다.
- 파드 개수 더 늘리는 스케일 아웃

크게 봤을 때 디플로이먼트가 할 수 있는 일은 두 가지라고 볼 수 있겠다.
일단 늘리고 줄이고 하는 파드 스케일링.
그리고 버전 관리이다.
# 동작 구조
디플로이먼트가 사실 직접적으로 파드를 관리하지는 않는다.
그 중간에는 `디플-레플리카셋-파드`로 이어지는 중간 개념이 들어간다.  
![https://zerotay-blog.vercel.app/img/optimized/DskDkAyWwn-700.webp](https://zerotay-blog.vercel.app/img/optimized/DskDkAyWwn-700.webp)
이를 잘 나타내는 그림이 바로 위의 것이라고 생각한다.
보다시피 디플로이먼트는 레플리카셋을 감싸고 있다.
용례를 보면 디플로이먼트는 버전 관리, 롤백 작업 등을 수행할 수 있다는 것을 알 수 있다.
이때 각 개별 버전이 바로 각각의 레플리카셋이 된다.
그러한 레플리카셋을 관리하기 때문에 디플로이먼트로 버전 관리가 가능해지는 것이다.

그렇다면 레플리카셋이 실질적으로 파드의 수를 유지하고 관리하는 객체가 아닌가?
맞다!
그런데 어차피 디플로이먼트랑 엮여서 사용되기 때문에 그냥 디플로이먼트가 관리한다고 부르는 것이다.
## 컨트롤러 동작
![](https://zerotay-blog.vercel.app/img/optimized/Z1SSg1oVeY-700.webp)

각각의 [[컨트롤러]]가 동작하는 방식이다.
별 건 없고, 그냥 위에서 한 말의 반복이다.
디플로이먼트는 사용자가 원하는 상태에 맞춰 레플리카셋을 만든다.
레플리카셋은 디플로이먼트가 정한 조건에 맞춰 파드를 만든다.
# 본격 조작 방법
내용이 많은 관계로, 따로 나누어서 작성하도록 한다.
[[디플로이먼트 조작]]
# 상태
디플로이먼트는 크게 세 가지 상태를 가진다.
## progressing
- 어떤 조작이 가해져서 그 과정이 진행 중
- 가령 새로운 버전으로 업데이트하는 과정 중에는 이 상태가 된다.
- 이 상태에서 `rollout status`를 하면 터미널 입력을 붙잡고 있는다.
## completed
다음의 특징을 가질 때 완료됐다고 간주된다.
- 희망 상태로 모든 파드가 돌아가고 있다.
- 해당 파드들이 전부 이용가능하다.
- 다른 레플리카셋은 돌아가고 있지 않다.
## failed 
다양한 요인으로 인해 디플로이먼트는 실패할 수 있다.
만약 `progressDeadlineSeconds`의 시간이 지나는 동안 completed가 되지 않는다면 해당 디플로이먼트의 상태는 failed로 간주된다.  
![](https://zerotay-blog.vercel.app/img/optimized/110cm572rk-700.webp)

잘못된 이미지를 받다가 에러가 난 모습이다.
수습하고 다시 적용하면 금새 사라진다.
근데 이전에 에러가 났던 이력을 확인할 수 있는 방법이 마땅치 않다.
애초에 이런 에러가 났더라는 것을 사람이 알기 보다는 쿠버네티스가 알아서 대처하도록 하는 게 좋을테니, 납득은 어느 정도 된다.

![](https://zerotay-blog.vercel.app/img/optimized/gpuF0-tDfb-531.webp)

근데 실패라고 해서 뭘 해주지는 않는다 ㅋㅋ
상위 컨트롤러를 통해서 이러한 상황에 대해 롤백을 하라던가, 리소스를 적게 주라던가 하는 설정을 해야 한다고 문서에서 언급하고 있다.

`kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'`
참고로 공식 문서에서는 이런 거창한 방법으로 시간을 지정할 수 있다는데..
그냥 edit해서 추가해도 되고~ 방법은 많다.

또한 [[디플로이먼트 조작#롤아웃 중단과 재개]]에서 보이는 중단을 걸게 되면 해당 타이머는 작동하지 않는다.
어찌 보면 중단을 하겠다해서 했는데 오래 걸렸다고 실패로 간주하는 것도 웃기다.
# 스펙 작성법

# 참고
- 문서 
	- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- 그림
	- https://subicura.com/k8s/guide/deployment.html#deployment-%E1%84%86%E1%85%A1%E1%86%AB%E1%84%83%E1%85%B3%E1%86%AF%E1%84%80%E1%85%B5