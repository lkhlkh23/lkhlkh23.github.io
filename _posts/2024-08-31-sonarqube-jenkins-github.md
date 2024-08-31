---
layout: post
title: SonarQube + Jenkins + Github 연동 (with PR Decoration)
subtitle: SonarQube + Jenkins + Github
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/banner.png
categories: sonarqube
tags: [sonarqube, github, pr, decoration, jenkins]
---
[이전 포스팅](https://lkhlkh23.github.io/sonarqube/2024/06/23/sonarqube-github.html)에서 `soanrqube + github action + github` 구조를 이용해서 pull request 를 했을 때, 정적분석 결과를 pull request 에 decoration 을 하는 작업을 정리했다.

폐쇠망에서 github enterprise 를 사용하는 조직이기 때문에 네트워크 방화벽으로 인해서 github action 과  통신할 수 없었다. 그래서 내부의 jenkins 를 활용할 수 밖에 없었다. 그러나 내부의 jenkins 도 여러 네트워크 이슈로 머가리가 많이 아팠다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/0.png)

그래서 이번에는 빌드 도구를 github action 이 아닌, jenkins 를 활용하려고 한다. 결국은, `soanrqube + jenkins + github`  구조로 정적분석 결과를 pull request 에 decoration 을 하는 작업을 포스팅하겠다.

### sonarqube flow

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/1.png)

1. `github` 에서 event 가 발생하면, `webhook` 를 통해 jenkins 호출
2. `jenkins` 는 트리거에 의해 설정한 job 실행
3. `jenkins job` 에서 `github checkout → build → sonarqube 정적분석 요청`  순서로 실행
4. `soanrqube` 에서 정적분석 완료 후, 결과를 github 에 전달

### Step.1 github setting

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/2.png)

`github project → settings → Hooks → Add webhook` 를 클릭해서 위와 같은 화면으로 진입

- Payload URL 입력
  - http:// `sonarqube-url` /generic-webhook-trigger/invoke?token= `token`
    - jenkins 에서 generic webhook trigger 플러그인을 이용해서 github 의 webhook 를 감지
    - token 은 webhook 발생 시, jenkins 의 job 중에서 실행할 job 을 식별하기 위한 고유값
- Content type 선택
  - application/json
- Let me select individual events 선택
  - Pull requests, Pushes 선택
    - webhook 를 발생시킬 이벤트 선택
- Add webhook

![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/3.png)

이벤트 발생으로 인한 webhook 의 결과를 `Webhooks → Manage webhook` 에서 확인이 가능하다. Payload 를 보면 jenkins 에 이벤트와 관련된 정보를 전달하고 있다.

- ref : push 당하는 브랜치 (pull request 이벤트 시, payload (X))
- pull_request.head.ref : pull request 요청 브랜치
- pull_request.number : pull request 번호
- pull_request.base.ref : pull request 에서 merge 당할 브랜치

### jenkins setting

![4.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/4.png)

github 에서 이벤트로 발생한 webhook 에 의해 jenkins 에서 job 을 실행하기 위해서는 플러그인이 필요하다.

`Dashboard → Manage Jenkins → Plugins → Available plugins` 메뉴에서 아래 2개의 플러그인을 설치하자.

- Generic Webhook Trigger
- Github

![5.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/5.png)

다음으로는 job 을 생성하자! `Dashboard → New Item` 을 클릭한다! job 의 이름을 입력하고, `Pipeline` 을 선택후, 생성한다.

![6.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/6.png)

- Description
  - job 에 대한 설명을 간략하게 입력
