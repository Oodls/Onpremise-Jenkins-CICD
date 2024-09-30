실습 환경

jenkins container

ubuntu host의 일반 폴더와 jenkins container를 bind 마운트

jar로 빌드해서 마운트된 폴더에 jar를 복사 → host의 폴더에 복사가 됨

host일반 폴더에 jar앱을 실행 하는 과정을 파이프라인으로 구현


host 

## 0. 실행환경 준비

```
docker run --name myjenkins --privileged -p 8080:8080 -v $(pwd)/appjardir:/var/jenkins_home/appjar jenkins/jenkins:lts-jdk17
```

### genkins public 주소 할당 
- GitHub 웹훅은 GitHub에서 이벤트가 발생할 때(예: 새로운 커밋, 풀 리퀘스트 등) 지정된 URL로 HTTP 요청을 보내는 방식
- 이 요청은 GitHub 서버에서 Jenkins 서버로 직접 전송되기 때문에, Jenkins가 외부에서 접근 가능한 public IP 또는 도메인을 가져야만 GitHub가 요청을 보낼 수 있다

**이를 위해 ngrok 활용!**
![image](https://github.com/user-attachments/assets/781a877f-8038-42ba-a272-18e6c1b75e29)
ngrok은 로컬에서 실행 중인 애플리케이션을 안전하게 인터넷에 노출할 수 있는 터널링 서비스다. 
예를 들어, 로컬 개발 환경에서 실행 중인 웹 애플리케이션이나 서버를 외부에서 접근 가능하도록 할 수 있다. 
주로 테스트나 개발 중 GitHub 웹훅, Slack API 등과 같은 외부 서비스를 테스트할 때 유용하다.

1. ngrok 설치
ngrok.exe 파일을 다운로드

2. ngrok.exe 파일 실행 후 해당 터미널에 입력
```
ngrok config add-authtoken [토큰값]

ngrok http http://localhost:[jenkins 실행 포트]
```

![image](https://github.com/user-attachments/assets/2d9b53bd-5f29-4b11-bc6f-0644f26999ad)
그 결과로 https://e5d3-118~~~ 의 public주소를 얻을 수 있었다.

### 깃허브 repository 설정
- 깃허브 레포의 settings -> webhook 에서 **[genkins의public주소]/github-webhook/** 을 payload url로 설정 
![image](https://github.com/user-attachments/assets/0bef6c5f-eb91-40c3-8c46-7a10136079c9)

### gradle 설정

### 파이프라인 구성
- 파이프라인에서 깃허브 레포의 변경을 감지하여 파이프라인을 실행하기 위해 github hook trigger 설정
![image](https://github.com/user-attachments/assets/25981c13-7f73-4a28-93f0-7e258c63bfad)

## 1. CI 구현

```
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: https://github.com/leesj000603/jenkins-test2'
            }
        }
          
        stage('Build') {   
            steps {
                dir('./step18_empApp') {                   
                    sh 'chmod +x gradlew'                    
                    sh './gradlew clean build -x test'   //*.jar
                    sh 'echo $WORKSPACE'
                }
            }
        }
        
        stage('Copy JAR') { 
            steps {
                script {
                    def jarFile = 'step18_empApp/build/libs/step18_empApp-0.0.1-SNAPSHOT.jar'                   
                    sh "cp ${jarFile} /var/jenkins_home/appjar/"
                }
            }
        }
    }
}
```

바인드 설정하고 실행

호스트의 $(pwd)/appjardir 컨테이너의 /var/jenkins_home/appjar 바인드 마운트


### jar파일을 복사하는 단계에서 실패
![image](https://github.com/user-attachments/assets/c94d4ba5-e82b-414c-855a-6ab6e9ed901e)
### 그 이유는 cp 명령을 위한 권한이 없었기 때문
![image](https://github.com/user-attachments/assets/73c28dc9-5cfd-461b-9a36-abd4d53a4ec9)


```
#루트로 들어가기
docker exec -u root -it 317871cc892b bash 

# appjar 디렉토리에 권한 부여해야함. (copy 명령이 필요하기 때문에 복사 대상 디렉토리에는 읽기 권한이 필요)
chmod 722 -R /var/jenkins_home/appjar
```
### 그 후 정상 동작을 확인.
![image](https://github.com/user-attachments/assets/c5d83e34-25f1-4459-bea7-4f77b2be52fd)


### 콘솔 출력 확인 
### 정상적으로 gitrepo가 clone이 되었고, build가 성공하였다. 
### 그 결과물인 jar파일을 바인드 마운트된 디렉토리에 복사하였음

```
Started by user admin
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/jenkins_home/workspace/pip1
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Clone Repository)
[Pipeline] git
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/pip1/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/leesj000603/jenkins-test2 # timeout=10
Fetching upstream changes from https://github.com/leesj000603/jenkins-test2
 > git --version # timeout=10
 > git --version # 'git version 2.39.2'
 > git fetch --tags --force --progress -- https://github.com/leesj000603/jenkins-test2 +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision 34d14d2001162985d95243c1cdb01ab3bf2c64ca (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 34d14d2001162985d95243c1cdb01ab3bf2c64ca # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git branch -D main # timeout=10
 > git checkout -b main 34d14d2001162985d95243c1cdb01ab3bf2c64ca # timeout=10
Commit message: "jenkins test를 위한 프로젝트"
 > git rev-list --no-walk 34d14d2001162985d95243c1cdb01ab3bf2c64ca # timeout=10
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Build)
[Pipeline] dir
Running in /var/jenkins_home/workspace/pip1/SpringApp
[Pipeline] {
[Pipeline] sh
+ chmod +x gradlew
[Pipeline] sh
+ ./gradlew clean build -x test
Starting a Gradle Daemon (subsequent builds will be faster)
> Task :clean
> Task :compileJava
> Task :processResources
> Task :classes
> Task :resolveMainClassName
> Task :bootJar
> Task :jar
> Task :assemble
> Task :check
> Task :build

BUILD SUCCESSFUL in 8s
6 actionable tasks: 6 executed
[Pipeline] sh
+ echo /var/jenkins_home/workspace/pip1
/var/jenkins_home/workspace/pip1
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Copy jar)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ cp SpringApp/build/libs/SpringApp-0.0.1-SNAPSHOT.jar /var/jenkins_home/appjar/
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

jenkins 컨테이너에 바인드 마운트 된 appjar 디렉토리에 성공적으로 복사된 모습
![image](https://github.com/user-attachments/assets/2f66a1d1-f7d3-4761-8307-41f552191a5b)

그리고 역시 host의 같은 바인드 디렉토리에도 존재하는 모습 확인
![image](https://github.com/user-attachments/assets/07baac31-47fa-4664-9a53-f3c69d740269)


## 2. CD 구현

```bash
#!/bin/bash

# 변수 설정
JAR_FILE="SpringApp-0.0.1-SNAPSHOT.jar"
DEPLOY_DIR="/home/username/appjardir"

# 이전 JAR 파일 백업
if [ -f "$DEPLOY_DIR/$JAR_FILE" ]; then
  mv "$DEPLOY_DIR/$JAR_FILE" "$DEPLOY_DIR/$JAR_FILE.bak"
fi

# 새로운 JAR 파일 복사
cp $JAR_FILE $DEPLOY_DIR/$JAR_FILE

# Spring Boot 애플리케이션 재시작
# 기존 8999 포트 사용 중인 프로세스 종료
if sudo lsof -i :8999 > /dev/null; then
  # 8999 포트가 사용 중일 경우 이전 프로세스를 종료
  sudo kill -9 $(sudo lsof -t -i:8999)
fi

# 백그라운드에서 새로 실행
# > $DEPLOY_DIR/app.log : 애플리케이션 로그를 app.log 파일에 저장하도록 구성
nohup java -jar $DEPLOY_DIR/$JAR_FILE --server.port=8999 > $DEPLOY_DIR/app.log 2>&1 &

echo "배포완료 및 실행됩니다."
```

프로젝트 파일을 8999 포트로 실행중이었다.

현재 상황이 host 에 spring server가 있고  jenkins가 host위에  docker container로 올라가있는 상황

jenkins pipeline에서 이를 실행하려면 jenkins container 에서 host server의 jar파일을 실행하는 명령을 해야하는데 
이를 위해 ssh 명령을 사용하였다.

### jenkins 컨테이너 내부
```
apt update
apt install openssh-server

# 키 생성 후 기본 경로에 저장, 비밀번호 생성 x
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# 공개키 확인 후 복사
cat ~/.ssh/id_rsa.pub
```

### 호스트
```
# host의  ~/.ssh/authorized_keys에 적용
echo "복사한 키" >> ~/.ssh/authorized_keys
```

### jenkins에서 ssh를 사용하기 위한 플러그인 설치
- ssh Agent 설치
![image](https://github.com/user-attachments/assets/bd0be9c7-2953-4f82-971a-85b244f8242a)

- credentials 선택
![image](https://github.com/user-attachments/assets/f520a69a-53d2-4bed-a820-181beb829ebb)

- Add credentials 선택
![image](https://github.com/user-attachments/assets/9fcabe37-4cb9-4531-b240-0d6cb95d4ddf)

- 해당 개인key를 하단의 private key에 복사 및 기타 설정
```
cat ~/.ssh/id_rsa
```
![image](https://github.com/user-attachments/assets/041316e7-4134-450d-be6e-15e6f796b88c)


### 파이프라인
```
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/leesj000603/jenkins-test2'
            }
        }
          
        stage('Build') {
            steps {
                dir('./SpringApp') {                   
                    sh 'chmod +x gradlew'                    
                    sh './gradlew clean build -x test'
                    sh 'echo $WORKSPACE'
                }
            }
        }
        
        stage('Copy jar') { 
            steps {
                script {
                    def jarFile = 'SpringApp/build/libs/SpringApp-0.0.1-SNAPSHOT.jar'                   
                    sh "cp ${jarFile} /var/jenkins_home/appjar/"
                }
            }
        }
        stage('Run jar on Host') {
            steps {
                script {
                    // SSH를 통해 호스트에서 JAR 파일을 실행
                    def host = 'username@10.0.2.15' // 호스트의 사용자 이름과 IP 주소
                    def jarPath = '/home/username/appjardir/SpringApp-0.0.1-SNAPSHOT.jar'
                    sh "ssh -o StrictHostKeyChecking=no ${host} 'nohup java -jar ${jarPath} > /home/username/appjardir/logs/output.log 2>&1 &'"
                }
            }
        }
    }
}

```

sudo 명령을 스크립트로 실행할 때 pw를 요구해서 프로세스가 kill되지 않아 생긴 문제
port충돌 오류가 계속 발생했다

```
#sudoers 파일 수정 하기
sudo visudo
```

```
# 스크립트에 패스워드를 사용하지 않도록 해당 권한 추가
myuser ALL=(ALL) NOPASSWD: /path/to/your/script.sh
```


![image](https://github.com/user-attachments/assets/85161c92-d575-4d28-8e6f-90094a93f719)
### 이후 정상적으로 실행까지 성공!

![image](https://github.com/user-attachments/assets/1bb9e020-96b9-4e93-b6d2-b364c8e728e2)


### repo push등의 변경사항 발생 시
![image](https://github.com/user-attachments/assets/7eaaf6f3-b22c-4b86-bf8c-04c8e0eca98a)
### 이를 감지
![image](https://github.com/user-attachments/assets/29674090-2c00-4694-935d-d249d479bc92)

