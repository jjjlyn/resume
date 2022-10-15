# 서버 어플리케이션 CI/CD 파이프라인 구축
## 지속적 통합 (CI)
서버 어플리케이션에 Dockerfile을 생성합니다.</br>
**Dockerfile**이란 docker가 기본적으로 제공하는 이미지를 이용하여 커스텀 이미지를 생성할 수 있는 스크립트 파일 입니다.

먼저 전역변수를 설정합니다.
```Dockerfile
ARG port=3000
ARG binary=product-api
```
빌드 환경을 구축합니다.

알파인 리눅스(alpine linux)는 도커 컨테이너 OS로 많이 사용되는 초경량화된 리눅스 배포판 입니다.
```Dockerfile
FROM golang:1.16.2-alpine3.13 as builder
```

Multi-Stage 도커파일에서 binary ARG를 공유합니다.
```Dockerfile
ARG binary
```

관리자를 지정합니다.
```Dockerfile
MAINTAINER jjjlyn <prize1142@gmail.com>
```

알파인 리눅스에서 패키지 관리자 명령어는 apk입니다.
```Dockerfile
RUN apk update && apk upgrade && \
    apk --update add git make gcc g++
```

`cd app/`과 동일합니다. 도커 컨테이너 OS의 /app 디렉토리로 이동합니다.
```Dockerfile
WORKDIR /app
```

호스트 OS의 경로에서 현재 위치한 경로로 파일을 복사합니다.
```Dockerfile
COPY . .
```

도커 컨테이너 전체에서 쓸 수 있도록 binary 환경변수를 지정합니다.
```Dockerfile
ENV binary=$binary
```

`go get-dependencies`와 동일합니다.
```Dockerfile
RUN make get-dependencies
```

`go build`와 동일합니다.
```Dockerfile
RUN make build
```

배포 환경을 구축합니다.
```Dockerfile
FROM alpine:3.10 as distribution
```

ARG는 각 단계(stage)에서만 유효합니다.</br>
그러므로 새로운 단계인 distribution에서 port, binary ARG를 갱신해야 합니다. 
```Dockerfile
ARG port
ARG binary
```

패키지 업데이트 및 TLS를 위한 ca-certificates를 설치합니다.
```Dockerfile
RUN apk update && apk upgrade && \
    apk --update add bash ca-certificates
```

컨테이너 내부에서 사용할 수 있는 변수로 PORT와 binary를 지정합니다.
```Dockerfile
ENV PORT=$port
ENV binary=$binary
```

`cd app/`과 동일합니다. 도커 컨테이너 OS의 /app 디렉토리로 이동합니다.
```Dockerfile
WORKDIR /app
```

**builder** alias를 설정한 단계에서 빌드된 파일을 배포 단계 컨테이너 OS /app 디렉토리로 복사합니다.
```Dockerfile
COPY --from=builder /app/${binary} /app
```

컨테이너의 특정 포트를 열어주는 명령어 입니다.(container listen to traffic on specific port)</br>
`EXPOSE` 명령어가 의미 있으려면, `docker run`의 **-p** 옵션으로 호스트 컴퓨터의 특정 포트를 해당 컨테이너의 특정 포트로 포워딩 해주어야 합니다. 
```Dockerfile
EXPOSE ${PORT}
```

`docker exec -it ${컨테이너 ID} bash`로 컨테이너 접속 시 `pwd`(현재 위치 장소가 /app)입니다.
```Dockerfile
CMD /app/${binary}
```
이로써 `docker container run` 시 실행되는 최종 이미지가 완성됩니다.

**전체 Dockerfile** 

Go 전용
```Dockerfile
ARG port=3000
ARG binary=product-api

# 빌드
FROM golang:1.16.2-alpine3.13 as builder

ARG binary

MAINTAINER jjjlyn <prize1142@gmail.com>

RUN apk update && apk upgrade && \
    apk --update add git make gcc g++

WORKDIR /app

COPY . .

ENV binary=$binary

RUN make get-dependencies

RUN make build

# 배포
FROM alpine:3.10 as distribution

ARG port
ARG binary

MAINTAINER jjjlyn <prize1142@gmail.com>

RUN apk update && apk upgrade && \
    apk --update add bash ca-certificates

ENV PORT=$port
ENV binary=$binary

WORKDIR /app

COPY --from=builder /app/${binary} /app

EXPOSE ${PORT}

CMD /app/${binary}
```

Spring Boot 전용
```Dockerfile
# Docker Multi-stage Build
# [참고] https://blog.outsider.ne.kr/1300

# jar를 빌드하는 환경에 대한 별칭을 지정(builder)
FROM adoptopenjdk/openjdk11:alpine-slim as builder

# 도커 컨테이너 내부 디렉토리
WORKDIR /app

# 호스트 환경에서 도커 환경으로 복사
COPY . .

# 빌드 성공 시 jar 파일이 생성
RUN ./gradlew bootJar

# jar를 실행하는 환경에 대한 별칭을 지정(distribution)
FROM adoptopenjdk/openjdk11:alpine-jre as distribution

#도커 컨테이너 내부 디렉토리
WORKDIR /app

# --from=builder: 컴파일된 jar 파일만 끌어온다
# 이미지를 만드는 데 필요한 의존성과 실제 최종 이미지에서만 필요한 의존성을 완전히 분리해서 
# 최종 Docker 이미지를 아주 간단하게 만들 수 있다
COPY --from=builder /app/build/libs/app.jar .

# 이 이미지가 해당 포트를 외부로 개방할 것이라고 명시하는 것
# 컨테이너를 실행할 때 호스트 OS로 들어오는 포트와 해당 포트를 매핑시켜야 한다
# docker run --port=$(port)
# 각 어플리케이션 별로 컨테이너가 생성될 것이므로 포트번호가 다른 앱과 겹쳐도 상관없음
# (컨테이너 내부에서만 포트 충돌이 발생하지 않으면 됨)
EXPOSE 7001

# 컨테이너로 올라가면 코드 실행
CMD java -jar /app/app.jar 
```

