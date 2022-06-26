# 싱글 사인온(Single Sign-On)
로그인, 등록 및 역할 기반 인증은 Red Hat SSO(일명 Keycloak)를 통해 처리할 수 있습니다.
SSO는 OpenShift 또는 Red Hat의 미들웨어 제품 서브스크립션의 일부로 포함되어 있습니다.

## 설치
서비스 메시는 SSO를 활용하여 사용자를 인증하고 JWT를 생성하므로 이를 설치해야 합니다. 현재 제공된 환경에는 Service Mesh와 마찬가지로, Operator를 통해 네임스페이스에 미리 설치되어 있습니다. (%username% 네임스페이스)

### 리소스 커스터마이징
이 워크샵에서 사용하는 SSO 인스턴스 및 설정은 보안 설정을 완화했으며(와일드카드 사용 등), 실제 프로덕션 환경에서는 SSO 보안 설정을 강화해야 합니다. 


### SSO 인스턴스 생성
<blockquote>
<i class="fa fa-terminal"></i> Kubernetes 리소스를 사용하여 Keycloak 인스턴스를 생성합니다.
</blockquote>

```execute
oc apply -f ./config/sso/sso-keycloak.yaml
```

<blockquote>
<i class="fa fa-terminal"></i> keycloak 파드가 준비될 때까지 몇 분 정도 기다립니다.
</blockquote>

```execute
oc wait --for=condition=Ready pod/keycloak-0 --timeout=300s
```

Output:
```
pod/keycloak-0 condition met
```

라우트를 허용하도록 keycloak 파드에 레이블을 지정합니다.

```execute
oc label pod keycloak-0 maistra.io/expose-route=true
```

<blockquote>
<i class="fa fa-terminal"></i> 다음 명령어를 통해 SSO의 realm, roles, clients, users를 생성합니다.
</blockquote>

```execute
sed "s|%APP_URL%|$GATEWAY_URL|" ./config/sso/sso-realm.yaml | oc create -f -
```

```execute
oc apply -f ./config/sso/sso-user1.yaml
```

```execute
oc apply -f ./config/sso/sso-user2.yaml
```

### SSO 관리 콘솔에 로그인
<blockquote>
<i class="fa fa-terminal"></i>
엔드포인트를 검색하고 SSO 콘솔을 엽니다.
</blockquote>

```execute
echo $(oc get route keycloak --template='https://{{.spec.host}}')
```

<p>
<i class="fa fa-info-circle"></i>
keycloak은 파드 실행 후에도 잠시 동안 계속 초기화되므로 바로 접속이 불가할 수도 있습니다.
파드 로그를 보게 되면 "Admin console listening on"이라는 줄이 표시되며 로그인할 준비가 되었음을 나타냅니다.
</p>
<br>

관리자 유저이름은 `admin`이며, 암호는 자동으로 생성되어 `credential-workshop-keycloak`이라는 시크릿에 base64-encoded 형식으로 저장됩니다.

<blockquote>
<i class="fa fa-terminal"></i>
다음을 실행하여 암호를 출력
</blockquote>

```execute
echo $(oc get secret/credential-workshop-keycloak -o jsonpath="{.data.ADMIN_PASSWORD}") | base64 --decode && echo
```

<blockquote>
<i class="fa fa-desktop"></i> SSO 웹 콘솔로 이동하여 "Administration Console"을 선택하고 방금 출력한 비밀번호를 사용해 "admin"으로 로그인합니다.
</blockquote>

<br>

### SSO 웹 콘솔로 설정 편집
<blockquote>
<i class="fa fa-desktop"></i> 이제 로그인했으므로 왼쪽 메뉴에서 "Users"를 선택하고, 다음엔 "View all users" 아이콘을 클릭합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> "demo" 사용자를 선택하고 "Credentials"로 이동합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 암호를 "demo"로 설정하거나 재설정합니다(Temporary 체크란이 "off"로 설정되어 있는지 확인).
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 사용자 "theterminator"에 대해 동일한 작업을 반복하되 암호는 "illbeback"으로 설정합니다.
</blockquote>

<br>

### 이제 SSO 웹 콘솔에서 사용자에 대한 몇 가지 역할을 확인합니다.


<blockquote>
<i class="fa fa-desktop"></i> 사용자 "theterminator"가 선택된 상태에서 "Role Mappings"를 클릭합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> "cool-kids" 역할이 사전에 부여되어 있음을 확인할 수 있습니다.
</blockquote>


*만약 목록에 cool-kids가 없으면*

<blockquote>
<i class="fa fa-desktop"></i> 왼쪽 사이드바에서 "Roles"를 선택하고, 다음은 "View all roles"를 선택합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 화면 오른쪽에서  "Add Role"을 클릭합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> "cool-kids"와 설명을 입력한 다음 "Save"를 클릭합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 이제 "Users"로 돌아가서 "terminator"를 선택하면 역할 매핑을 수행할 수 있습니다.
</blockquote>


<br>

## 이 SSO 서비스를 사용하도록 APP UI에 지시
이전 실습까지 우리는 SSO를 사용하지 않고 앱을 테스트하기 위해 가짜 로그인을 했습니다. 이제 그것을 제거하고 app-ui 서비스를 재배포합니다.

<blockquote>
<i class="fa fa-terminal"></i>
CLI에서 다음을 실행합니다.
</blockquote>

```execute
SSO_SVC=$(oc get route keycloak --template='{{.spec.host}}')
oc set env dc/app-ui NODE_TLS_REJECT_UNAUTHORIZED=0 FAKE_USER=false SSO_SVC_HOST=$SSO_SVC
```

<br/>

## 접속 정보 / API 문서
- 관리자는 https://keycloak-%username%.apps.%cluster_subdomain%/ 에서 로그인할 수 있습니다.
- 유저는 https://keycloak-%username%.apps.%cluster_subdomain%/auth/realms/microservices/account 에서 로그인할 수 있습니다.
- keycloak에 대한 자세한 내용은 아래 리소스를 확인하십시오.
  - [공식 문서][1]
  - [업스트림 문서][2]
  - [블로그 예제][3]

<br/>

[1]: https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.3/html-single/red_hat_single_sign-on_for_openshift/
[2]: https://www.keycloak.org/documentation.html
[3]: https://developers.redhat.com/blog/2020/01/29/api-login-and-jwt-token-generation-using-keycloak/
