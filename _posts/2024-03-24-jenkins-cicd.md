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

# jenkins github 연동
## jenkins private ssh 등록
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

## github public ssh 등록
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

## jenkins git credentials 등록
마찬가지로 jenkins에서도 해당 토큰을 연결해야한다. 서로 상호 연결한다고 생각하면 된다.

{% highlight text %}
Jenkins 대시보드 > Jenkins 관리 > Security > Credentials > global > Add Credentials
{% endhighlight %}

![jenkins-cicd]({{site.url}}/assets/images/posts/jenkins-cicd/jenkins-cicd-05.png)

`Username with password`로 설정 후 본인의 git 아이디를 username에, 패스워드를 입력 후 ID에는 아무값이나 구분자로 넣어준다.

# 파이프라인
{% highlight shell %}
pipeline {
    agent any

    environment {
        dockerCredential = 'docker-hub'
        dockerImage = ''
        slack_channel = '#deploy_jenkins'
    }

    stages {
        stage('Prepare') {
            steps {
                echo 'Clonning Repository'
                git branch: 'master', 
                url: 'https://github.com/wo3okey/spring-challenge-lecture',
                credentialsId: 'my-github'
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
                dockerImage = docker.build("wookey/sparta-lecture", "--platform linux/x86_64 .")
            }
          }
          post {
	        success { 
              echo 'Successfully Docker Build'
            }
            failure {
              error 'This pipeline stops here...'
              slackSend (channel: slack_channel, color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
            }
          }
        }

        stage('Push Docker') {
          steps {
            echo 'Push Docker'
            script {
                docker.withRegistry( '', dockerCredential) {
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
                sshagent (credentials: ['ssh']) {
                    // sh "ssh -o StrictHostKeyChecking=no ubuntu@3.39.96.29 'docker pull wookey/sparta-lecture'" 
                    // sh "ssh -o StrictHostKeyChecking=no ubuntu@3.39.96.29 'docker run -d --name sparta-lecture -p 8080:8080 wookey/sparta-lecture'"
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@3.39.96.29 './deploy.sh'"
                }
            }
        }
    }
}
{% endhighlight %}

## 한줄 리뷰
> 유연하고 변경에 강한 소프트웨어를 개발하기 위해, 클린 아키텍처는 고수준의 도메인 중심에서 경계를 명확히 할 수 있는 방법을 가이드 한다.