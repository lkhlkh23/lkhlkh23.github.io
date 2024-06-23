---
layout: post
title: SonarQube 와 Github 연동 (with PR Decoration)
subtitle: SonarQube to Github with PR Decoration
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/banner.png
categories: sonarqube
tags: [sonarqube, github, pr, decoration]
---

현 조직에서 개발 관련된 KPI 에서 높은 점수를 취득하기 위해서는 SonarQube 정적분석 결과 ALL GREEN 을 달성해야 한다. 결국은 SonarQube 를 이용한 코드 품질을 개선하기 위한 목표라고 생각한다.

하지만, 지금 조직에서 사용하는 SonarQube 는 코드 품질보다는 `Bug, Vulnerability, Code Smell` 를 해결하는 수준으로만 활용된다고 생각한다.

> **Bug** : 런타임 오류, 예기치 않은 동작으로 이어질 수 있는 코딩 실수
>
>
> **Vulnerability** : 코드에서 공격에 노출된 지점
>
> **Code Smell** : 코드를 혼란스럽고 유지 관리하기 어렵게 만드는 유지 관리 가능성 이슈
>

하지만, SonarQube 를 더 활용한다면, 코드 품질 개선에도 많은 도움이 될 것으로 생각한다. 그러기위해서는 더 많은 공부가 필요하겠지?!

그래서 이번 포스팅은 SonarQube 와 Github 를 연동해서, Pull Request 에 정적분석 결과를 노출할 수 있는 설정을 해보려고 한다. 먼저 SonarQube 에 대해 간단히 알아보자!

### SonarQube 소개

SonarQube document 를 보면 SonarQube 를 `clean code` 를 제공하는데 도움이 되는 `자체 관리형 자동 코드 검토 도구` 라고 소개하고 있다.

여기서 `clean code`  는 안정적으로 유지보수가 가능한 소프트웨어를 만드는 코드의 표준을 의미한다. 그리고 `자체 관리형 자동 코드 검토 도구` 는 기존 조직의 개발 프로세스에 통합되어 코드의 품질, 보안, 유지보수성을 자동으로 분석하고 검토하는 기능을 제공하는 도구를 의미한다.

결국 SonarQube 는 `개발 프로세스에서 코드를 분석해 품질과 보안을 검토해주는 소프트웨어라`고 정의할 수 있다.

### SonarQube 동작원리

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/0.png)

