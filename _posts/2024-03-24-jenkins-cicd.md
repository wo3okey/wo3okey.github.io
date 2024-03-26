---
layout: post
title: 간단한 jenkins docker CI/CD 구성
categories: [cs]
tags: [clean architecture, 클린 아키텍처]
---

AWS EC2에 Jenkins를 설치 및 어플리케이션 서버를 위한 추가적인 ec2 서버를 구성하면 좋지만, jenkins를 설치하기에는 프리티어 메모리 문제도 있으며, 추가적인 ec2 인스턴스를 기동시 비용 문제가 발생될 수 있다. 그래서 jenkins는 로컬에서 설치하며 어플리케이션 서버는 ec2로 구성하여 AWS 프리티어에서 가능한 수준의 간단한 CI/CD 구축을 해본다.

# local docker 설치
도커 설치법은 생략한다. 도커 데스크탑까지 설치하면 좋다.

# docker jenkins pull
{% highlight nginx %}
docker pull jenkins/jenkins:lts
{% endhighlight %}

# docker jenkins run
{% highlight nginx %}
docker volume create volume-jenkins
{% endhighlight %}
{% highlight nginx %}
docker network create cicd-net
{% endhighlight %}

{% highlight nginx %}
docker run -d -it \
--name jenkins \
--net cicd-net \
-p 8080:8080 -p 50000:50000 \
-v volume-jenkins:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock \
jenkins/jenkins:lts
{% endhighlight %}

docker 컨테이너 종료 및 삭제 이후에도 설정을 유지하려면 volume 설정을 해주면 좋다. 그리고 docker 엔진은 host OS의 /var/run/docker.sock 아래에 마운트 된 unix 소켓을 사용한다.
docker.sock은 도커 컨테이너 내부에서 데몬과 상호 작용을 할 수 있게 해주는 unix 소캣이다.

만약 권한 문제가 발생한다면 아래와 같이 권한을 주면 된다.
{% highlight nginx %}
sudo chmod 666 /var/run/docker.sock
{% endhighlight %}

# jenkins 접속 및 설치

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-01.png)

`http://localhost:8080/`에 접속하면 설치를 위한 초기 패스워드를 입력이 필요하다.


{% highlight nginx %}
docker exec -i -t --user root jenkins /bin/bash
{% endhighlight %}

docker 컨테이너에 root 권한으로 접속한다.

{% highlight nginx %}
cat /var/jenkins_home/secrets/initialAdminPassword
{% endhighlight %}

초기 패스워드 정보는 위 경로에서 확인할 수 있으며, 해당 값을 입력 후 설치를 진행한다.

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-02.png)

Install suggested plugins 를 클릭하여 기본 플러그인을 설치하면 위 처럼 설치가 진행된다.
이후, 계정 및 간단한 이름을 등록하면 젠킨스를 시작할 수 있다.

# jenkins 설정
## jenkins private key 등록
jenkins에 ssh 접속을 위한 과정이다. jenkins 내에서 자격 등록을 위해 private key를 등록하고, public key를 git에 등록할 예정이다.

{% highlight nginx %}
docker exec -i -t --user root jenkins /bin/bash
{% endhighlight %}

이미 접속해있다면 상관없고, jenkins 컨테이너에 접속 한다.

{% highlight nginx %}
ssh-keygen
{% endhighlight %}

ssh key를 생성한다. 명령어 입력 후 엔터 계속 클릭하면 생성이 된다.

{% highlight nginx %}
cd /root/.ssh/
cat id_rsa
cat id_rsa.pub
{% endhighlight %}

위 경로에 private key `id_rsa` 및 public key `id_rsa.pub` 가 각각 잘 생성되었는지 확인한다.

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-06.png)

{% highlight text %}
Setting > Developer settings > Personal access tokens > tokens(classic) > Generate new token(classic)
{% endhighlight %}

