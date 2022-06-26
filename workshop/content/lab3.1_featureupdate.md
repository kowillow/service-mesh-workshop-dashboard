# 관찰 가능성(Observability) 파고들기

Istio는 서비스 메시와 그 성능을 분석할 수 있는 추가 기능을 제공합니다. User profile 서비스의 새 버전을 배포하고 서비스 메시에 미치는 영향을 분석해 보겠습니다.

## 기능 업데이트

새 버전의 코드는 repo의 'workshop-feature-update' 브랜치에 이미 작성되었습니다.

<blockquote>
<i class="fa fa-terminal"></i>
이 feature-update 브랜치로 새로운 빌드를 만듭니다.
</blockquote>

```execute
oc new-app -f ./config/app/userprofile-build.yaml \
  -p APPLICATION_NAME=userprofile \
  -p APPLICATION_CODE_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -p APPLICATION_CODE_BRANCH=workshop-feature-update \
  -p APP_VERSION_TAG=2.0
```

<p><i class="fa fa-info-circle"></i> 'imagestream이 이미 존재한다'는 실패 메시지는 무시하시면 됩니다.</p>

<br>

<blockquote>
<i class="fa fa-terminal"></i>
빌드 시작
</blockquote>

```execute
oc start-build userprofile-2.0 -F
```

빌드를 시작하면, 빌더는 소스 코드를 컴파일하고 베이스 이미지를 사용해서 클러스터에 배포 가능한 이미지 아티팩트를 만듭니다. 기다리면 빌드가 성공적으로 완료됩니다.

Output (snippet):
```
...
[INFO] [io.quarkus.deployment.pkg.steps.JarResultBuildStep] Building thin jar: /tmp/src/target/userprofile-1.0-SNAPSHOT-runner.jar
[INFO] [io.quarkus.deployment.QuarkusAugmentor] Quarkus augmentation completed in 7988ms
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:41 min
[INFO] Finished at: 2020-02-24T19:13:59Z
[INFO] ------------------------------------------------------------------------...
```

<br>

빌드가 완료되면 이미지는 OpenShift 로컬 리포지토리에 저장됩니다.

<blockquote>
<i class="fa fa-terminal"></i>
이미지가 생성되었는지 확인합니다.
</blockquote>

```execute
oc describe is userprofile
```

Output (snippet):
```
...
2.0
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile@sha256:147d836e9f7331a27b26723cbb99f2b667e176b4d5dd356fea947c7ca4fc24a6
      2 minutes ago

1.0
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile@sha256:f01d00409f44962ab321517e18fb06483fadfc07b2f70c088f567acf20dc65eb
      23 hours ago
```

<p><i class="fa fa-info-circle"></i> 이번에 새로 생성한 최신 이미지에는 '2.0' 태그가 있어야 합니다.</p>

<br>

<blockquote>
<i class="fa fa-terminal"></i>
로컬 이미지를 참조하도록 합니다.
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

Deployment 파일 'userprofile-deploy-v2.yaml'은 애플리케이션을 배포하기 위해 생성되었습니다.

<blockquote>
<i class="fa fa-terminal"></i>
이미지 URI를 사용하여 서비스를 배포합니다.
</blockquote>

```execute
sed "s|%USER_PROFILE_IMAGE_URI%|$USER_PROFILE_IMAGE_URI|" ./config/app/userprofile-deploy-v2.yaml | oc create -f -
```

<blockquote>
<i class="fa fa-terminal"></i>
User profile 서비스의 배포 상태 보기
</blockquote>

```execute
oc get pods -l deploymentconfig=userprofile --watch
```

Output:
```
userprofile-2-xxxxxxxxxx-xxxxx            2/2     Running        0          22s
userprofile-xxxxxxxxxx-xxxxx              2/2     Running        0          2m55s
```

<br>

## 애플리케이션 접속

브라우저에서 새 버전의 프로필 서비스를 테스트해 보겠습니다(스포일러: 일부러 버그를 추가했습니다).

<blockquote>
<i class="fa fa-desktop"></i>  
헤더의 'Profile' 섹션으로 이동합니다.
</blockquote>

<p><i class="fa fa-info-circle"></i> URL을 분실한 경우 아래 명령어를 통해 검색할 수 있습니다.</p>

```execute
echo $GATEWAY_URL
```

<br>
프로필 페이지는 버전 1과 2 사이에서 라운드 로빈(round robin)됩니다. 버전 2는 매우 느리게 로드되며 아래 화면과 같습니다.

<img src="images/app-profilepage-v2.png" width="1024"><br/>
 *Profile Page*
다음으로 Service Mesh를 사용하여 문제를 디버깅합니다.
