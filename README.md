# 코디세이 개발 작업실 구축

## 1. 프로젝트 개요

리눅스 터미널에서 파일과 권한을 다루고,
Docker로 웹 서버를 실행한 뒤 포트 매핑과 저장 공간을 검증했다.
Git으로 작업 이력을 관리하고 수행 결과를 기록한다.

## 2. 실행 환경

- 호스트 OS : Windows 11
- 리눅스 환경 : WSL2 / Ubuntu 26.04.1 LTS
- 셸 : bash
- 터미널 : Windows PowerShell에서 WSL Ubuntu 실행
- Docker : Docker Desktop / Engine 29.9.2
- Git : Ubuntu 내 Git 2.53.0

## 3. 프로젝트 구성

- `Dockerfile` : NGINX 기반 커스텀 이미지 제작 지침
- `app/index.html` : 직접 작성한 웹페이지
- `practice/` : 파일 조작 및 권한 실습 파일
- `evidence/` : 터미널 명령과 출력 기록

## 4. 실습 로그

- [터미널/권한 실습](evidence/terminal-session.log)
- [Docker 기본 실습](evidence/docker-session.log)
- [이미지 빌드 및 후속 실습](evidence/build-session.log)

## 5. Dockerfile과 커스텀 이미지

```dockerfile
FROM nginx:alpine
COPY app/ /usr/share/nginx/html/
```

- `FROM` : Alpine 기반 NGINX 이미지를 바탕으로 사용한다.
- `COPY` : 직접 작성한 웹페이지를 NGINX의 기본 웹 제공 경로에 복사한다.
- 커스텀 목적 : NGINX 기본 페이지 대신 내가 만든 페이지를 제공한다.

빌드할 때 사용한 명령 :

```bash
docker build -t codyssey-web:1.0 .
docker images
```

빌드가 FINISHED로 완료됐고, 이미지 목록에서
`codyssey-web:1.0`을 확인했다.

주의 : 최초 이미지는 버전 1 페이지로 빌드했다.
이후 바인드 마운트 실습에서 원본 HTML을 버전 2로 수정했다.
따라서 현재 소스로 다시 빌드하면 버전 2가 이미지에 포함된다.

## 6. 포트 매핑과 접속 확인

실행한 명령 :

```bash
docker run -d --name codyssey-web-8080 -p 8080:80 codyssey-web:1.0
```

호스트의 8080 포트를 컨테이너의 80 포트에 연결했다.
브라우저에서 http://localhost:8080에 접속하여
직접 작성한 웹페이지와 '버전: 1'문구를 확인했다.

접속 증거 :

![8080 포트 웹페이지 접속](evidence/localhost_8080.png)

## 7. 바인드 마운트 변경 반영

실행한 명령 : 

```bash
docker run -d --name codyssey-bind -p 8081:80 --mount type=bind,source="$(pwd)/app",target=/usr/share/nginx/html,readonly codyssey-web:1.0
```

호스트의 app 폴더를 컨테이너의 웹 제공 경로에 연결했다.
readonly는 컨테이너에서 연결된 파일을 수정하지 못하게 한다.
호스트에서는 원본 파일을 수정할 수 있다.

검증 절차 :

1. localhost:8081에서 버전 1 페이지를 확인했다.
2. 호스트의 app/index.html 문구를
	'버전 2 - 바인드 마운트 반영 확인'으로 수정했다.
3. 이미지를 다시 빌드하거나 컨테이너를 재시작하지 않고 브라우저를 새로고침했다.
4. localhost:8081에는 버전 2가 표시됐고, localhost:8080에는 버전 1이 유지됐다.

결론 : COPY는 빌드 당시의 파일을 이미지에 복사하고,
바인드 마운트는 연결된 호스트 폴더의 현재 파일을 제공한다.

변경 전 - 8081에서 버전 1 확인 : 

![바인드 마운트 변경 전](evidence/localhost_8081.png)

변경 후 - 8081에서 버전 2 확인 :

![바인드 마운트 변경 후](evidence/localhost_8081_V2.png)

8080과 8081 비교 :

![이미지의 버전 1과 바인드 마운트의 버전 2 비교](evidence/8080_8081_comp.png)

## 8. Docker 볼륨 영속성 검증

컨테이너를 삭제해도 데이터가 유지되는지 확인했다.

### 볼륨 생성과 파일 저장

실행한 명령 : 

```bash
docker volume create codyssey-data
docker run -dit --name codyssey-volume-1 --mount type=volume,source=codyssey-data,target=/data ubuntu:24.04 bash
docker exec codyssey-volume-1 bash -c 'echo "Codyssey volume test" > /data/hello.txt'
docker exec codyssey-volume-1 cat /data/hello.txt
```