위 경로에서 ssh private 자격을 등록한다. `SSH Username with private key` 타입으로 등록하며, ID는 아무거나 작성해도 좋다. 나는 ssh로 작성했다. 그리고 private key를 작성할때는 `id_rsa` 파일의 전체 정보를 넣으면 된다. 반드시 `-----BEGIN OPENSSH PRIVATE KEY-----` 문구 부터 `-----END OPENSSH PRIVATE KEY-----` 까지 끝까지 다 넣어주도록 한다.

## github jenkins public key 등록
방금 등록한 private ssh와 대응되도록 git에는 public ssh를 등록한다.
{% highlight text %}
github > 등록할 레포지토리 > Settings > Deploy keys > Add deploy key
{% endhighlight %}

이름은 편하게 짓고, jenkins 컨테이너의 `id_rsa.pub` 파일 내용을 그대로 넣어주면 정상적으로 등록된다.

## github developer token 발급
github에서 아래 경로로 이동 후 토큰을 발급한다. 이제는 jenkins에서 git에 연결하고 개발 관련 컨트롤을 하기 위함이다.

{% highlight text %}
전체 메뉴의 Setting > Developer settings > Personal access tokens > tokens(classic) > Generate new token(classic)
{% endhighlight %}

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-03.png)

토큰의 권한은 레포를 관리할 수 있는 최소한만 줘도 된다. 토큰 이름은 jenkins로 지었으며, 토큰 만료기간은 따로 설정하지 않았다.

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-04.png)


신규 발급된 토큰의 private key는 잘 저장해두고 복사한다.

## jenkins git-hub credentials 등록
마찬가지로 jenkins에서도 해당 토큰을 연결해야한다. 서로 상호 연결한다고 생각하면 된다.

{% highlight text %}
Jenkins 대시보드 > Jenkins 관리 > Security > Credentials > global > Add Credentials
{% endhighlight %}

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-05.png)

`Username with password`로 설정 후 본인의 git 아이디를 username에, 패스워드는 로그인할때 패스워드가 아닌, 방금 위에서 발급받은 token을 넣는다! git auth 정책이 바뀌어 일반 비밀번호를 넣으면 auth 에러가 발생하니, 반드시 유의할것!!!

## jenkins docker-hub credentials 등록
도커 이미지를 push 하기 위해서는 jenkins에 도커 자격 증명이 필요하다. github 계정 등록과 같은 방법으로 docker hub 계정을 등록하면 된다.

{% highlight text %}
Jenkins 대시보드 > Jenkins 관리 > Security > Credentials > global > Add Credentials
{% endhighlight %}

`Username with password`로 설정 후 본인의 docker 아이디를 username에, 패스워드를 입력 후 ID에는 아무값이나 구분자로 넣어준다.

## ec2 jenkins public key 등록
application 서버로 사용할 ubuntu ec2에도 public key가 있다. 해당 부분은 아래 ec2 생성 후 다시 보는걸 추천한다. 이후 ec2에 접속 후 `authorized_keys` 를 ssh 경로에 만들어, jenkins의 public key를 등록한다. jenkins 컨테이너 내에서 만든 `id_rsa.pub` 파일을 그대로 넣으면 된다.

jenkins 컨테이너의 public key 내용 복사
{% highlight nginx %}
cat /root/.ssh/id_rsa.pub
{% endhighlight %}

ec2 서버에 authorize_keys 파일에 내용 붙여넣기
{% highlight nginx %}
vi ~/.ssh/authorized_keys
{% endhighlight %}

## ssh agent 플러그인 설치
![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-07.png)
![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-09.png)

jenkins에서 직접 붙여쓰는 자격 증명과 별개로 ec2에 직접 ssh로 연결하기 위한 `SSH Agent` 플러그인, docker 파이프라인을 위한 `Docker Pipeline`을 available plugins에서 설치한다.
Docker Pipeline은 미리 설치해놔서 검색 사진이 없다ㅎ

