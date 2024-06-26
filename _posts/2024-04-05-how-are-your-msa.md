---
layout: post
title: 당신의 MSA 는 안녕하신가요?!
subtitle: 나는 지금 MSA 를 잘 수행하고 있는가?!
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-05/banner.png
categories: spring
tags: [springboot, msa]
---
대부분의 개발자라면 MSA 기반으로 개발을 하고 있으며, 혹은 한번쯤은 들어봤을 것이다.

> **Micro Service Architecture**
작은 역할을 가진 여러 개의 서비스로 구성되있으며, 각 서비스가 독립적으로 개발되고 배포되는 구조
>

Spring 이 Java 기반의 애플리케이션을 구축하는 표준 프레임워크가 되었듯이, MSA 는 Spring 의 기본 아키텍처가 되었다. 그렇다면, 왜 MSA 에 모두 열광하는가?!

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-05/0.png)

MSA 의 장점을 **유연성, 회복성 그리고 확장성**으로 3가지로 분류할 수 있다.

- 유연성
  - 새로운 기능을 신속하게 제공하도록 분리된 서비스를 구성하고, 재배치 가능
  - 서비스 단위가 작을수록 변경에 따른 복잡성을 낮출 수 있고, 배포 및 테스트 시간 단축 가능
- 회복성
  - 오류는 애플리케이션 작은 부분에 국한되어 애플리케이션 전체 장애로 확장되기 전에 억제
- 확장성
  - 모놀로식 애플리케이션은 한 부분이 병목점이 되면 전체를 확장해야하지만, MSA 는 분리된 서비스를 수평적으로 쉽게 분산할 수 있기 때문에 기능 및 서비스를 적절하게 확장 가능
  - 작은 서비스를 확장하는 것은 비용 효율이 높음

그렇다면… 나는 지금 MSA 를 적절하게 사용하고 있는가?! 나의 MSA 를 평가하려고 한다. 평가에 앞서 평가 기준을 먼저 정하겠다. 나는 서비스를 개발하고 운영하는 개발자이기 때문에 내 역할 관점에서 가장 많이 수행하며, 중요한 순서로 가중치를 할당했다. (S, A, B, C, F)

- 서비스 분해도 (35)
- 유연성 및 확장성 (30)
- 모니터링 및 로깅 (20)
- 안전성 (15)

이제 평가를 해보자!

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-05/1.png)

### 평가-1. 서비스 분해도

서비스 분해도는 시스템이 적절한 크기의 서비스로 분해되어 있는지를 평가한다. MSA 에서 각각의 서비스는 하나의 책임 집합을 가지고 있으며, 범위가 좁다. 예를들어, MSA 서비스들은 베터리 전시, 베터리 분석, 인증서 결재와  같이 각각의 책임을 가지고 있다. 마치 객체지향의 5원칙 (**S**OLID) 중 하나인 ‘단일 책임 원칙’ 과 비슷하다.

서비스 분해는 모놀로식 아키텍처에서 MSA 로 전환하는 가장 중요한 단계이며 매우 어렵다. 잘못된 서비스 분해에는 특징이 있다.

서비스 분해를 크게 했을 경우, 하나의 서비스에 너무 많은 책임이 발생된다. 책임이 많은 서비스는 유연성과 확장성을 잃게 된다. 서비스가 무거워 배포 및 테스트 시간이 많이 소요되며, 기능 변경에 따른 영향범위가 커진다. 또한, 수평적 확장에 방해가 된다.

반대로 서비스 분해를 너무 작게 했을 경우, 단순한 CRUD 만을 수행하는 서비스로 전락할 수 있다. 또한, 각각의 서비스들에 대한 상호 의존성이 높아져서 오류에 대한 트랙킹이 어려워지고 개발의 복잡성을 증가시킬 수 있다.

개인적으로 서비스 분해는 처음부터 작은 단위로 나누는 것을 추천하지 않는다. 처음에는 큰 서비스로 시작해서 작게 나누는 것이 좋다. 우선 큰 서비스로 운영하면서 각각의 서비스 또는 기능이 서로 교류하는 방식에 집중하고 설계하는 것이 효과적이다.