- Github project
  - github repository url 입력 (예 : https://github.com/lkhlkh23/jenkins-webhook.git)

![7.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/7.png)

Build Trigger 에서 `Generic Webhook Trigger` 선택한다.

![8.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/8.png)

Post content parameter 에서 정의한 파라미터들은 Pipeline 스크립트에서 사용할 예정이다. 여기서 정의하는 파라미터의 값은 `Webhook Payload` 이다. 아래 4개의 파라미터를 정의하자!

- push 와 pull request 를 구분하기 위한 파라미터
  - Variable (Pipeline 스크립트에서 식별하기 위한 변수명)
    - ref
  - Expression
    - $.ref
  - Default value
    - none
- pull request 를 요청하는 브랜치
  - Variable
    - prBranch
  - Expression
    - $pull_request.head.ref
- pull request 의 번호
  - Variable
    - pullRequestNo
  - Expression
    - $pull_request.number
- pull request 에서 merge 당할 브랜치
  - Variable
    - prTarger
  - Expression
    - $pull_request.base.ref

![9.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/9.png)

Token 에는 github 의 `Payload URL 에 정의한 token` 의 값을 입력하면 된다. token 은 많은 Jenkins 의 job 중에서 webhook 의해서 트리거될 job 을 식별하기 위해 필요하다.

마지막으로는 webhook 에 의해 job 이 트리거되었을 때, 실행할 Pipeline 스크립트를 작성하면 된다. 코드는 아래와 같다

```groovy
pipeline {
    agent any
    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-amazon-corretto.x86_64'
        PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    def branch = getBranch()
                    echo "Checking out branch: $branch"
                    git branch: branch, credentialsId: 'github-credentialsId', url: 'https://github.enterprise.com/platform-dev/eplat-authentication-api.git'
                }
            }
        }
        stage('Build') {
            steps {
                sh '''
                    ./gradlew clean
                    ./gradlew build -x test --refresh-dependencies
                '''
            }
        }
        stage('SonarQube analysis') {
            steps {
                script {
                    def branch = getBranch()
                    def parameters = getSonarQubeParameters(branch)
                    echo "$parameters"

                    sh """
                        ./gradlew sonar \
                          -Dsonar.token=squ_314f69a9bbd43cda678aeb2e19xxxxxxxxxxx \
                          -Dsonar.projectKey=platform-dev_eplat-authentication-api_xxxx \
                          -Dsonar.projectName=eplat-authentication-api \
                          -Dsonar.host.url=http://10.47.198.116:9000/sonar \
                          -Dsonar.sources=src/main/java \
                          ${parameters} \
                          -Dsonar.java.binaries=build/classes/java/main
                    """
                }
            }
        }
    }
}

def getBranch() {
    return ("$ref" == 'none') ? "$prBranch" : "$ref".split('refs/heads/')[1]
}

def getSonarQubeParameters(branch) {
    def parameters = ""
    if ("$ref" == 'none') {
        parameters += "-Dsonar.pullrequest.key=${pullRequestNo} "
        parameters += "-Dsonar.pullrequest.branch=${branch} "
        parameters += "-Dsonar.pullrequest.base=${prTarget} "
    } else {
        parameters += "-Dsonar.branch.name=${branch} "
    }
    return parameters
}

```

Pipeline 스크립트는 3단계가 순차적으로 진행된다.

- stage(’Checkout’)
  - github repository 에서 브랜치의 코드 checkout

    ```groovy
    git branch: branch, credentialsId: 'github-credentialsId', url: 'https://github.enterprise.com/platform-dev/eplat-authentication-api.git'
    ```

  - credentialsId
    - github 에서 생성한 token
      - `github → Settings → Developer settings → Personal access tokens → Generate new token` 클릭
      - scope
        - repo 전체 선택
        - admin:repo_hook 전체 선택
  - url
    - checkout 할 github url
- stage(’Build’)
  - checkout 한 코드 빌드
- stage(’Sonarqube analysis)
  - 빌드한 코드를 Sonarqube 에게 정적분석 의뢰

    ```groovy
    script {
        def branch = getBranch()
        def parameters = getSonarQubeParameters(branch)
        echo "$parameters"
    
        sh """
            ./gradlew sonar \
              -Dsonar.token=squ_314f69a9bbd43cda678aeb2e19xxxxxxxxxxx \
              -Dsonar.projectKey=platform-dev_eplat-authentication-api_xxxx \
              -Dsonar.projectName=eplat-authentication-api \
              -Dsonar.host.url=http://10.47.198.116:9000/sonar \
              -Dsonar.sources=src/main/java \
              ${parameters} \
              -Dsonar.java.binaries=build/classes/java/main
        """
    }
      
    def getBranch() {
        return ("$ref" == 'none') ? "$prBranch" : "$ref".split('refs/heads/')[1]
    }
    
    def getSonarQubeParameters(branch) {
        def parameters = ""
        if ("$ref" == 'none') {
            parameters += "-Dsonar.pullrequest.key=${pullRequestNo} "
            parameters += "-Dsonar.pullrequest.branch=${branch} "
            parameters += "-Dsonar.pullrequest.base=${prTarget} "
        } else {
            parameters += "-Dsonar.branch.name=${branch} "
        }
        return parameters
    }
    ```

  - `getSonarQubeParameters` 함수는 push 이벤트와 pull_request 이벤트일 때, soanrqube 에 전달할 파라미터가 다르기 때문에 분기처리 필요
    - $ref == none → pull request 이벤트
    - $ref ≠ none → push 이벤트
  - Dsonar.token : sonarqube 에서 생성한 token
  - Dsonar.projectKey : sonarqube 프로젝트 생성 시, 발급되는 키
  - Dsonar.projectName : github repository 이름
  - Dsonar.pullrequest.key : pull request 를 식별하기 위한 목적으로 pull request 번호
  - Dsonar.pullrequest.branch : pull request 하는 브랜치명
  - Dsonar.pullrequest.base : merge 당하는 브랜치명
  - Dsonar.branch.name : push 하는 브랜치명

### sonarqube setting + pr decoration

sonarqube 설정과 github pr decoration 은 이전 포스팅에서 수행해야할 Step 목록만 정의해줄테니, 이전 포스팅을 참고해라! 중복제거!

- [**SonarQube 와 Github 연동 (with PR Decoration)**](https://lkhlkh23.github.io/sonarqube/2024/06/23/sonarqube-github.html)

`Step.0 ~ 6` 를 순차적으로 수행한다. 그리고 빌드 도구로 github action 이 아닌 jenkins 를 사용할 예정이기 때문에 `Step.7` 에서는 github action 에 대한 설정은 제외하고, build.gradle 에 대해서만 아래와 같이 정의한다.

```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.3.0'
	id 'io.spring.dependency-management' version '1.1.5'
	id "org.sonarqube" version "5.0.0.4638"
}

sonar {
	properties {
		property "sonar.projectKey", "soanrqube project 추가하는 과정에서 발급받은 키"
		property "sonar.projectName", "프로젝트명"
	}
}
```

그리고나서 pr decoration 을 위해서 `Step.8` 을 진행하면 끝! 위와 같이 진행하면 `soanrqube + github action + github` 구성이 완료된다! 끝!

### 회고

그 누구도 시키지 않았지만, sonarqube 를 도입하게된 이유는 아래와 같다.

개발과 연관된 KPI 의 평가기준에 sonarqube 의 정적분석 결과가 지표가 되었다. 그러다보니 평소와 같이 개발 하고 평가 직전에 local 에 설치된 sonarqube 정적 분석 결과를 바탕으로 수정하는 경우가 발생했다. 이런 프로세스는 마치 개학 전날에 밀린 일기를 한꺼번에 작성하는 것과 같다고 생각한다. 결국은 코드의 품질에 전혀 도움이 되지 않는다.

그렇다고 pr 을 할때마다 개발자가 local 에 설치된 sonarqube 로 정적 분석을 실행하는 것도 매우 번거롭게 귀찮은 일이다. 그렇기 때문에 이러한 과정을 자동으로 지원해주는 기능이 필요하다고 생각했다. 그래서 도입을 하게되었다.

![10.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-08-31/10.png)

하지만 … 진짜 이유는 업무 개선을 하고, 그에 따른 보상금을 GET 하기 위한 것이다! 기달려라! 20만원!