## jenkins docker in docker 설치
jenkins 내에서 docker image build를 통해 docker hub에 push 하는 작업을 위해 docker 설치가 필요하다.

{% highlight nginx %}
docker exec -i -t --user root jenkins /bin/bash
{% endhighlight %}
터미널에서 jenkins 컨테이너 접속 후 아래 명렁어에 따라 docker를 설치를 한다. docker 컨테이너 내에서 docker를 설치해야하는 아이러니한 상황이긴하다. 이를 docker in docker라고 한다.

{% highlight nginx %}
apt-get update -y
{% endhighlight %}
{% highlight nginx %}
apt-get install -y\
    ca-certificates \
    curl \
    gnupg \
    lsb-release
{% endhighlight %}
{% highlight nginx %}
mkdir -p /etc/apt/keyrings
{% endhighlight %}
{% highlight nginx %}
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
{% endhighlight %}
{% highlight nginx %}
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
{% endhighlight %}
{% highlight nginx %}
apt-get update -y
{% endhighlight %}
{% highlight nginx %}
apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
{% endhighlight %}

{% highlight nginx %}
docker --version
{% endhighlight %}

docker 설치가 완료 되었으면 버전이 뜰 것이다. 확인해보면 된다. 여기까지 진행했으면, jenkins에서 설정해야하는 사항은 끝이다. 

# app server ec2

## ec2 생성
springboot application server를 띄울 ec2 하나를 띄운다. 나는 ubuntu os로 만들었으며, 어떠한 것으로 하든 상관 없다. 그 외 EC2 생성 방법은 생략하며, 키페어를 저장하고 위치만 잘 확인해두도록 한다.

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-08.png)

다만 보안그룹 설정 시, 인바운드 설정이 필요하다. nginx를 통해 들어올 TCP 80포트 및 application server의 blue green 배포를 통해 들어올 8080, 8081 포트는 열어둬야 한다. 또한 ec2에 ssh 연결을 해야하므로 ssh 연결을 위한 22번 포트도 반드시 열어둬야한다. https 443 포트는 선택이다.

## ec2 docker 설치
ec2에는 docker-compose 명령어를 통해 blue green 배포를 한다. 따라서 docker 및 docker-compose 설치가 필요하다. 설치 내용은 생략한다.

## deploy.sh
jenkins에서 ssh 연결 후 실행시킬 deploy 쉘 스크립트가 필요하다. blue가 켜져있으면 green을, green이 켜져있으면 blue를 배포하는 간단한 스크립트이다. ec2의 root 경로에 작성했다.

{컨테이너명} 부분은 필히 본인이 사용할 이름으로 변경해야한다! ex) {컨테이너명} -> test-container

{% highlight shell %}
# 1
EXIST_BLUE=$(docker-compose -p {컨테이너명}-blue -f docker-compose.blue.yaml ps | grep Up)

if [ -z "$EXIST_BLUE" ]; then
    docker-compose -p {컨테이너명}-blue -f ~/docker-compose.blue.yaml up -d
    BEFORE_COMPOSE_COLOR="green"
    AFTER_COMPOSE_COLOR="blue"
    BEFORE_PORT_NUMBER=8081
    AFTER_PORT_NUMBER=8080
else
    docker-compose -p {컨테이너명}-green -f ~/docker-compose.green.yaml up -d
    BEFORE_COMPOSE_COLOR="blue"
    AFTER_COMPOSE_COLOR="green"
    BEFORE_PORT_NUMBER=8080
    AFTER_PORT_NUMBER=8081
fi

echo "${AFTER_COMPOSE_COLOR} server up(port:${AFTER_PORT_NUMBER})"

