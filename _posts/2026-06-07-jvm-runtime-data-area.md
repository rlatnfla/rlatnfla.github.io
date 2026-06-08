---
layout: post
title: "JVM Runtime Data Area"
date: 2026-06-07
categories: [Java]
---

Java 애플리케이션을 실행하는 순간, JVM은 운영체제로부터 실행에 필요한 메모리를 할당받아 실행 엔진(Execution Engine)과 함께 프로그램을 구동시킨다.

이렇게 OS로부터 할당받은 JVM 고유의 메모리 영역을 **JVM 런타임 데이터 영역(Runtime Data Area)** 이라고 부른다.

이 영역이 어떻게 구성되어있는지, 그리고 우리가 작성한 코드가 이 메모리 위에서 어떤 생명주기를 갖는지 그 구조에 대해 자세히 알아보겠다.

---

## 1. JVM Runtime Data Area 구조

구조 이미지

Runtime Data Area, 줄여서 RDA는 스레드별로 메모리 공간을 공유하는 Shared Area와 그렇지 않은 Thread-Local Area로 나눌 수 있다.

---

## 2. Shared Area

Shared Area는 모든 스레드에서 접근 가능한 공유 자원의 영역이다. 

Java 애플리케이션이 실행되거나 JVM이 시작될 때 단 하나만 생성되며, 모든 스레드가 이 공간을 공유하기 때문에 명확한 특징이 있다.

- **높은 자원 효율성:** 클래스 설계도나 객체들을 스레드마다 따로 만들지 않고 한 곳에 집중시켜 놓기 때문에 메모리 공간을 효율적으로 사용할 수 있다.
- **데이터 공유:** 서로 다른 스레드가 동일한 데이터와 객체를 참조 할 수 있으므로 실시간으로 값을 주고 받을 수 있다.
- **동시성 이슈:** 데이터 공유가 가능하다는 사실은 **필수적으로 Race Condition 과 같은 동시성 이슈를 수반한다.** 
- **메모리 관리:** 많은 스레드에서 해당 영역에 데이터를 생성하고 사용하지만, 더이상 참조되지 않는 데이터가 자동으로 해제되지 않기 때문에 메모리 관리가 필수적이며, 이를 위해 **GC(Garbage Collector)** 가 등장하였다.

이러한 Shared Area는 관리하는 데이터의 목적에 따라 크게 세 가지 구역으로 나뉜다.

### 1) Method Area (Metaspace)

메소드 영역이란 쉽게말해 클래스/인터페이스의 설계도 정보가 할당되는 영역이다.

- **클래스/인터페이스 메타데이터:** 이름, 부모클래스, 접근 제어자등
- **필드 & 메서드 정보:** 변수, 데이터 타입, 메서드 시그니처, 실제 실행될 바이트코드 명령어
- **static 변수:** 특정 인스턴스(`new`)의 소유가 아니라 클래스 자체에 종속되어 프로그램 시작부터 종료까지 살아남아야 하는 자원들이므로 메서드 영역에 할당된다.
- **Runtime Constant Pool (런타임 상수 풀):** 클래스 파일의 Constant Pool을 런타임에 메모리로 올려놓은 자료구조로, 문자열 리터럴뿐 아니라 클래스, 필드, 메서드 참조 정보 등을 저장한다.

>Java 8 HotSpot JVM에서는 클래스 메타데이터와 Runtime Constant Pool은 Metaspace에 저장되지만, static 필드의 실제 값과 String Pool의 문자열 객체는 Heap에 저장되며 GC의 관리 대상이 된다

사실 'Method Area' 라는 용어는 JVM 명세서에 적힌 **논리적인 개념** 에 가깝다. Java 버전에 따라 이 개념이 실제로 메모리에 구현된 형태는 완전히 다르다.

**Java 7 이하 버전**에서는 물리적으로 JVM의 힙(Heap) 영역 내부에 `Perm Gen (Permanent Generation)`이라는 제한된 크기의 메모리 공간을 할당받아 사용하였다.

> Java 7 HotSpot JVM
```text
<----- Java Heap ----->             <--- Native Memory --->
+------+----+----+-----+-----------+--------+--------------+
| Eden | S0 | S1 | Old | Permanent | C Heap | Thread Stack |
+------+----+----+-----+-----------+--------+--------------+
                        <--------->
                       Permanent Heap
S0: Survivor 0
S1: Survivor 1
```

`Perm Gen` 영역은 최대 값이 제한되어있는 Heap 영역에 위치하기 때문에 최대 크기를 넘어서는 순간 컴퓨터의 가용 RAM 용량이 충분하더라도 `OutOfMemoryError: PermGen space`와 함께 애플리케이션이 종료되는 문제가 발생했다.

이러한 문제 때문에 **Java 8 이상 버전**부터 `Perm Gen`이 없어지고 `Metaspace`가 도입되었다. 가장 큰 차이점은 `Metaspace`는 더이상 Heap 영역에 위치하지 않고 크기가 가변적인 **Native Memory Area**에 위치해있다는 점이다.

> Java 8 HotSpot JVM
```text
<----- Java Heap -----> <--------- Native Memory --------->
+------+----+----+-----+-----------+--------+--------------+
| Eden | S0 | S1 | Old | Metaspace | C Heap | Thread Stack |
+------+----+----+-----+-----------+--------+--------------+
```

필요에 따라 확장이 가능한 `Metaspace`의 도입 덕분에 `Perm Gen`에 비해 OOM 발생 가능성이 크게 감소하였다.

다만 이 역시도 무한정 확장되는 것이 아니기 때문에 클래스 메타데이터가 과도하게 증가하면 `OutOfMemoryError: Metaspace`가 발생할 수 있다.

### 2) Heap


