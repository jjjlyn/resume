# 스타트업 소규모 서비스 배포 경험기
- 신규 스타트업에서 서비스를 처음 배포하기까지의 전 과정을 공유합니다. 
- AWS(+ Azure) VM + RDS + Azure DevOps + Docker를 이용하여 CI/CD를 구성한 경험을 정리합니다.

## 목차
- [서버 구성](#서버-구성)
  - [운영 환경](#운영-환경)
  - [개발 환경](#개발-환경)
- [운영 중 이슈](#운영-중-이슈)

## 서버 구성
### 운영 환경

:one: 전체 구성 및 흐름

**MSA + Nginx + Docker**

<!-- <details>
<summary>(여담)</summary>
<div markdown="1">

</div>
</details> -->

전체 구성 및 흐름은 개략적으로 아래와 같습니다.

![infra-architecture](https://user-images.githubusercontent.com/43571225/170717475-8349481f-520d-45e6-8372-212b80fcc67c.png)
![prod-structure](https://user-images.githubusercontent.com/43571225/171210864-90c8c808-d1c7-413f-8fca-d628952e0a09.png)

- 서비스 사용자가 클라이언트 어플리케이션(e.g. 안드로이드 앱)으로 네트워크 통신을 합니다. 
- `https://api.x.y/account/**` 등의 도메인 주소로 자원을 요청하게 되겠죠.
- `api.x.y/**` -> 서비스(VM) 외부 IP 주소 -> Nginx에서 `api.x.y를 받으면 http://127.0.0.1:{지정포트}로 포트 포워딩 한다`라는 환경설정의 명시에 따라 서버 요청을 내부의 해당 포트로 전달합니다.(`/etc/nginx/nginx.conf`에서 사전 설정을 해주어야 합니다).

사실 도커에 컨테이너를 올리고 난 뒤, 가상머신 보안그룹 인바운드 규칙에서 각 컨테이너의 지정 포트를 열어주면 간단하게 끝날 일입니다. 그러나 실사용 서비스에는 예의(보안!!!)를 지켜야 합니다.<br>
Nginx의 포트포워딩을 사용하면 내부망의 서버 정보를 외부로부터 은닉할 수 있습니다. 또한 Nginx는 리버스 프록시로서 내부 서버의 정보를 모두 알고 있기 때문에 서버 부하 정도에 따라 자원을 분배하여 요청하는 로드 밸런서의 기능도 수행합니다 :)

이 흐름대로 가려면 아래의 과정이 필요합니다.
  1. [가상머신 도커 컨테이너 실행](#가상머신-도커-컨테이너-실행)
  2. [도메인 구매](#도메인-구매)
  3. [도메인 레코드 추가](#도메인-레코드-추가)
  4. [각 도메인 레코드에 따른 포트포워딩 설정](#각-도메인-레코드에-따른-포트포워딩-설정)

(Azure 클라우드 서비스 기준으로 설명합니다.)
#### 가상머신 도커 컨테이너 실행
`docker run`로 직접 띄우거나, `docker compose`를 사용할 수도 있지만 팀에서는 Azure Pipelines로 컨테이너를 실행하였습니다. 자세한 내용은 [Azure Pipelines로 CI/CD 자동화하기]()를 참조하세요.

#### 도메인 구매
클라우드 서비스에서도 직접 구매할 수 있습니다.
- 참고: [Azure에서 도메인 구매하는 법](https://mpain.tistory.com/227)

#### 도메인 레코드 추가
DNS에 대한 레코드를 추가합니다.

![common-resource](https://user-images.githubusercontent.com/43571225/170854853-2213259a-4443-4bb3-b84c-7593f44f267f.png)

참고1. [리소스 그룹]에서 붉은색 표시선과 같이 구매한 도메인(`jjjlyn.com`)과 동일한 이름의 DNS Zone이 생성됩니다.

![common-resource-detail](https://user-images.githubusercontent.com/43571225/170854863-0a52f1cb-2a0d-4c6e-a3b9-3fc97a97be69.png)

참고2. [DNS 영역]으로 들어와 레코드 추가를 클릭하여 별칭을 생성할 수 있습니다.
예시로 `admin`이라는 별칭을 생성하면 앞서 [리소스 그룹]에서 지정된 `jjjlyn.com`이 뒤에 붙어 `admin.jjjlyn.com`이 VM IP 주소를 바라보게 됩니다.

![vm-ip](https://user-images.githubusercontent.com/43571225/170854867-aa40befd-e5b8-4958-92d6-4c1b4cd0ff85.png)

참고3. DNS가 바라보는 외부 IP (VM) 정보입니다.

#### 각 도메인 레코드에 따른 포트포워딩 설정

먼저 ssh로 VM에 접근합니다.
```shell
$ ssh -i private-key [사용자명]@[VM IP 주소 혹은 DNS] -p 22
```

Nginx 설정 폴더에 진입합니다.
```shell
$ cd /etc/nginx/
$ ls
```
![nginx-folder-structure](https://user-images.githubusercontent.com/43571225/170855661-71c823a1-d227-4823-8272-5c9730984fdd.png)

`sites-enabled`에 진입하여 위에서 파일을 생성합니다.<br>
(관리가 용이하도록 레코드를 붙인 DNS를 파일명으로 지정했습니다.)
```shell
$ cd sites-enabled/
$ vi admin.jjjlyn.com
```

운영 환경이기 때문에 미리 ssl 인증서를 구입하여 https를 적용하였습니다. 
- 참고: [SSL 인증서 구입처](https://www.koreassl.com/)

```shell
# 80번(http)로 접근하였을 때 https로 포워딩 합니다.
server {
       listen 80;

       server_name admin.jjjlyn.com;
       location / {
                return 301 https://admin.jjjlyn.com$request_uri;
       }
}
server {
        listen 443;
        server_name admin.jjjlyn.com;

        ssl on;

        # 사전에 구매한 사설 인증서를 적용합니다.
        ssl_certificate /etc/nginx/server_jjjlyn.crt;
        ssl_certificate_key /etc/nginx/server_jjjlyn.key;
        ssl_session_timeout 3m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers   on;

        client_max_body_size 0;
        chunked_transfer_encoding on;
        location / {
                rewrite ^(/.*)$ $1 break;
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-Proto $scheme;
                # admin.jjjlyn.com으로 들어온 요청은 내부망의 4000번 포트로 포워딩 합니다.
                proxy_pass http://127.0.0.1:4000/;
                proxy_redirect off;
        }
}
```

`nginx.conf`에서 `admin.jjjlyn.com` 포트포워딩 설정이 반영되도록 합니다.
```shell
$ cd ..
$ vi nginx.conf

http {
        # 기본 설정 생략
        include /etc/nginx/sites-enabled/*;
}
```

:two: 어플리케이션 별 CI/CD 파이프라인 구축

자세한 내용은 링크를 통해 확인하실 수 있습니다.
  1. [안드로이드 어플리케이션]()
  2. [서버 어플리케이션]()
  3. [정적 웹앱]()


### 개발 환경

:one: 전체 구성

운영에 비해 구조를 간소화 하였습니다. 이를테면 웹서버를 통한 포트포워딩은 과감히 생략하였습니다.

![dev-structure](https://user-images.githubusercontent.com/43571225/171211376-dbfde753-113d-45ae-ab96-3c4013182dd6.png)

:two: 클라우드 서비스

**Azure vs. AWS : 운영과 동일한 클라우드 서비스를 사용할 것인가?**

서비스 첫 배포에 앞서 `개발 | 운영` 환경을 분리해야 했습니다.
운영 인프라 환경은 Azure로 구축한 상태여서 개발 환경도 동일하게 구성하려 했으나 **비용 절감** 차원에서 AWS의 EC2와 RDS를 채택했습니다. Azure 가상머신과 비교하여 한화로 약 10만원 정도의 차이가 있으며, 때마침 스타트업 이벤트성(?) AWS 크레딧을 지급 받은 덕에 사용하지 않을 이유가 없었죠. (액셀 자료 기재)

팀 내부에서만 사용하기에 트래픽이 거의 발생할 일이 없어
- **vCPU 4개, RAM 16GiB, SSD 64GiB**의 일반적인 성능을 갖춘 가상머신을 선택하였습니다.
- RDS는 뭐더라? (나중에 추가 기재)

:three: Docker Private Registry

Azure Pipelines 빌드 환경에서 내부 사설 도커 레지스트리에 접근하는 것은 외부망에서 내부망으로 접속하는 것과 같습니다. 도커 환경에서는 보안상의 이유로 클라이언트와 원격지 레지스트리 간 통신에서 https 프로토콜만을 허용합니다. 이 이유 하나만으로 운영 환경도 아닌데 사설 ssl 인증서를 구매하는 것은 부담이 되었습니다. 그래서 자체 서명 인증서(Self-Signed Root CA)를 생성하게 되었죠.

자체 서명 인증서를 생성하여 도커 레지스트리 컨테이너에 적용하는 방법 입니다. 키 이름은 편의상 **temp**로 지정하였습니다. 폴더명과 경로는 `~/certs`로 하는 것을 권장합니다. 여기서는 편의상 `test-ssl`라는 임시 폴더를 생성하였습니다.
```shell
mkdir ./test-ssl
cd test-ssl/
openssl genrsa -des3 -out temp.key 2048
```
<img width="894" alt="스크린샷 2022-06-04 오후 3 20 32" src="https://user-images.githubusercontent.com/43571225/171987925-284073f2-a383-4d0c-82cb-f40ffb6876e0.png">


```shell
openssl req -new -key temp.key -out temp.csr
```
<img width="687" alt="스크린샷 2022-06-04 오후 3 22 41" src="https://user-images.githubusercontent.com/43571225/171987933-f6928501-9f39-46b3-bc6f-438de20c3056.png">


```shell
openssl rsa -in temp.key -out temp.key
```
<img width="778" alt="스크린샷 2022-06-04 오후 3 25 04" src="https://user-images.githubusercontent.com/43571225/171987940-c1510cb6-a1ce-46f3-b691-6edfb33f732b.png">

한가지 주의할 점은 아래 단계에서 {DOCKER_HOST_IP}에 **레지스트리가 존재하는 원격지의 외부 IP 주소**를 지정해야 한다는 것입니다. 차후 클라이언트에서 원격지 레지스트리에 REST API 요청을 할 때, 여기서 지정한 IP 주소로 호출을 시도하기 때문입니다.
```shell
echo subjectAltName=IP:{DOCKER_HOST_IP} > extfile.cnf
```

```shell
openssl x509 -req -days 800 -signkey temp.key -in temp.csr -out temp.crt -extfile extfile.cnf
```
<img width="962" alt="스크린샷 2022-06-04 오후 3 26 52" src="https://user-images.githubusercontent.com/43571225/171987963-4f999786-c056-44ca-9dff-11ff526fbfa9.png">

**클라이언트와 호스트 간에 https 통신을 하려면 호스트(원격지) 서버에서 자체 서명한 인증서를 클라이언트 환경의 `신뢰할 수 있는 인증서 목록`에 추가해야 합니다.** 이후에는 클라이언트에서 내부망의 도커 레지스트리에 자유롭게 REST API로 요청할 수 있습니다.

클라이언트 환경에서 REST API를 사용하여 원격지 레지스트리에 저장된 이미지 목록을 불러오게 하는 예시입니다.<br>
레지스트리 컨테이너는 기본적으로 포트 5000번에 할당됩니다. 
```shell
curl -X https://{DOCKER_HOST_IP}:5000/v2/_catalog
```
`docker ps`나 `docker pull` 등의 도커 관련 명령어를 사용하는 것도 내부적으로 REST API로 도커 데몬을 통해 도커 호스트로 요청하는 것과 같습니다.

그런데 레지스트리에 대한 권한이 있는 사용자만 접근할 수 있어야 하겠죠. 이를 위해 레지스트리에 basic auth로 추가 보안을 해주어야 합니다.

```shell
apt install apache2-utils
htpasswd -Bbn {레지스트리 ID} {비밀번호} > /home/{사용자 디렉토리}/certs/htpasswd
```

레지스트리 이미지를 컨테이너로 띄웁니다. TLS와 basic auth를 모두 지정해야 합니다.
```shell
docker run --name registry -d --restart=always -p 5000:5000 \  
-v /home/ubuntu/test-ssl:/test-ssl \  
-v /data/registry:/var/lib/registry/Docker/registry/v2 \  
-e REGISTRY_HTTP_TLS_CERTIFICATE=/test-ssl/temp.crt \  
-e REGISTRY_HTTP_TLS_KEY=/test-ssl/temp.key \    
-e REGISTRY_AUTH=htpasswd \     
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \     
-e REGISTRY_AUTH_HTPASSWD_PATH=/test-ssl/htpasswd \     
registry:2.7.0  
```

이제 클라이언트에서 원격지 레지스트리의 아이디, 패스워드를 헤더에 담아 요청하지 않으면 `authentication required` 오류 메시지가 뜨면서, 권한이 없는 사용자는 접근하지 못하게 됩니다.
![basic-auth-failure](https://user-images.githubusercontent.com/43571225/171328664-6c9411eb-a25d-4ef0-995c-c384c5ce92e7.png)

아래와 같이 레지스트리 ID와 비밀번호를 헤더에 담아 요청해야 합니다.
```shell 
curl --user {레지스트리 ID}:{비밀번호} -X GET https://{외부 IP}:5000/v2/_catalog
```

레지스트리 이미지 목록을 성공적으로 가져온 예시입니다.
![basic-auth-success](https://user-images.githubusercontent.com/43571225/171328728-eafba112-1033-4b99-8527-16fc7a1cbe64.png)

:question: 위에서도 언급하였지만, 클라이언트와 호스트 간에 https 통신을 하려면 자체 서명 인증서를 클라이언트의 `신뢰할 수 있는 인증서 목록`에 추가해야 합니다. 로컬에서 원격지의 레지스트리에 접근하는 경우에는 로컬(클라이언트) 환경에 한번 적용한 후 계속 목록에 유지될 것입니다. 그러나 Azure Pipelines 같이 매번 새로운 빌드 환경을 제공할 때는 어떻게 해야 할까요?

저는 고민 끝에 꼼수를 하나 썼습니다. 자체 인증서 추가 로직을 CI/CD 파이프라인에 삽입한 것입니다. 과연 올바른 방법인지는 모르겠으나, 당시 달리 떠오르는 방법이 없었고 같은 고민을 하시는 분들이 있을 것 같아 공유합니다.

## 운영 중 이슈
:one: 점점 쌓여가는 도커 blob images... 어떻게 지울 것인가? 레지스트리가 full일 경우를 대비하자

[마법같은 docker image GC 스크립트](https://github.com/andrey-pohilko/registry-cli)
