# 설치

OpenShift 클러스터에서 왼쪽 탭에 있는 실습들을 수행합니다. 먼저 콘솔과 CLI를 통해 클러스터에 접근할 수 있는지 테스트합니다.

## OpenShift

<blockquote>
<i class="fa fa-desktop"></i> 대시보드에서 콘솔 탭을 확인하면 다음과 같은 화면이 표시되어야 합니다.
</blockquote>

<img src="images/openshift-welcome.png" width="1024"><br/>
 *OpenShift Welcome*

<br>

You will use the OpenShift `oc` CLI  to execute commands for the majority of this lab.  
이번 실습의 대부분에서 OpenShift `oc` CLI를 사용해서 명령을 실행합니다.

<blockquote>
<i class="fa fa-terminal"></i> 웹 터미널에서 클러스터에 로그인되어 있어야 합니다.
</blockquote>

**터미널** 탭으로 전환하고 다음을 실행해보세요. 

```execute
oc whoami
```
*코드 블록의 오른쪽 상단 모서리에 있는 재생 버튼을 클릭하면 자동으로 명령을 실행할 수 있습니다.*

사용자 이름(%username%)이 표시되어야 합니다.

실습에서 사용하실 프로젝트는 미리 구성해두었습니다.

<blockquote>
<i class="fa fa-terminal"></i> 프로젝트 목록 확인:
</blockquote>

```execute
oc projects
```

사용자 프로젝트(예: '%username%')와 '%username%-istio', 두 개의 프로젝트가 표시되어야 합니다.

<br>

<blockquote>
<i class="fa fa-terminal"></i> 다음 명령어를 통해 사용자 프로젝트로 전환합니다. 
</blockquote>

```execute
oc project %username%
```

<br>

프로젝트를 살펴보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i> 프로젝트의 파드를 나열합니다.
</blockquote>

```execute
oc get pods
```

Output (sample):

```
NAME                                    READY   STATUS    RESTARTS   AGE
rhsso-operator-xxxxxxxxx-xxxxx          1/1     Running   0          15h
```

RH-SSO 오퍼레이터는 나중에 보안 설정 실습에서 사용됩니다.

<br>

## 애플리케이션 코드
다음으로 애플리케이션 코드의 로컬 복사본이 필요합니다.

<blockquote>
<i class="fa fa-terminal"></i> 리포지토리를 복제합니다.
</blockquote>

```execute
git clone https://github.com/RedHatGov/service-mesh-workshop-code.git
```

<blockquote>
<i class="fa fa-terminal"></i> Workshop-stable 브랜치를 확인합니다.
</blockquote>

```execute
cd service-mesh-workshop-code && git checkout workshop-stable
```

## Istio
Istio는 클러스터에 미리 설치된 상태입니다. 클러스터에서 실행 중인지 확인합니다.

%username%-istio 프로젝트는 당신 전용 서비스 메시입니다.

<blockquote>
<i class="fa fa-terminal"></i> 서비스 메시 프로젝트의 파드를 나열합니다.:
</blockquote>

```execute
oc get pods -n %username%-istio
```

Output:

```
NAME                                      READY   STATUS    RESTARTS   AGE
grafana-xxxxxxxxx-xxxxx                   2/2     Running   0          5h30m
istio-egressgateway-xxxxxxxx-xxxxx        1/1     Running   0          5h30m
istio-ingressgateway-xxxxxxxxx-xxxxx      1/1     Running   0          5h30m
istio-telemetry-xxxxxxxxx-xxxxx           2/2     Running   0          5h25m
istiod-workshop-install-xxxxxxxxx-xxxxx   1/1     Running   0          5m28s
jaeger-xxxxxxxxxx-xxxxx                   2/2     Running   0          5h25m
kiali-xxxxxxxxxx-xxxxx                    1/1     Running   0          5h25m
prometheus-xxxxxxxxx-xxxxx                2/2     Running   0          5h30m
```

The primary control plane component is the Istio daemon `istiod`.  `istiod` handles [Traffic Management][1], [Telemetry][2], and [Security][3].  The `istio-ingressgateway` is a load balancer for your service mesh.  You will configure this with a microservices application in the next lab.

Istio 컨트롤 플레인의 기본 구성 요소는 Istio 데몬인 istiod입니다. `istiod`는 [트래픽 관리][1], [원격 측정(Telemetry)][2] 및 [보안][3]을 처리합니다. 
`istio-ingressgateway`는 서비스 메시용 로드 밸런서입니다. 다음 실습에서는 마이크로서비스 애플리케이션과 함께 이를 구성합니다.

[1]: https://istio.io/docs/concepts/traffic-management/
[2]: https://istio.io/docs/concepts/observability/
[3]: https://istio.io/docs/concepts/security/
