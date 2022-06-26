# 상호 TLS(Mutual TLS)
서비스 메시의 주요 기능 중 하나는 애플리케이션에 추가 보안을 제공하는 기능입니다. 이 작업은 다음 몇 가지 랩에서 살펴볼 몇 가지 다른 방식으로 수행합니다. 첫 번째는 "Mutual TLS" 또는 줄여서 mTLS라고 알려진 개념입니다.

마이크로 서비스 애플리케이션을 배포하고 서비스 간에 많은 PII가 흐를 것으로 예상하는 시나리오를 상상해 보십시오. 이 시나리오에는 준수해야 할 모든 종류의 보안 요구사항이 있을 수 있습니다. 개발/운영 팀에 큰 충격을 줄 수 있는 일반적인 요구사항 중 하나는 모든 서비스 통신을 암호화하는 것입니다. 즉, SSL 키를 관리 및 교환하고, 네트워크 트래픽을 검증 및 인증하고, 암호화 및 암호 해독하고, 각 애플리케이션 스택(Node.js, Java, Go 등)에서 이를 수행하는 방법을 파악해야 합니다.

## 암호화가 없는 현 상태
현재 이전 실습에서 배포한 서비스는 표준 OpenShift 네트워킹/라우팅을 통해 외부 세계의 접근로부터 보호됩니다. 기본적으로 대부분의 서비스에 대한 인그레스 경로가 없습니다(인그레스 게이트웨이를 통해 제어됨). 그러나 프로젝트에서 실행 중인 불량 파드는 우리 데이터를 스누핑할 수 있으며 다른 서비스에 직접 HTTP 요청을 할 수도 있습니다.

<p><i class="fa fa-info-circle"></i> 아직 게시판 서비스의 공유 보드에 아무 것도 추가하지 않은 경우, 생성한 공유 보드에 일부 item을 추가해야 합니다. </p>

<blockquote>
<i class="fa fa-terminal"></i> 다음과 같이 CLI를 사용하여 작업을 시작하겠습니다.
</blockquote>

```execute
curl http://boards.%username%:8080/shareditems | jq
```

이 작업은 현재 공유 보드 목록에 대한 직접 HTTP curl 요청을 실행합니다. 다음과 비슷한 내용이 출력됩니다.

```json
[
  {
    "_id": "5e5d75b33396fe0043f63e5c",
    "owner": "anonymous",
    "type": "string",
    "raw": "something goes here",
    "name": "",
    "id": "MNCkr3mK",
    "created_at": "2020-03-02T21:08:03+00:00"
  },
  {
    "_id": "5e5d75b63396fe0043f63e5d",
    "owner": "anonymous",
    "type": "string",
    "raw": "another item",
    "name": "",
    "id": "x5ORoJu8",
    "created_at": "2020-03-02T21:08:06+00:00"
  }
]
```

<br>

## 기존 서비스에 mTLS 추가
이제 Service Mesh가 코드 변경 없이, 복잡한 네트워킹 업데이트 없이, 도구(예: ssh-keygen) 또는 서버를 설치/사용하지 않고 모든 트래픽을 암호화할 수 있는 방법을 보여줍니다. 우리는 우리 앱의 모든 서비스에 적용되는 정책으로 이를 수행할 것입니다.

<p>
<i class="fa fa-info-circle"></i>
서비스 메시를 사용하면 개별 서비스, 네임스페이스 및 전체 메시 레벨에서 정책을 설정할 수 있습니다(이 순서대로 적용/시행됨).
</p>

이를 수행하기 위해 이미 YAML 구성을 작성했습니다. 내용은 다음과 같습니다.


```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
```

<blockquote>
<i class="fa fa-terminal"></i> 그럼 먼저 다음 명령으로 정책을 적용합니다.
</blockquote>

```execute
oc create -f ./config/istio/peer-authentication-mtls.yaml
```

이제 각 서비스에 대해 Destination Rules를 설정하여 사이드카가 서로 통신할 때 mTLS를 사용하도록 해야 합니다. 이에 대한 YAML은 다음과 같습니다. 와일드카드 사용에 유의하세요.
```yaml
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "destinationrule-mtls-istio-mutual"
spec:
  host: "*.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

<blockquote>
<i class="fa fa-terminal"></i> 기존 Destination Rule 제거
</blockquote>

```execute
oc delete dr --all
```
<br>

<blockquote>
<i class="fa fa-terminal"></i> 다음 명령으로 새로운 Destination Rule을 적용합니다.
</blockquote>

```execute
oc create -f ./config/istio/destinationrule-mtls.yaml
```

## 앱이 mTLS를 켠 상태에서 계속 작동하는지 확인
이제 정책이 적용되었으므로 모든 서비스 간(일명 peer) 통신에 mTLS가 **필요** 합니다.

<blockquote>
<i class="fa fa-desktop"></i> 웹앱으로 이동하고 웹사이트를 몇 번 새로고침하여 마이크로서비스를 통해 일부 트래픽을 보냅니다.
</blockquote>
이 때 모든 것이 이전과 같아야 합니다. 배후에서 추가 보안이 이루어지고 있습니다.

<img src="images/app-boardslist.png" width="1024" class="screenshot"><br/>

<br>
다음으로 방금 활성화한 mTLS를 확인합니다.

