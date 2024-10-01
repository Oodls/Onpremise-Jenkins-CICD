# 📚 실습 환경

- ## 🚀 1. 빌드된 jar파일을 하나의 ubuntu vm 에서 받아 실행
    ![image](https://github.com/user-attachments/assets/6352f8eb-ba7a-4857-8bb5-77a213409119)



- ## 🚀 2. 빌드된 jar파일을 cicd용 ubuntu vm 에서 operation용 vmd으로 scp로 전송하여 실행
    ![image](https://github.com/user-attachments/assets/4b8018ff-ca59-4a26-8731-83a07766c31e)





- ### 🐳 Jenkins Container 설명
    
    🔗 Ubuntu host의 일반 폴더와 Jenkins container를 bind 마운트
    
    🏗️ JAR로 빌드해서 마운트된 폴더에 JAR를 복사 → host의 디렉토리에 복사가 됨
    
    🚀 Host server 디렉토리의 JAR 앱을 실행하는 과정을 파이프라인으로 구현




- ### 0️⃣ 실행환경 준비
    ```
    docker run --name myjenkins --privileged -p 8080:8080 -v $(pwd)/appjardir:/var/jenkins_home/appjar jenkins/jenkins:lts-jdk17
    ```

- ### 🌐 Jenkins Public 주소 할당 
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
    
    1. ngrok 설치
    ngrok.exe 파일을 다운로드
    
    2. ngrok.exe 파일 실행 후 해당 터미널에 입력
    ```
    ngrok config add-authtoken [토큰값]
    
    ngrok http http://localhost:[jenkins 실행 포트]
    ```

    ![image](https://github.com/user-attachments/assets/2d9b53bd-5f29-4b11-bc6f-0644f26999ad)
    <br>
    그 결과로 jenkins에 대한 public 주소로 https://e5d3-118~~~를 얻을 수 있었다.

- ### 🐙 GitHub Repository 설정
    - GitHub 레포의 settings -> webhook 에서 **[jenkins의public주소]/github-webhook/** 을 payload url로 설정 
    ![image](https://github.com/user-attachments/assets/0bef6c5f-eb91-40c3-8c46-7a10136079c9)

- ### 🛠️ Gradle 설정
    ![image](https://github.com/user-attachments/assets/5f5d8424-de45-4833-ac02-6a9e7456df61)
    project에서 사용한 gradle 버전 8.6으로 설정


# 🚀 빌드된 jar파일을 하나의 ubuntu vm 에서 받아 실행

- ### 🔧 파이프라인 설정
    - 파이프라인에서 GitHub 레포의 변경을 감지하여 파이프라인을 실행하기 위해 github hook trigger 설정
    ![image](https://github.com/user-attachments/assets/25981c13-7f73-4a28-93f0-7e258c63bfad)

- ## 1️⃣ CI 구현
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


- ### ❌ JAR 파일을 복사하는 단계에서 실패
    ![image](https://github.com/user-attachments/assets/c94d4ba5-e82b-414c-855a-6ab6e9ed901e)
- ### 🔍 그 이유는 cp 명령을 위한 권한이 없었기 때문
    ![image](https://github.com/user-attachments/assets/73c28dc9-5cfd-461b-9a36-abd4d53a4ec9)


    ```
    # 루트로 들어가기
    docker exec -u root -it 317871cc892b bash 
    
    # appjar 디렉토리에 권한 부여해야함. (copy 명령이 필요하기 때문에 복사 대상 디렉토리에는 읽기 권한이 필요)
    chmod 722 -R /var/jenkins_home/appjar
    ```
- ### ✅ 그 후 정상 동작을 확인.
    ![image](https://github.com/user-attachments/assets/c5d83e34-25f1-4459-bea7-4f77b2be52fd)


- ### 📋 콘솔 출력 확인 
- ### ✅ 정상적으로 gitrepo가 clone이 되었고, build가 성공하였다. 
- ### 🚚 그 결과물인 jar파일을 바인드 마운트된 디렉토리에 복사되었음

    - 그 결과 콘솔 output

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

- ### jenkins 컨테이너에 바인드 마운트 된 appjar 디렉토리에 성공적으로 복사된 모습
  ![image](https://github.com/user-attachments/assets/2f66a1d1-f7d3-4761-8307-41f552191a5b)

    
- ### 그리고 역시 host의 같은 바인드 디렉토리에도 존재하는 모습 확인
  ![image](https://github.com/user-attachments/assets/07baac31-47fa-4664-9a53-f3c69d740269)

- ## 2️⃣ CD 구현

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

- ### 🐳 Jenkins 컨테이너 내부
    ```
    apt update
    apt install openssh-server
    
    # 키 생성 후 기본 경로에 저장, 비밀번호 생성 x
    ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    
    # 공개키 확인 후 복사
    cat ~/.ssh/id_rsa.pub
    ```

    - ### 🖥️ 호스트
    ```
    # host의  ~/.ssh/authorized_keys에 적용
    echo "복사한 키" >> ~/.ssh/authorized_keys
    ```

- ### 🔌 Jenkins에서 SSH를 사용하기 위한 플러그인 설치
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


- ### 🔧 파이프라인
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
            stage('Run startcd.sh on Host') {
                steps {
                    script {
                        // SSH를 통해 호스트에서 startcd.sh 스크립트 실행
                        def host = 'username@10.0.2.15' // 호스트의 사용자 이름과 IP 주소
                        def scriptPath = '/home/username/startcd.sh'
                        sshagent(['myjenkins-10.0.2.15-ssh-key']) {
                            // 스크립트 실행
                            sh "ssh -o StrictHostKeyChecking=no ${host} 'bash ${scriptPath}'"
                        }
                    }
                }
            }
        }
    }
    
    ```
- ###  sudo 명령을 스크립트로 실행할 때 pw를 요구해서 프로세스가 kill되지 않아 생긴 문제
    - port충돌 오류가 계속 발생했다

    ```
    #sudoers 파일 수정 하기
    sudo visudo
    ```
    
    ```
    # 스크립트에 패스워드를 사용하지 않도록 해당 권한 추가
    myuser ALL=(ALL) NOPASSWD: /path/to/your/script.sh
    ```


    ![image](https://github.com/user-attachments/assets/85161c92-d575-4d28-8e6f-90094a93f719)
- ### 이후 정상적으로 실행까지 성공!
    ![image](https://github.com/user-attachments/assets/1bb9e020-96b9-4e93-b6d2-b364c8e728e2)


- ### repo push등의 변경사항 발생 시 (c160b54 커밋)
    ![image](https://github.com/user-attachments/assets/bfbf0311-994e-4670-9131-c34a2639f728)

- ### 이를 감지 (c160b54 커밋 감지)
    ![image](https://github.com/user-attachments/assets/eec46797-95c5-443c-88a4-4e9976f4b65e)


    ![image](https://github.com/user-attachments/assets/960f3545-1ae8-4ee1-b098-a45050751adb)


# 🚀 빌드된 jar파일을 cicd용 ubuntu vm 에서 operation용 vmd으로 scp로 전송하여 실행

- ## 사전 준비사항 : virtual box로 진행 중이므로 vm간의 통신을 구현
    - ### NAT network 생성
      - 도구 -> NAT 네트워크 -> 만들기
      ![image](https://github.com/user-attachments/assets/38e1e28b-5ee9-4369-9d4a-4978170eadb0)

    - ### 각 VM에 네트워크를 할당
      - vm 선택 후 설정 메뉴 선택
      ![image](https://github.com/user-attachments/assets/53f96dc9-cbb5-420d-8fc4-52fd0643e981)

        - 네트워크 메뉴 선택 -> 네트워크 어댑터 종류를 NAT네트워크로 선택, -> 생성했던 NAT네트워크를 설정
      ![image](https://github.com/user-attachments/assets/3cad37b7-7866-4ee5-8b6d-b38779b70b63)

    - ### 각 VM의 ip 확인
      - ifconfig 를 통해 NAT 네트워크 내부 사설 ip 주소를 확인하고 Ping test로 통신 확인
            
        ```
        ifconfig
        ```

        ![image](https://github.com/user-attachments/assets/f9ed5cae-37a9-4335-9e87-e6dd312e310c)

- ## scripts

    - ### ionotify tool 설치

        ```
        sudo apt-get update
        sudo apt-get install inotify-tools
        ```

    - ### change.sh 구성 (실행해 두면 ionotify를 통해 jenkins가 받아온 jar파일의 변경 사항을 감지함)
        ```
        #!/bin/bash
        
        # JAR 파일 경로 설정
        JAR_FILE="/home/username/appjardir/SpringApp-0.0.1-SNAPSHOT.jar"
        
        # 실행할 .sh 파일 경로 설정
        SH_FILE="/home/username/appjardir/operation.sh"
        
        # COOLDOWN 중복 실행 방지 대기 시간 (예: 10초)
        COOLDOWN=10
        LAST_RUN=0
        
        # 파일 수정 감지 및 .sh 파일 실행
        inotifywait -m -e close_write "$JAR_FILE" |
        while read -r directory events filename; do
            CURRENT_TIME=$(date +%s)
        
            # 마지막 실행 후 지정된 시간이 지났는지 확인
            if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
                echo "$(date): $filename 파일이 수정되었습니다."  # 수정 시간 로그 추가
                # .sh 파일 실행
                bash "$SH_FILE"
                # 마지막 실행 시간 업데이트
                LAST_RUN=$CURRENT_TIME
            else
                echo "$(date): 쿨다운 기간 중입니다. 실행하지 않음."
            fi
        done
        
        ```

    - ### operation.sh 구성, scp를 통해 jar파일을 operation server로 전달 후 ssh를 통해 원격 실행 
        ```
        # 파일을 원격 서버에 복사
        scp /home/username/appjardir/SpringApp-0.0.1-SNAPSHOT.jar username@10.0.2.19:/home/username/operation/
        echo "복사완료"
        
        # 복사 후 원격 서버에서 명령어 실행
        ssh username@10.0.2.19 << 'ENDSSH'
        if lsof -i :8999 > /dev/null; then
          # 8999 포트가 사용 중일 경우 이전 프로세스를 종료
          kill -9 $(lsof -t -i:8999)
          echo '정상적으로 종료되었습니다.'
        else
          echo '포트 8999는 사용 중이지 않습니다.'
        fi
        
        
        cd operation
        
        chmod 777 SpringApp-0.0.1-SNAPSHOT.jar
        nohup java -jar SpringApp-0.0.1-SNAPSHOT.jar --server.port=8999 > app.log 2>&1 &
        
        echo '애플리케이션이 실행되었습니다.'
        ENDSSH
        
        echo "운영서버에 jar 전송 및 실행 완료"
        
        ```

    - ### pipeline 설정 변경 (jar파일의 변경을 자동으로 감지하여 실행되므로 직접 jar를 실행하는 stage가 필요 없음)
        - 따라서 repo clone, build, bind mount된 디렉토리에 copy하는것으로 파이프라인을 구성
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
            }
        }
        ```
## 실행 과정

- ### 변경사항 커밋
  ![image](https://github.com/user-attachments/assets/c778955c-ff38-4c98-81be-0eecb718666c)


- ### 이를 감지하고 jenkins가 빌드 진행
  ![image](https://github.com/user-attachments/assets/1daa7baa-2d15-4b49-b0c2-d6ffcc99bcfa)

- ### cicd vm에서 change.sh을 실행하고 있어 jar파일의 변동을 감지.
  ![image](https://github.com/user-attachments/assets/1f6616d4-7943-46ad-a8b3-531f33549d46)

- ### change.sh는 jar파일의 변동 시 scp로 operation vm에 jar를 전송하고 ssh로 실행 명령,
  ![image](https://github.com/user-attachments/assets/2ae9d8bd-f100-4d54-a7ba-ab7c012d2220)


- ### operationserver에서 반영된 변경사항으로 실행되는 모습
  ![image](https://github.com/user-attachments/assets/6f2287bf-745e-41d7-bd36-6044eb2aac78)