다음은 Dockerfile로 빌드한 이미지를 private registry에 푸시하는 작업입니다. 이를 위해서는 Azure DevOps의 Service Connections에 Private Registry를 미리 생성해야 합니다.
![Add Private Registry to Service Connections](/infra/images/ap_service_connection_add_private_registry.png)

![Private Registry Service Connections Settings](/infra/images/ap_service_connection_private_registry_settings.png)

**CI 스크립트**
```yml
# Starter pipeline
# Start with a minimal pipeline that you can customize to build and deploy your code.
# Add steps that build, run tests, deploy, and more:
# https://aka.ms/yaml

trigger:
  branches:
    include:
      - master

resources:
- repo: self

pool:
  vmImage: 'ubuntu-latest'

variables:
  tag: '$(Build.BuildId)' #"`git rev-parse --short HEAD`"
  repositoryName: '$(Build.Repository.Name)'

stages:
- stage: Build
  displayName: Build image
  jobs:  
  - job: Build
    displayName: Build
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Docker@2
      inputs:
        containerRegistry: 'Private Registry'
        repository: $(repositoryName)
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        tags: |
          $(tag)
```

**개발환경에서는?**

운영 환경은 클라이언트 OS에서 기본적으로 신뢰하는 인증기관(CA)에서 발급받은 인증서를 사용하기에 private registry에 접근(`docker login`)하고자 하는 클라이언트 환경에 인증기관 목록으로 추가하지 않아도 됩니다. 그러나 개발 환경은 자체 인증서를 사용하기 때문에 클라이언트 환경에 이를 신뢰할 수 있는 인증기관으로 추가해야 합니다. 이를 위해 CI와 CD 파이프라인에 모두 추가하였습니다.

사전에 자체 인증서를 **Azure DevOps Library > Secure files**에 등록해야 합니다. 업로드한 키를 Pipelines 환경에서 다운받습니다.
![CA secure files](/infra/images/ap-library-ca.png)
```yml
steps:
  - task: DownloadSecureFile@1
    name: download_ca_certificates
    displayName: Download ca certificates
    inputs:
      secureFile: domain.crt
      retryCount: 5
```

Azure 클라이언트에서 private registry에 접근하기 전에 자체 인증서를 호스트의 신뢰할 수 있는 인증기관으로 추가합니다.
```yml
 - script: |
    sudo cp $(Agent.TempDirectory)/domain.crt /usr/local/share/ca-certificates/
    sudo update-ca-certificates    
```

**전체 CI 스크립트**
```yml
# Starter pipeline
# Start with a minimal pipeline that you can customize to build and deploy your code.
# Add steps that build, run tests, deploy, and more:
# https://aka.ms/yaml

trigger:
  branches:
    include:
      - develop

resources:
  - repo: self

pool:
  vmImage: 'ubuntu-latest'

variables:
  tag: '$(Build.BuildId)' #"`git rev-parse --short HEAD`"
  repositoryName: '$(Build.Repository.Name)'

stages:
  - stage: Build
    displayName: Build image
    jobs:
      - job: Build
        displayName: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: DownloadSecureFile@1
            name: download_ca_certificates
            displayName: Download ca certificates
            inputs:
              secureFile: domain.crt
              retryCount: 5

          - script: |
              sudo cp $(Agent.TempDirectory)/domain.crt /usr/local/share/ca-certificates/
              sudo update-ca-certificates     

          - task: Docker@2
            inputs:
              containerRegistry: 'Private Registry Dev'
              repository: $(repositoryName)
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(tag)
```

## 지속적 배포 (CD)
**환경변수 지정**
```bash
export origin_repo='$(Build.Repository.Name)'
export CONTAINER_NAME=${origin_repo#*/}-cont
export CONTAINER_NAME=${CONTAINER_NAME/_/-}
export IMAGE_NAME=${origin_repo,,}

echo ${origin_repo}
echo ${CONTAINER_NAME}
echo ${IMAGE_NAME}

# 위와 같이 CONTAINER_NAME, IMAGE_NAME 조작해서 azure vso 변수로 지정합니다
echo "##vso[task.setvariable variable=CONTAINER_NAME]${CONTAINER_NAME}"
echo "##vso[task.setvariable variable=IMAGE_NAME]${IMAGE_NAME}"
```

**Docker Pull & Retagging & Push**
```bash
docker pull docker.jjjlyn.io/$(IMAGE_NAME):$(Build.BuildId)

docker tag docker.jjjlyn.io/$(IMAGE_NAME):$(Build.BuildId) docker.jjjlyn.io/$(IMAGE_NAME):latest

docker push docker.jjjlyn.io/$(IMAGE_NAME):latest
```

**Pull Latest Image**
```bash
docker pull localhost:5000/$(IMAGE_NAME):latest
```

**Stop, Remove Container**
```bash
docker stop $(CONTAINER_NAME)
docker rm $(CONTAINER_NAME)
```

**Run Container**
```bash
# --net $(CONTAINER_NET): kong-net으로 묶여 있습니다
docker run -d -e port=$(port) -p {HOST Port}:{Docker Container Port} --net $(CONTAINER_NET) --name $(CONTAINER_NAME) --restart unless-stopped localhost:5000/$(IMAGE_NAME):latest
```