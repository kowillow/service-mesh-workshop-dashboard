# 수신 및 송신 보안

엄격한 보안이 필요한 경우 수신(ingress) 및 송신(egress) 트래픽 보안에 대한 세부 사항을 구성해야 합니다. 송신과 관련한 보안은 종종 네트워크 외부의 잠재적으로 유해한 리소스에 대한 접근을 차단하고 거부하는 데 사용됩니다. 또한 클러스터에서 악의적인 활동이 발생하지 않도록 하는 것에 좋습니다.

기본 수신 보안 개념에는 이미 익숙할 것입니다. 일반적으로 특정 서비스만 클러스터 외부에서 액세스할 수 있도록 노출하고 기본 TLS/SSL을 사용합니다. 서비스 메시에는 기본적으로 실행되는 인그레스 라우터(standalone Envoy)가 있으며 우리는 "서비스 메시에 앱 배포" 실습에서 이미 이것을 구성해두었습니다.

마이크로 서비스의 인바운드 트래픽을 추적/잠금하는 훨씬 더 좋은 방법은 모든 API 서비스 앞에서 'API 관리' 도구를 활용하는 것입니다. 3scale용 서비스 메시 플러그인으로 이 작업을 수행할 수 있지만, 오늘은 설명하지 않겠습니다. 관심 있으시다면 [여기][7], [그리고 여기][8]에서 자세히 읽어 보세요.

그럼 일단 밖으로 나가는 송신 트래픽을 막도록 하겠습니다.

## Egress 잠그기
이 예시에서는 외부 엔드포인트에 대한 접근을 승인된 호스트로만 제한하려고 합니다. 서비스 메시에는 기본적으로 실행 중인 이그레스 라우터(standalone Envoy)가 있으며 이를 구성하기만 하면 됩니다.

Istio를 설치하는 데 사용한 `ServiceMeshControlPlane` 사용자 지정 리소스에는 송신 보안에 대한 기본값을 제어할 수 있는 `global` 구성이 있습니다. 이는 다음과 같습니다.

```
# Set the default behavior of the sidecar for handling outbound traffic from the application:
# ALLOW_ANY - outbound traffic to unknown destinations will be allowed, in case there are no
#   services or ServiceEntries for the destination port
# REGISTRY_ONLY - restrict outbound traffic to services defined in the service registry as well
#   as those defined through ServiceEntries  
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
```

이 워크숍에서 사용 중인 메시는 이미 모드가 `REGISTRY_ONLY`로 설정되어 있습니다. 자동 생성된 config map에서 다음을 확인합니다.

<blockquote>
<i class="fa fa-terminal"></i>
다음 명령을 실행하고 outboundTrafficPolicy를 찾습니다.
</blockquote>

```execute
oc describe cm/istio-workshop-install -n %username%-istio | grep outboundTrafficPolicy -A 1
```

Output:

```
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
```

<blockquote>
<i class="fa fa-terminal"></i>
이제 이 정책이 작동 중인지 확인하겠습니다. 데이터를 스크랩하려면 다음 명령을 실행합니다.
</blockquote>

```execute
curl context-scraper.%username%:8080/scrape/custom_search?term==skynet | jq
```