# 2
for cnt in {1..10}
do
    echo "서버 응답 확인중..(${cnt}/10)";
    UP=$(curl -s http://localhost:${AFTER_PORT_NUMBER}/actuator/health | grep 'UP')
    if [ -z "${UP}" ] 
        then
	    sleep 10
	    continue       
        else
            break
    fi
done

if [ $cnt -eq 10 ]
then
    echo "서버가 정상적으로 구동되지 않았습니다."
    exit 1
fi

# 3
sudo sed -i "s/${BEFORE_PORT_NUMBER}/${AFTER_PORT_NUMBER}/" /etc/nginx/conf.d/service-url.inc
sudo nginx -s reload
echo "Deploy Completed!!"

# 4
echo "$BEFORE_COMPOSE_COLOR server down(port:${BEFORE_PORT_NUMBER})"
docker-compose -p {컨테이너명}-${BEFORE_COMPOSE_COLOR} -f docker-compose.${BEFORE_COMPOSE_COLOR}.yaml down
{% endhighlight %}

## docker-compose
위 쉴에서 구동할 blue, green docker-compose 파일이 필요하다. docker-compose.blue.yaml, docker-compose.green.yaml 각각의 compose 파일을 작성했다. deploy.sh 파일과 같은 ec2 root 경로에 작성했다.

{% highlight shell %}
version: '3.1'
 
services: 
  api:
    image: {dockerID}/{컨테이너명}
    container_name: {컨테이너명}-blue
    environment:
      - LANG=ko_KR.UTF-8
      - UWSGI_PORT=8080
    ports:
      - '8080:8080'
{% endhighlight %}

{% highlight shell %}
version: '3.1'
 
services: 
  api:
    image: {dockerID}/{컨테이너명}
    container_name: {컨테이너명}-green
    environment:
      - LANG=ko_KR.UTF-8
      - UWSGI_PORT=8081
    ports:
      - '8081:8080'
{% endhighlight %}

# jenkins pipeline
{% highlight shell %}
pipeline {
    agent any

    environment {
        gitCredential = 'git'
        dockerCredential = 'docker-hub'
        sshCredential = 'ssh'
    }

    stages {
        stage('Prepare') {
            steps {
                echo 'Clonning Repository'
                git branch: 'master', 
                url: '{연결할 git 레포지토리 http url}',
                credentialsId: gitCredential
            }
            
            post {
                success { 
                    echo 'Successfully Cloned Repository'
                }
                failure {
                    error 'This pipeline stops here...'
                    
                }
            }
        }

        stage('Bulid Gradle') {
          steps {
            echo 'Bulid Gradle'
            dir('.'){
                sh './gradlew clean build'           
			}
          }
          post {
          	success { 
              echo 'Successfully Project Build'
            }
            failure {
              error 'This pipeline stops here...'
            }
          }
        }
        
        stage('Bulid Docker') {
          steps {
            echo 'Bulid Docker'
            script {
                dockerImage = docker.build("{도커아이디}/{컨테이너명}", "--platform linux/x86_64 .")
            }
          }
          post {
	        success { 
              echo 'Successfully Docker Build'
            }
            failure {
              error 'This pipeline stops here...'
            }
          }
        }

        stage('Push Docker') {
          steps {
            echo 'Push Docker'
            script {
                docker.withRegistry('', dockerCredential) {
                    dockerImage.push() 
                }
            }
          }
          post {
          	success { 
              echo 'Successfully Docker push'
            }
            failure {
              error 'This pipeline stops here...'
            }
          }
        }
        
		stage('Docker Run') {
            steps {
                echo 'Pull Docker Image & Docker Image Run'
                sshagent (credentials: [sshCredential]) {
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@{ec2 IP주소} './deploy.sh'"
                }
            }
        }
    }
}
{% endhighlight %}

위 스크립트에 `{XXX}` 해놓은 부분은 본인이 사용하고자 하는 명명에 맞게 변경해서 사용해야한다. 그대로 사용하면 절대안된다ㅎ 
참고로 jenkins 컨테이너 내에 pull 받은 git repo는 /var/jenkins_home/workspace 경로에 있다.

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-10.png)

중간중간 에러 있어서 9트에 완료..