현재 **서비스의 분해도를 평가하자면 C** 이다. 이유는 하나의 서비스에서 너무 많은 책임을 가지고 있다고 생각한다. 지금의 서비스에서 더 많은 서비스로 분해가 필요하다고 생각한다.

### 평가-2. 유연성 및 확장성

유연성에서는 서비스가 독립적으로 배포되고 업데이트 될 수 있는지를 평가한다. 결국은 서비스 분해도, 경계분리 라는 단어가 다시 나올 수 밖에 없다. 서비스 간의 경계가 명확하게 정의되어야만 수정에 대한 영향범위가 낮아지며, 배포도 각각 진행될 수 있다.

확장성에서는 트래픽에 유동적인 서비스의 수평적 확장을 평가한다.

현재 서비스의 **유연성 및 확장성을 평가하자면 C** 이다. 이유는 앞서 평가했듯이 경계분리에 대한 숙제가 남아있기 때문에 높은 유연성을 자랑하지는 못한다. 또한, 트래픽에 따른 오토 스케일아웃이 적절히 구성되있지 않아서 확장성이 낮기 때문이다.

### 평가-3. 안전성

안전성에서는 서비스에서 발생한 장애가 다른 서비스에 영향을 주는지에 대한 장애격리를 평가한다.

현재 서비스의 **안전성을 평가하자면 B** 이다.  MSA 를 하고 있기 때문에 서비스의 장애가 다른 서비스에 영향을 주지 않는다. 단, 하나의 서비스에 많은 책임이 부여되었기 떄문에 하나의 서비스의 죽음이 굉장히 크리티컬하다.

### 평가-4. 모니터링 및 로깅

모니터링 및 로깅에서는 실시간 모니터링 및 오류에 대한 추적 가능 유무를 평가한다.

MSA 는 다수의 서비스로 구성되있기 때문에 복잡성이 증가한다. 이러한 복잡성으로 인해 장애를 식별하고 추적하는 것이 어렵다. 서비스의 성능 문제나 장애가 발생할 경우 신속하게 처리하지 않으면 사용자 경험에 부정적인 영향을 줄 수 있다. 그렇기 때문에 모니터링 및 디버깅은 성능을 지속적으로 평가하고 개선하는데 필수적인 요소이다.

현재 서비스의 **모니터링 및 로깅을 평가하자면 A** 이다. 이유는 그라파나가 구축되어있기 때문이다.

그라파나는 프로메테우스를 이용해서 모니터링 데이터를 수집하고 대시보드로 시각화하고 있다. 이를 통해 서비스의 가용성, 성능, 로그 정보를 쉽게 파악할 수 있다. 또한, 그라파나를 이용해서 서비스의 장애나 비정상적인 상황을 식별해서 알람을 주고 있다.

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-05/2.png)

결과적으로 **‘75점 C+’** 으로 평가했다. 조직원의 역량이 뛰어남에도 불구하고, 왜 생각보다 낮은 점수로 채점이 되었을까?!

![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-05/3.png)

첫번째 이유는 **역활에 따른 컨센서스의 불일치** 때문이라고 생각한다. 개발자와 엔지니어의 역할이 다르기 때문에 추구하는 목표와 가치도 물론 다르다. 개발자는 서비스의 독립성과 유연성을 강조하고, 이는 서비스 분해를 촉진하는 경향이 있다. 하지만, 엔지니어 입장에서는 서비스의 분해는 비용의 증가를 의미하기 때문에 서비스의 분해를 환영하지 않을 수 있다.

하지만, 모든 이해관계자가 동일한 목표를 가지는 것은 서비스의 방향성을 명확히하고, 협업을 강화시킬 수 있다. 또한, 신속한 결정 및 작업의 효율을 증가시키기 때문에 비용을 절감할 수 있다.

정말 어려운 일이다.. 토론과 타협과 같은 추상적인 단어를 해결책으로 제시하지만, 현실은 그렇게 아름답지 않다. 현실은 지옥이다.

두번째 이유는 **조직이 추구하는 가치와 MSA 의 가치의 컨센서스 불일치** 때문이라고 생각한다. 현재 조직의 가치는 비용 절감에 많이 집중되있다. 반면 MSA 는 변화의 빠른 대응과 장애의 격리에 집중되있다. 빠른 변화로 인해 성공한 사례의 대표적인 예가 Slack 이다.