ECONNRESET 오류와 함께 아래와 유사한 출력을 얻어야 합니다.

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   682  100   682    0     0  42590      0 --:--:-- --:--:-- --:--:-- 45466
{
  "oops": {
    "name": "RequestError",
    "message": "Error: read ECONNRESET",
    "cause": {
      "errno": "ECONNRESET",
      "code": "ECONNRESET",
      "syscall": "read"
    },
    "error": {
      "errno": "ECONNRESET",
      "code": "ECONNRESET",
      "syscall": "read"
    },
    "options": {
      "method": "GET",
      "uri": "https://www.googleapis.com/customsearch/v1/siterestrict",
      "qs": {
        "key": "AIzaSyDRdgirA2Pakl4PMi7t-8LFfnnjEFHnbY4",
        "cx": "005627457786250373845:lwanzyzfwji",
        "q": "=skynet"
      },
      "headers": {
        "user-agent": "curl/7.29.0",
        "x-request-id": "07ce0cef-5b07-9ee7-8565-a9705bfd90b1",
        "x-b3-traceid": "663d4c9ff4336678433857d48c7a8b5b",
        "x-b3-spanid": "433857d48c7a8b5b",
        "x-b3-sampled": "1"
      },
      "json": true,
      "simple": true,
      "resolveWithFullResponse": false,
      "transform2xxOnly": false
    }
  }
}
```

이것은 메시에 있는 컨텍스트 스크래퍼(context-scraper) 마이크로 서비스와 통신하려고 하는 curl 입니다. 오류 세부 정보를 통해, 서비스가 googleapis.com으로 나가려고 하지만 실패한다는 것을 알 수 있습니다.

<br>

<blockquote>
<i class="fa fa-desktop"></i>
(선택 사항) Kiali의 Graph 보기를 열면 다음과 같이 실패하는 것을 볼 수 있습니다.
</blockquote>

<img src="images/kiali-egress-fail.png" width="1024" class="screenshot"><br/>

<br>

## 승인된 호스트로의 송신 허용
이제 [ServiceEntry][5] 유형의 Istio API 개체를 사용하여 제어된 송신을 허용합니다. 이 ServiceEntry를 추가하면 우리의 마이크로 서비스가(Envoy 사이드카를 통해) 메시의 서비스인 것처럼 지정된 외부 서비스로 트래픽을 보낼 수 있습니다.

ServiceEntry는 다음과 같습니다.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: context-scraper-egress
spec:
  exportTo:
  - "."
  hosts:
  - www.googleapis.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

<br>

<blockquote>
<i class="fa fa-terminal"></i>
다음 명령으로 적용합니다.
</blockquote>

```execute
oc apply -f ./config/istio/serviceentry-googleapis.yaml
```

<br>

<blockquote>
<i class="fa fa-terminal"></i>
일부 데이터를 다시 스크랩해 보세요.
</blockquote>

```execute
curl context-scraper.%username%:8080/scrape/custom_search?term==skynet | jq
```

이제 아래와 유사한 결과를 보여야 합니다.

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  5197  100  5197    0     0    865      0  0:00:06  0:00:06 --:--:--  1157
[
  {
    "link": "https://www.reddit.com/r/DeepBrainChain/comments/8rar5b/deepbrain_chain_launches_skynet_project/",
    "thumbnail": [
      {
        "src": "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQrwBIFnOZygv1EKJ1z7TnNeWIfom8Kp0Zgbs6YvM4
DXYP0zvra6GNCnh0",
        "width": "225",
        "height": "225"
      }
    ],
    "title": "DeepBrain Chain Launches ''Skynet Project''— Recruiting AI ...",
    "snippet": "Nov 29, 2017 ... To achieve that, DeepBrain Chain Foundation will activate the ''Skynet Project
'' \non June 15th Beijing time and, recruit AI computing power from ..."
  },
  {
    "link": "https://www.reddit.com/r/Terminator/comments/eo6qgf/what_would_happen_if_skynet_wins_would_they_bu
ild/",
    "thumbnail": [
      {
        "src": "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRq6RYcGTf2e6W8pH3h962NB0_0DNTi93EeqEdtktl
mLoALKvdA1rUY_AU",
        "width": "132",
        "height": "92"
      }
    ],
    "title": "What would happen if Skynet wins? Would they build a world for ...",
    "snippet": "Skynet is an accident and many ways an unstable AI, which makes it different \nfrom the termina
tors it created. We get a hint of this in TSCC of a second machine\n ..."
  },
  ...
```

<br>

원하는 경우 이제 Virtual Service 및 Destination Rule을 구성하여 ServiceEntry에 대한 트래픽을 보다 세분화된 방식으로 제어할 수 있습니다. 메시의 다른 서비스에 대한 트래픽을 구성한 것과 동일한 방법입니다.

그리고 Kiali는 이 ServiceEntry도 추적합니다. 그래프를 보고 어떻게 시각화되는지 살펴보겠습니다.

<br>

<blockquote>
<i class="fa fa-desktop"></i>
(선택 사항) 이제 Kiali의 그래프 보기를 열면 외부 서비스가 표시되는 것을 볼 수 있습니다.
<br>
</blockquote>

<img src="images/kiali-egress.png" width="1024" class="screenshot"><br/>

<br>

# Summary
축하합니다! 알려진 외부 호스트에서만 송신이 가능하도록 설정하고, 서비스 메시를 통해 이에 대한 추적을 구성했습니다. 이것은 메시의 고급 기능입니다. [ingress는 이쪽에서][1], 그리고 [egress는 이쪽에서][2] 관리하는 방법에 대한 자세한 내용을 읽을 수 있습니다.

몇 가지 주요 사항은 다음과 같습니다.

* 승인된 외부 엔드포인트 또는 서비스에 대한 송신 트래픽만 허용할 수 있습니다.
* 승인된 외부 엔드포인트 또는 서비스에 대한 수신 트래픽만 허용할 수 있습니다.
* 표준 TLS 또는 상호 TLS를 통해 수신 트래픽을 보호할 수 있습니다.
* 3scale API 관리 도구를 활용하면 노출된 API에 액세스할 수 있는 사용자와 방법을 제한할 수 있습니다. (이번 워크샵에서는 생략됨)


[1]: https://istio.io/docs/tasks/traffic-management/ingress/
[2]: https://istio.io/docs/tasks/traffic-management/egress/
[3]: https://istio.io/docs/ops/best-practices/security/
[4]: https://istio.io/docs/tasks/traffic-management/ingress/secure-ingress-mount/
[5]: https://istio.io/docs/concepts/traffic-management/#service-entries
[6]: https://archive.istio.io/v1.4/docs/tasks/traffic-management/egress/egress-control/
[7]: https://docs.openshift.com/container-platform/4.3/service_mesh/threescale_adapter/threescale-adapter.html
[8]: https://www.redhat.com/en/technologies/jboss-middleware/3scale