파일 확인 결과

```text
Codyssey volume test
```

### 컨테이너 삭제 후 데이터 재확인

첫 번째 컨테이너만 삭제하고, 같은 볼륨을 새 컨테이너에 연결했다.

```bash
docker rm -f codyssey-volume-1
docker run -dit --name codyssey-volume-2 --mount type=volume,source=codyssey-data,target=/data ubuntu:24.04 bash
docker exec codyssey-volume-2 cat /data/hello.txt
```

새 컨테이너에서도 다음 내용이 출력됐다.

```text
Codyssey volume test
```

결론 : 파일을 다시 만들지 않았는데도 내용이 유지됐다.
데이터가 컨테이너의 쓰기 계층이 아닌 별도의 codyssey-data 볼륨에
저장되어 있기 때문이다. 볼륨 자체는 삭제하지 않았다.

## 9. 트러블슈팅

### 사례 1. 존재하지 않는 파일 삭제 시도

- 문제 : `rm rename.txt`와 `rm renamed.txt`에서
`No such file or directory` 오류가 발생했다.
- 원인 가설 : 삭제할 파일 이름이 잘못됐거나,
아직 해당 이름으로 변경하지 않았을 수 있다고 생각했다.
- 확인 : 앞서 실행한 `ls -la`결과에는 `copy.txt`가 있었고,
`renamed.txt`는 없었다. 이름 변경 명령을 빠뜨린 것을 확인했다.
- 해결 : 파일 이름을 변경한 뒤 삭제했다.

```bash
mv copy.txt renamed.txt
rm renamed.txt
ls -la
```

- 결과 : 오류 없이 삭제됐고, 목록에서 해당 파일이 사라졌다.

### 사례 2. --mount 설정값 누락

- 문제 : `docker run ` 명령을 `--mount`까지만 입력하자
다음 오류가 발생했다.

```text
flag needs an argument: --mount
```

- 원인 가설 '--mount'에 필요한 연결 설정값이
빠졌기 때문이라고 생각했다.
- 확인 : 도움말의 `--mount mount`는 옵션 뒤에
마운트 설정값이 필요하다는 뜻이었다.
- 해결 : 마운트 유형, 원본 경로, 연결 위치와 읽기 전용 설정을 지정하고 
이미지 이름까지 입력했다.

```bash
docker run -d --name codyssey-bind -p 8081:80 --mount type=bind,source="$(pwd)/app",target=/usr/share/nginx/html,readonly codyssey-web:1.0
```

- 결과 : 컨테이너 ID가 출력됐고,
localhost:8081에서 웹페이지 접속과 파일 변경 반영을 확인했다.

## 10. 수행 체크리스트

- [x] 터미널 기본 파일 조작
- [x] 파일/디렉토리 권한 변경 전후 확인
- [x] Docker 버전 및 엔진 동작 확인
- [x] hello-world 실행
- [x] Ubuntu 컨테이너 내부 명령 실행
- [x] 이미지/컨테이너 목록, 로그, 리소스 확인
- [x] 컨테이너 중지/재시작과 attach/exec 차이 관찰
- [x] Dockerfile 직접 작성 및 커스텀 이미지 빌드
- [x] 포트 매핑 및 브라우저 접속 확인
- [x] 바인드 마운트 파일 변경 반영 확인
- [x] 볼륨의 컨테이너 삭제 전후 데이터 유지 확인
- [x] Git 작성자 이름/이메일과 기본 브랜치 설정
- [x] 로컬 Git 저장소 생성 및 첫 커밋
- [x] 트러블슈팅 2건 기록
- [x] 접속/마운트 비교 캡처를 저장소에 넣고 README에 연결
- [x] Git 설정 확인 결과 기록
- [ ] VS Code에서 Github 로그인 및 저장소 연동 확인
- [ ] GitHub 업로드 및 제출 링크 확인

## 11. Git 설정 및 로컬 버전 관리

설정 확인 명령과 출력 : 

```console
$ git config --list --local
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true
user.name=DasFd-rgb
user.email=cedn2015@gmail.com

$ git config --global --get init.defaultBranch
main

$ git branch --show-current
main
```

작성자 이름과 이메일은 현재 저장소에 설정했다.
새 저장소의 기본 브랜치와 현재 작업 브랜치는 모두 main이다.

첫 커밋 확인 결과 : 

```console
$ git log -1 --oneline
5647024 (HEAD -> main) Add workstation practice and Docker web server
```

Git은 로컬 변경 이력을 관리하고,
GitHub는 저장소를 온라인에서 공유하고 협업하는 서비스다.
현재 첫 커밋은 로컬에 저장했으며 GitHub 업로드는 아직 진행하지 않았다.
