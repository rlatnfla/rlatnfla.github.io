---
layout: post
title: "JVM 아키텍쳐와 execution engine"
date: 2026-03-16
categories: [Java]
---

Java의 핵심 철학인 WORA(Write Once, Run Anywhere)을 가능케 하는 JVM의 구조와   
그 중 하나인 실행엔진(Execution Engine)에 대해서 정리하고자 한다.

---

### 1. Java 프로그램의 전체 흐름

Java 소스코드가 실제 CPU 위에서 실행되기 까지는 크게 3가지 단계를 거친다.

1. **컴파일 타임(Compile Time)**   
    - 개발자가 작성한 `.java` 파일은 JDK의 자바 컴파일러인 `javac`에 의해 `.class` 파일로 변환된다.
    - 이 `.class` 파일은 특정 OS의 기계어가 아니라, **JVM이 이해할 수 있는 추상적 명령어 집합인 바이트코드(Bytecode)** 이다.
    - 이 단계 덕분에 코드가 OS에 종속되지 않는다.
2. **로드 타임(Load Time)**
    - JVM이 구동되면 **클래스 로더 서브시스템(Class Loader Subsystem)** 이 동적 로딩을 통해 필요한 `.class`파일을 찾아 JVM 내부의 메모리 영역에 배치한다.
3. **런타임(Run Time)**
    - 메모리에 탑재된 바이트코드(`.class`)를 **실행엔진(Execution Engine)** 이 컴퓨터의 CPU가 이해할 수 있는 기계어로 실시간 번역하여 실행한다.

---

### 2. WORA (Write Once, Run Anywhere)

Java의 WORA 철학은 위의 흐름에서 첫번째 단계인 **컴파일 타임**에서 시작된다.

JDK의 Java 컴파일러인 `javac`는 `.java`파일을 컴파일해 `.class`라는 JVM이 이해하는 바이트코드 파일로 변환시킨다고 하였다.

여기서 `javac`는 본인이 어떤 OS 환경에서 실행되는지에 관계없이 동일한 결과의 `.class`파일을 만들어낸다. (추상화)   

그리고 여기서 OS 환경에 맞게 설치된 JVM이 OS 별로 구체화된 본인의 실행 엔진을 가지고 클래스 로더에 의해 메모리로 올려진 `.class`파일들을 실행한다. (구체화)

타 언어인 `C/C++`과 시스템 콜을 가지고 비교를 해보겠다.

C언어로 스레드를 생성할 때 윈도우 환경에서는 `CreateThread()`를 사용하는 반면, 리눅스 환경에서는 `pthread_create()`를 사용해야 하며 이 때문에 소스코드 내부에서 분기처리를 하는 등, OS 종속적인 개발을 해야한다.

반면 Java의 경우 OS에 관계없이 `new Thread()`한 문장으로 스레드를 생성 할 수 있으며, 이는 **각 OS 환경에 맞게 설치된 JVM이 내부적으로 본인의 OS 최하단 API로 변환** 해주기 때문이다. 이로인해 개발자는 OS 별로 분기처리를 하는 등의 작업이 필요없이 OS 환경에 독립적인 개발을 할 수 있다.

---

### 3. JVM 구조

![JVM Model]({{ site.baseurl }}/assets/images/jvm-model.png)