Slack은 코로나 팬데믹 이후 급증한 원격 근무 수요에 빠르게 대응하여 MSA 를 통해 성공적으로 성장한 사례이다. Slack은 사용자들에게 안정적이고 효율적인 커뮤니케이션 환경을 제공하며, 글로벌 시장에서도 강력한 입지를 확립할 수 있었다. MSA의 유연성과 안정성은 Slack의 성장과 기업 가치 창출에 중요한 역할을 했다.

MSA 는 장기적인 관점으로 보면 변화에 빠르게 대응할 수 있는 유연성과 안정성을 제공하므로, 기업의 경쟁력을 강화하고 지속 가능한 성장을 이루는 데 중요한 역할을 할 것이다.

그래서 우리의 과제는 컨센서스의 불일치를 해결해야 한다고 생각한다. 우선, **역활에 따른 컨센서스의 불일치를 해결하기 위해서는 명확한 역할 정의와 책임에 대한 할당이 필요**하다. 역할은 다음과 같이 설정할 수 있다.

- **개발자** : 코드 작성, 서비스 기능 개발 및 테스트, 지속적인 기능 개선
- **엔지니어**: 서비스 배포, 인프라 관리, 모니터링 및 문제 해결
- **아키텍트**: 시스템 설계, 마이크로 서비스 분해 전략 수립, 서비스 간 상호 작용 설계

다음으로 책임 할당 매트릭스를 통해 역할 간의 경계를 설정하여 중복이나 갈등을 최소화한다.

> **책임 할당 매트릭스(Responsibility Assignment Matrix, RAM)**
프로젝트 관리를 위해 사용되는 도구로, 프로젝트에 대한 책임과 역할을 명확히 정의하고 할당하는 데 사용
>

책임 할당 매트릭스 (RACI) 는 다음과 같은 네 가지 역할을 정의한다.

- **Responsible (R)**: 작업을 수행하는 사람. 실제로 일을 하는 주체.
- **Accountable (A)**: 작업에 대한 최종 책임을 지는 사람. 작업의 성공 여부를 결정하는 사람.
- **Consulted (C)**: 작업에 대해 조언을 제공하는 사람. 작업에 대한 의견을 제공하거나 정보가 필요한 사람.
- **Informed (I)**: 작업의 진행 상황이나 결과를 알아야 하는 사람. 정보만 제공받는 사람.

**MSA 도입을 위한 책임 할당 매트릭스 예시**

| 작업/역할 | 개발자 (Developer) | 엔지니어 (Engineer) | 아키텍트 (Architect) |
| --- | --- | --- | --- |
| 서비스 설계 | C | C | A |
| 코드 작성 및 테스트 | R | I | C |
| 배포 파이프라인 설계 | I | R | C |
| 모니터링 설정 | I | R | C |
| 서비스 분해 전략 수립 | C | C | A |
| 애플리케이션 배포 | I | R | C |
| 장애 대응 | I | R | C |
| 기술 선택 | C | C | A |
| 교육 및 훈련 | R | R | A |

역할과 책임이 명확히 정의되어 있어, 협업 과정에서의 충돌을 줄이고, 효율적인 의사소통을 촉진할 수 있다.

마지막으로 조직과 MSA 의 컨센서스의 불일치를 해결하려면 소규모 서비스를 분해함으로써 단계적으로 도입이 필요할 것 같다. 파일럿 프로젝트를 통한 비용 절감을 수치화하고 분석하여 장기적으로 비용절감에 기여할 수 있다는 것을 설득해야할 것 같다. 그리고 조직내 지속적인 공유를 통해 MSA 도입의 필요성을 인식시키는 방법으로 컨센서스의 불일치를 해결해야 할 것 같다.

![4.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-05/4.png)

나는 조직에 온지 얼마되지 않아 해당 서비스에 대한 기여도가 매우 낮다. 그리고 조직에 더 빨리 왔다해도 지금보다 더 나은 구조로 개발할 수 있다고 확신할 수 없다. 그렇기 때문에 ‘내가 이런 평가를 해도 될까?’ 생각을 3초 했다. 하지만 해도된다. 나는 된다! 그러하다!

![5.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-05/5.png)