위 이미지는 SonarQube 의 특징과 동작원리에 대해 잘 표현한 그림인 것 같다. 물론 [SonarQube 공식 document](https://docs.sonarsource.com/sonarqube/latest/) 에서 제공하고 있는 그림이다.

- `sonarlint` 는 코드를 작성할 때, IDE 에 즉각적인 피드백을 제공해서 commit 전에 문제를 찾아 해결할 수 있도록 한다.
- `quality gate` 는 문제가 있는 코드를 출시되지 않도록 유지하며 Clean as You Code 방법론을 통합하는데 도움이 되는 핵심도구이다.
  - green : 표준을 충족하고 출시 가능한 상태
  - red : 해결해야할 문제가 있는 상태

이제는 SonarQube 를 AWS EC2 에 설치하고, Github 와 연동해서 활용해보자! 간단한 이론은 끝났고, 실습이다!

### SonarQube in AWS

**Step.0 docker command**

```bash
# conatiner 정지
docker stop [container id]

# conatainer 확인
docker ps -a

# conatainer 삭제
docker rm [container id]

# 이미지 확인
docker images

# 이미지 삭제
docker rmi [이미지 id]

# container 삭제 전에, 이미지 부터 삭제할 경우
docker rmi --force [이미지 id]

# container 에 명령어 실행
docker-compose exec db psql -U sonar
```

**Step.1 Specifications**

- **t3.midium** 이상 필요
  - cpu : 2 vCPU
  - volume : 20GB
  - ram : 4GB
- security port

  ![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/1.png)



**Step.2 Java 설치**

```bash
# ec2 인터넷 연결 확인
ping -c 4 [google.com](http://google.com/)

# java-21 설치
sudo yum install -y java-21-amazon-corretto-devel

# java 설치 확인
java -version
```

**Step.3 docker & docker-compose 설치**

```bash
# docker 설치
sudo yum install docker -y

# docker 설치 확인
docker --version

# docker 시작
sudo systemctl start docker

# docker 상태 확인
sudo systemctl status docker

# docker-compose 설치
curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) | sudo tee /usr/local/bin/docker-compose > /dev/null

# docker-compose 권한 부여
sudo chmod +x /usr/local/bin/docker-compose

# docker-compose 설치 확인
docker-compose -v
```

**Step.4 sonarqube 설치 & 실행**

아래 docker-compose.yml 파일은 SonarQube 에서 샘플로 제공해주는 코드이다.

- [docker-compose.yml](https://github.com/SonarSource/docker-sonarqube/blob/master/example-compose-files/sq-with-postgres/docker-compose.yml)

```yaml
services:
  sonarqube:
    image: sonarqube:community
    hostname: sonarqube
    container_name: sonarqube
    depends_on:
      db:
        condition: service_healthy
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_conf:/opt/sonarqube/conf
    ports:
      - "9000:9000"

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
    hostname: postgresql
    container_name: postgresql
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonar
    volumes:
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  sonarqube_conf:
  postgresql:
  postgresql_data:
```

`db.healthcheck` 는 postgreSQL 서버가 준비되었는지 확인하기 위한 명령어이다. 10초마다 health check 를 수행하고, health check 에 대한 응답이 5초를 초과하면 timeout 이 발생한다. health check 실패에 대해서 최대 5회까지 재시도를 한다. **필수적인 옵션이 아니기 때문에 삭제해도 무방**하다.

SonarQube 는 내부적으로 ElasticSearch 를 사용하고 있기 때문에 VM Memory 용량이 늘여줘야 한다.

ElasticSearch 가 요구하는 용량 설정을 `vm.max_map_count=262144` 로 해준다.

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/2.png)

```bash
sudo vi /etc/sysctl.conf

# sysctl.conf 에 추가
vm.max_map_count=262144

# 설정 적용 확인
sudo sysctl -p
```

이제는 아래 명령어를 통해 실행시켜 보자!

```bash
sudo docker-compose up
```

**Step.5 콘솔 접속**

`3.38.152.172:9000` 으로 접속 후, 패스워드 변경한다. 초기 계정 정보는 아래와 같다.

- id : admin
- pwd : admin

**Step.6 sonarqube - github 연동**

`import from github` 클릭한다. 아래 입력할 정보는 여기에 있다! 아래 입력한 값은 내 정보 기준이다!

github → settings → developer settings → github apps 화면이동해라!

- configuration name : practice
  - Give your configuration a clear and succinct name. This name will be used at project level to identify the correct configured GitHub App for a project.
- github api url : https://api.github
- github app id
  - Settings → Developer Settings → GitHub Apps → new github app 버튼 클릭
    → app id, client id, client secret, private key 정보 확인 가능
    - github app name : practice-sonarqube
    - homepage url : https://github.com/lkhlkh23
    - callback url : http://3.38.152.172:9000
      - 사용자가 GitHub App 에 권한을 부여한 후 리디렉션될 URL
      - [참고 문서](https://docs.github.com/ko/apps/creating-github-apps/registering-a-github-app/about-the-user-authorization-callback-url)
    - permissions

    ![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/3.png)


  - github app id : 900005
  - client id : Iv23xxxxxxxxxxxxHWk5
  - client secret : 82xxxxefb10xxxxe28exxxx981xxxx92xxxx5564

그러나 아래와 같은 메세지가 나오고 완료되지 않음

![4.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/4.png)

Administration → Configuration → DevOps Platform Integrations 에서 설정이 정상적으로 됬는지 확인하자! 아래와 같이 `Configuration valid` 가 나왔다!

![5.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/5.png)

이번에는 Administration → Configuration → Authentication → Create configuration 버튼 클릭하자!

- client id : Iv23xxxxxxxxxxxxHWk5
- client secret : 82xxxxefb10xxxxe28exxxx981xxxx92xxxx5564
- app id : 900005
- private key

![6.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/6.png)

Test configuration 을 했지만, 실패했다! 아직 끝나지 않았다!

github → settings → developer settings → github apps → 생성한 app → install app 화면으로 이동한다! 그리고나서, `install` 클릭한다! 나는 모든 레파지토리에 대상으로 할 것이다.

![7.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/7.png)

이제 다시 `import from github` 를 클릭하면, 내 계정의 모든 레파지토리가 보인다!

![8.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/8.png)

**Step.7 github action 빌드도구 설정**

`import`를 하면 build 도구를 선택하게 된다! 나는 github actions 를 사용한다! 이제 끝이 보인다!

설명이 아주 자세하게 나와있다! 가이드라인을 따라가보자!

![9.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/9.png)

import 한 프로젝트 → settings → secrets and variables → actions 화면으로 이동해서 `repository secrets`  를 생성한다.

![10.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/10.png)

그리고나서 build.gradle, .github/workflow/build.yml 설정한다! 설정 파일은 아래와 같다!

- [build.gradle](https://github.com/lkhlkh23/sonarqube/blob/main/build.gradle)
- [build.yml](https://github.com/lkhlkh23/sonarqube/blob/main/.github/workflows/build.yml)

그러나, github action 에서 build 실패가 발생했다! 그래서 github action → sonarqube 접근이 되는지 확인해보려고 한다. build.yml 에 아래와 같은 작업을 추가했다.

```bash
- name: Test SonarQube Server Access
  run: curl -v ${{ secrets.SONAR_HOST_URL }}
```

![11.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/11.png)

접근이 되지 않는다! 이유를 찾았다! 내가 보안정책으로 인바운드 9000 포트를 모두 허용하지 않았다! 그래서 보안정책을 수정했다! 이번에는 해당 단계를 통과했다!

이젠 github action 에서 빌드가 성공적으로 끝났다! 그리고나서 sonarqube 를 봤더니! 아래와 같이 나왔다!

![12.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/12.png)

**Step.8 PR Decoration 설정**

이제 github pr 과 sonarqube 의 정적분석을 연결해보려고 한다. 무료로 decoration 을 지원하는 `community-branch-plugin` 플러그인을 설치해보자! docker 로 설치한 sonarqube 내부에 있는 확장 플러그인 디렉토리에 직접 설치가 필요하다.

우선 soanrqube 에 버전을 확인하고, 그에 호환되는 플러그인 버전을 찾아야 한다!

```bash
SonarQube ID information
Server ID: 243B8A4D-AZA6Ao8XPRwEWHcHAv3L
Version: 10.5.1.90531
Date: 2024-06-22
```

`community-branch-plugin` 에 github 의 [release note](https://github.com/mc1arke/sonarqube-community-branch-plugin/releases) 를 확인해보니, 위 버전에 맞는 `community-branch-plugin` 버전은 1.19.0 이다.

```bash
# container id 확인
sudo docker ps -a

# sonarqube 내부로 진입
sudo docker exec -it --user root ${CONTAINER ID} /bin/bash

# 확장 플러그인 디렉토리 이동
cd extensions/plugins/

# community-branch-plugin 설치
wget https://github.com/mc1arke/sonarqube-community-branch-plugin/releases/download/1.19.0/sonarqube-community-branch-plugin-1.19.0.jar

# /conf/sonar.properties 에 설정 추가 (vim, nano not found)
echo "sonar.web.javaAdditionalOpts=-javaagent:./extensions/plugins/sonarqube-community-branch-plugin-1.19.0.jar=web" >> /conf/sonar.properties
echo "sonar.ce.javaAdditionalOpts=-javaagent:./extensions/plugins/sonarqube-community-branch-plugin-1.19.0.jar=ce" >> /conf/sonar.properties

# /conf/sonar.properties 에 설정 제거 
sed -i '/^sonar.web.javaAdditionalOpts=-javaagent:.*sonarqube-community-branch-plugin-1.8.1.jar=web$/d' /opt/sonarqube/conf/sonar.properties
sed -i '/^sonar.ce.javaAdditionalOpts=-javaagent:.*sonarqube-community-branch-plugin-1.8.1.jar=ce$/d' /opt/sonarqube/conf/sonar.properties
```

플러그인을 설치하고, docker 재실행해보자! 그리고나서, Aministration → market place 메뉴로 이동하면 아래와 같이 플러그인이 적용된 것을 확인할 수 있다.

![13.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/13.png)

설정이 끝나면, decoration 이 적용된 것을 확인할 수 있다.

![14.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/14.png)

하지만, 아직도 문제가 있다! 분석 결과의 아이콘이 이상하다. 그리고 URL 클릭 시, [localhost:9000](http://localhost:9000) 으로 이동한다. 기본 URL 에 대한 설정이 필요하다! 아래와 같이 설정하자!

![15.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/15.png)

정말 끝이다!

![16.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-06-23/16.png)

이제 이 작업을 그대로 회사에 적용하려고 한다! 걱정된다. SonarQube 를 설치할 기존 서버는 있을까?! 기존 서버가 있다면 네트워크 작업이 있을텐데, 네트워크 관련 요청 및 협의를 하는데 얼마나 힘이 들까?! 만약 신규 서버로 해야한다면, 신규서버 할당이 될까?! 비용 및 효용성에 대해서 설득하는데 얼마나 많은 노력이 들까?!

기술적인 이슈로 인한 어려움과 피곤함이 아닌, 위와 같은 이슈로 인한 피곤함이 발생했을 때, 그때도 내가 SonarQube 를 도입하고 싶은 마음이 계속 남아있을까?!

글을 마무리하면서 이런 생각이 계속 머리를 떠나지 않았다!
