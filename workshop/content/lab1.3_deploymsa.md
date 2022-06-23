# 서비스 메시에 앱 배포

이제 마이크로서비스 애플리케이션을 배포할 시간입니다. 이 애플리케이션은 사용자가 공유 게시판에 댓글을 게시할 수 있는 게시판 앱입니다. 
다음은 아키텍처 다이어그램입니다.

<img src="images/architecture-highlevel.png" width="800"><br/>

*앱 아키텍처(그림)*

마이크로서비스에는 SSO(Single Sign-On), UI(사용자 인터페이스), 게시판 애플리케이션 및 컨텍스트 스크레퍼(context scraper)가 포함됩니다. 
이 시나리오에서는 이러한 서비스들을 배포한 다음, 새 사용자 프로필 서비스를 추가합니다.

<br>

## 마이크로서비스 배포

소스 코드에서 애플리케이션 이미지를 빌드한 다음, 클러스터에 리소스를 배포해야 합니다.

소스 파일의 레이블은 '{microservice}-fromsource.yaml'입니다. 각 파일에 'sidecar.istio.io/inject' 주석이 추가되어 Service Mesh가 사이드카 프록시를 삽입하도록 설정했습니다.

<blockquote>
<i class="fa fa-terminal"></i>
'app-ui' 파일에서 주석을 확인합니다.
</blockquote>

```execute
cat config/app/app-ui-fromsource.yaml | grep -B 1 sidecar.istio.io/inject
```

Output:
```
	annotations:
	  sidecar.istio.io/inject: "true"
```

<br>

이제 마이크로서비스를 배포해 보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i>
게시판 서비스를 배포합니다. 
</blockquote>

```execute
oc new-app -f ./config/app/boards-fromsource.yaml \
  -p APPLICATION_NAME=boards \
  -p NODEJS_VERSION_TAG=16-ubi8 \
  -p GIT_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -p GIT_BRANCH=workshop-stable \
  -p DATABASE_SERVICE_NAME=boards-mongodb \
  -p MONGODB_DATABASE=boardsDevelopment
```

<blockquote>
<i class="fa fa-terminal"></i>
컨텍스트 스크래퍼 서비스를 배포합니다.
</blockquote>

```execute
oc new-app -f ./config/app/context-scraper-fromsource.yaml \
  -p APPLICATION_NAME=context-scraper \
  -p NODEJS_VERSION_TAG=16-ubi8 \
  -p GIT_BRANCH=workshop-stable \
  -p GIT_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git
```

<blockquote>
<i class="fa fa-terminal"></i>
사용자 인터페이스를 배포합니다.
</blockquote>

```execute
oc new-app -f ./config/app/app-ui-fromsource.yaml \
  -p APPLICATION_NAME=app-ui \
  -p NODEJS_VERSION_TAG=16-ubi8 \
  -p GIT_BRANCH=workshop-stable \
  -p GIT_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -e FAKE_USER=true
```

<blockquote>
<i class="fa fa-terminal"></i>
마이크로서비스 설치 확인하기
</blockquote>

```execute
oc get pods --watch
```

<br>

몇 분간 기다립니다. 'app-ui', 'boards' 및 'context-scraper' 파드가 실행 중(running) 상태로 표시되어야 합니다. 
예를 들어 아래와 같이 출력되면 파드들이 모두 정상적으로 실행 중인 것을 알 수 있습니다.

```
NAME                                    READY   STATUS      RESTARTS   AGE
app-ui-1-build                          0/1     Completed   0          64m
app-ui-1-xxxxx                          2/2     Running     0          62m
app-ui-1-deploy                         0/1     Completed   0          62m
boards-1-xxxxx                          2/2     Running     0          62m
boards-1-build                          0/1     Completed   0          64m
boards-1-deploy                         0/1     Completed   0          62m
boards-mongodb-1-xxxxx                  2/2     Running     0          64m
boards-mongodb-1-deploy                 0/1     Completed   0          64m
context-scraper-1-build                 0/1     Completed   0          64m
context-scraper-1-xxxxx                 2/2     Running     0          62m
context-scraper-1-deploy                0/1     Completed   0          62m
rhsso-operator-xxxxxxxxx-xxxxx          1/1     Running     0          15h
```

<br>

각 마이크로서비스 파드는 두 개의 컨테이너(애플리케이션 자체와 Istio 프록시)를 실행합니다.

<blockquote>
<i class="fa fa-terminal"></i>
아래 명령어를 통해 'app-ui' 파드의 컨테이너 이름을 표시합니다. app-ui와 istio-proxy, 두 개의 컨테이너가 실행 중임을 확인할 수 있습니다.
</blockquote>

```execute
oc get pods -l app=app-ui -o jsonpath='{.items[*].spec.containers[*].name}{"\n"}'
```

Output:
```
app-ui istio-proxy
```

<br>

## 애플리케이션에 접속

애플리케이션이 배포되었습니다! 하지만 사용자 인터페이스를 통해 애플리케이션에 액세스하는 방법이 필요합니다.

Istio는 서비스 메시의 엣지에서 로드 밸런서를 구성할 수 있는 [게이트웨이][1] 리소스를 제공합니다. 다음 단계는 이 게이트웨이 리소스를 배포하고 애플리케이션 사용자 인터페이스로 라우팅하도록 로드 밸런서를 구성하는 것입니다.

<blockquote>
<i class="fa fa-terminal"></i>
게이트웨이 구성(config) 및 라우팅 규칙을 만듭니다.
</blockquote>

```execute
oc create -f ./config/istio/gateway.yaml
```

애플리케이션에 접속하려면 로드 밸런서의 엔드포인트가 필요합니다.

<blockquote>
<i class="fa fa-terminal"></i>
로드 밸런서의 URL을 출력합니다.
</blockquote>

```execute
GATEWAY_URL=$(oc get route istio-ingressgateway -n %username%-istio --template='http://{{.spec.host}}')
echo $GATEWAY_URL
```

<blockquote>
<i class="fa fa-desktop"></i>
새 브라우저 탭에서 이 URL로 이동합니다. 예를 들어 아래와 같은 형식의 주소입니다.
</blockquote>

```
http://istio-ingressgateway-userx-istio.apps.cluster-naa-xxxx.naa-xxxx.example.opentlc.com:6443
```

<br>

그러면 아래와 같은 애플리케이션 사용자 인터페이스가 표시되어야 합니다. 새로운 공유 게시판을 만들고 글을 게시해 보세요.

<img src="images/app-pasteboard.png" width="1024"><br/>
 *새로운 게시판 생성*

## 요약

축하합니다. 마이크로서비스 애플리케이션을 배포했습니다!

주요 사항은 다음과 같습니다.

* 마이크로서비스 데모 애플리케이션은 paste board 애플리케이션입니다.
* 'sidecar.istio.io/inject' 주석을 입력해주면 Istio가 사이드카 프록시를 마이크로서비스 파드에 삽입합니다.
* 게이트웨이 리소스는 서비스 메시에 대한 인바운드 연결을 허용하도록 엣지 로드 밸런서를 구성합니다.

[1]: https://istio.io/docs/reference/config/networking/gateway/
