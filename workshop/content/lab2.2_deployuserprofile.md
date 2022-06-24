# 서비스 메시에 새 서비스 추가

방금 생성한 사용자 프로필 애플리케이션을 서비스 메시에 배포해야 합니다.

## 애플리케이션 배포

Deployment 파일 'userprofile-deploy-all.yaml'을 통해 애플리케이션을 배포할 수 있습니다. 이 파일은 사용자 프로필 서비스와 함께 PostgreSQL 데이터베이스를 생성합니다. 다른 소스 파일과 유사하게 'sidecar.istio.io/inject' 주석이 추가되어 있어서, 이를 인지한 Istio가 파드에 사이드카 프록시를 주입하고 메시에 추가합니다.

<blockquote>
<i class="fa fa-terminal"></i>
'userprofile' 파일에서 주석을 확인합니다.
</blockquote>

```execute
cat ./config/app/userprofile-deploy-all.yaml | grep -B 1 sidecar.istio.io/inject
```

Output:
```
    annotations:
      sidecar.istio.io/inject: "true"
  --
    annotations:
      sidecar.istio.io/inject: "true"
```

주석은 userprofile과 PostgreSQL 서비스에 대해 나타나므로 두 번 표시됩니다.

<br>
<br>

서비스를 배포하기 전에 이전 실습에서 구축한 로컬 이미지에 대한 참조가 필요합니다.

<blockquote>
<i class="fa fa-terminal"></i>
다음 명령을 실행합니다.
</blockquote>

```execute
USER_PROFILE_IMAGE_URI=$(oc get is userprofile --template='{{.status.dockerImageRepository}}')
echo $USER_PROFILE_IMAGE_URI
```

Output (sample):
```
image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile
```

<br>
<br>

<blockquote>
<i class="fa fa-terminal"></i>
다음 이미지 URI를 사용하여 서비스를 배포합니다.
</blockquote>

```execute
sed "s|%USER_PROFILE_IMAGE_URI%|$USER_PROFILE_IMAGE_URI|" ./config/app/userprofile-deploy-all.yaml | oc create -f -
```

<br>
<br>

<blockquote>
<i class="fa fa-terminal"></i>
사용자 프로필 앱 배포 상태 확인
</blockquote>

```execute
oc get pods -l deploymentconfig=userprofile --watch
```

<p>
<i class="fa fa-info-circle"></i>
PostgreSQL 파드가 아직 실행되지 않았으면 userprofile 서비스에 오류가 발생하고 다시 시작될 수 있습니다.
</p>

Output:
```
userprofile-xxxxxxxxxx-xxxxx              2/2     Running		    0          2m55s
```

<br>

다른 마이크로서비스와 마찬가지로, 사용자 프로필 서비스는 애플리케이션과 함께 Istio 프록시를 실행합니다.

<blockquote>
<i class="fa fa-terminal"></i>
'userprofile' 파드에 포함된 컨테이너의 이름을 출력합니다.
</blockquote>


```execute
oc get pods -l deploymentconfig=userprofile -o jsonpath='{.items[*].spec.containers[*].name}{"\n"}'
```

Output:
```
userprofile istio-proxy
```

<br>

## 애플리케이션 접속

사용자 프로필 서비스가 배포되었습니다! 브라우저에서 이것을 테스트해 봅시다.

<blockquote>
<i class="fa fa-desktop"></i>
웹페이지 헤더의 'Profile' 섹션으로 이동합니다.
</blockquote>

<p><i class="fa fa-info-circle"></i> URL을 분실한 경우 다음을 통해 검색할 수 있습니다.</p>

```execute
echo $GATEWAY_URL
```

<br>

다음과 같은 화면이 표시되어야 합니다.

<img src="images/app-profilepage.png" width="1024"><br/>
 *프로필 페이지*
