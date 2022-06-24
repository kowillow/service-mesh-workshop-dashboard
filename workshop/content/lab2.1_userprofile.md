# 마이크로서비스 구축

<blockquote>
<i class="fa fa-desktop"></i>
브라우저에서 화면 상단(헤더)에 있는 '프로필' 섹션으로 이동합니다.
</blockquote>

<p><i class="fa fa-info-circle"></i> URL을 분실한 경우 아래 명령어를 통해 검색할 수 있습니다. </p>

```execute
echo $GATEWAY_URL
```

<br>

그러면 다음과 같은 화면이 표시됩니다.

<img src="images/app-unknownuser.png" width="1024"><br/>
 *알 수 없는 프로필 페이지*

UI에 '알 수 없는 사용자'가 표시되며 이는 애플리케이션에 대한 프로필 서비스가 없기 때문입니다. 사용자 프로필을 위한 새 마이크로서비스를 구축하고 이를 서비스 메시에 추가해보도록 하겠습니다.

## 애플리케이션 코드

지금 배포할 프로필 애플리케이션은 Java로 작성되지만, 'app-ui' 및 'boards'와 같은 다른 백엔드 구성요소는 NodeJS로 작성되어 있습니다. Istio의 장점 중 하나는 실행 중인 마이크로서비스의 프로그래밍 언어에 구애받지 않는다는 것입니다.

<blockquote>
<i class="fa fa-terminal"></i>
리포지토리에서 "UserProfile" 클래스를 살펴보십시오.
</blockquote>

```execute
cat ./code/userprofile/src/main/java/org/microservices/demo/json/UserProfile.java | grep "public UserProfile(String" -A 7
```

Output:
```java
    public UserProfile(String id, String firstname, String lastname, String aboutme) {
        this.id = id;
        this.firstName = firstname;
        this.lastName = lastname;
        this.aboutMe = aboutme;
        this.createdAt = Calendar.getInstance().getTime();
    }
```

이 클래스는 '이름'과 '성' 같은 사용자에 대한 정보를 캡슐화합니다.

<br>

또한 애플리케이션은 서비스와 상호 작용하기 위해 REST API를 노출합니다.

<blockquote>
<i class="fa fa-terminal"></i>
다음으로 "UserProfileService" 클래스를 살펴봅니다.
</blockquote>

```execute
cat ./code/userprofile/src/main/java/org/microservices/demo/service/UserProfileService.java | grep "UserProfile getProfile(" -B 5
```

Output:
```java
    /**
     * return a specific profile
     * @param id
     * @return the specified profile
     */
    UserProfile getProfile(@NotBlank String id);
```

이 인터페이스에는 사용자 프로필 정보를 가져오고 설정하기 위한 REST 메서드가 포함되어 있습니다.

<br>

## 애플리케이션 빌드

이제 애플리케이션을 빌드할 준비가 되었습니다.

[BuildConfig][1]를 사용하여 애플리케이션 이미지를 빌드합니다. `BuildConfig`템플릿은 이미 생성되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
애플리케이션을 빌드하는 데 사용할 기본 이미지를 확인합니다.
</blockquote>

```execute
cat ./config/app/userprofile-build.yaml | grep -A 4 sourceStrategy
```

Output (snippet):
```yaml
      sourceStrategy:
        from:
          kind: ImageStreamTag
          name: java:11
          namespace: openshift
```

위와 같이, 빌드는 기본 Java 이미지를 사용하여 애플리케이션을 빌드합니다.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
아래처럼 git의 코드, 브랜치를 기반으로 빌드를 생성합니다.
</blockquote>

```execute
oc new-app -f ./config/app/userprofile-build.yaml \
  -p APPLICATION_NAME=userprofile \
  -p APPLICATION_CODE_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -p APPLICATION_CODE_BRANCH=workshop-stable \
  -p APP_VERSION_TAG=1.0
```

<blockquote>
<i class="fa fa-terminal"></i>
생성된 빌드를 시작합니다.
</blockquote>

```execute
oc start-build userprofile-1.0 -F
```

빌더는 소스 코드를 컴파일하고 기본 이미지를 사용하여 배포 가능한 이미지 아티팩트를 만듭니다. 결과적으로 빌드가 성공하는 것을 볼 수 있습니다.

Output (snippet):
```
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:35 min
[INFO] Finished at: 2020-02-19T21:00:22Z
[INFO] ------------------------------------------------------------------------
...
```

<br>

빌드가 완료되면 이미지는 OpenShift의 로컬 리포지토리에 저장됩니다.

<blockquote>
<i class="fa fa-terminal"></i>
이미지가 생성되었는지 확인합니다.
</blockquote>

```execute
oc get is userprofile
```

Output:
```
NAME          IMAGE REPOSITORY                                                                  TAGS     UPDATED
userprofile   image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile   1.0   3 minutes ago
```

[1]: https://docs.openshift.com/container-platform/4.10/builds/understanding-buildconfigs.html
