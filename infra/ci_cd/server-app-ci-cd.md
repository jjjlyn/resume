# 서버 어플리케이션 CI/CD 파이프라인 구축
## Dockerfile
Dockerfile이란 docker가 기본적으로 제공하는 이미지를 이용하여 커스텀 이미지를 생성할 수  있는 스크립트 파일 입니다.

먼저 전역변수를 설정합니다.
```Dockerfile
ARG port=3000
ARG binary=product-api
```
**빌드 환경을 구축합니다.**

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

**배포 환경을 구축합니다.**
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

**전체 파일**
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

![](/infra/images/ap_service_connection_add_private_registry.png)

![](/infra/images/ap_service_connection_private_registry_settings.png)
