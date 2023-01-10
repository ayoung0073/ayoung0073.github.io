---
layout: default
title: Zookeeper
parent: Kafka
nav_order: 1
---

KAFKA
{: .label }

**분산 코디네이션 서비스를 제공하는 오픈소스 프로젝트로 직접 어플리케이션 작업을 조율하는 것을 쉽게 개발할 수 있도록 도와주는 도구**
API를 이용해 동기화나 마스터 선출 등의 작업을 쉽게 구현할 수 있게 해준다.

안전성과 성능을 인정 받아 Hadoop, HBase, Storm, Kafka 등 다양한 Open Source Project에 이용되고 있다.

![주키퍼 아키텍처 ([링크](https://ssup2.github.io/theory_analysis/ZooKeeper/))](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F7f4ebc9f-d268-4520-9d9b-cc42e2926f45%2FUntitled.png?id=50e8ebed-b0dc-4409-8265-1389981a6294&table=block&spaceId=b897da56-853b-41fc-a775-9d2efbbfb74a&width=2000&userId=09d07b42-3885-4a64-9d43-62494b058a86&cache=v2)

분산 Coordinator는 분산 시스템의 일부분이기 때문에 동작을 멈춘다면 분산 시스템이 멈출 수 있다.

주키퍼는 안정성을 확보하기 위해 클러스터로 구축한다. (과반수 이상의 데이터를 기준으로 consistency를 맞추기 떄문에 홀수로 구축한다. → 짝수로 구성하는 경우, 쿼럼을 형성할 때 비례적으로 노드 수가 더 필요하므로 잘 사용되지 않는다.)

- **Server Cluster : Client 구조**

  ***Server*** 는 일정시간 Client로 부터 ping을 받지 못하면 Client/Network Issue가 발생했다고 간주하고 해당 Session을 종료한다. (Client는 Server에게 주기적으로 ping을 전송하여 Client의 동작을 알려준다.)
  ***Client*** 는 Server로부터 ping 응답을 받지 못하면 Server/Network Issue가 발생했다고 간주하고 다른 Server에게 연결을 시도한다.


### **주키퍼 서버의 구조**

- **Request Processor (Leader 서버만 이용)**: ZNode write 요청 처리 (모든 write 요청은 Leader가 처리)
- **Atomic Broadcast**: 이를 통해 **Transaction**을 생성&전파하여 ZNode write가 모든 Server에 적용되도록 한다. Server의 Atomic Broadcast를 통해서 ZNode 생성/변경/삭제 동작은 Client 입장에서는 Sequence Consistency, Atomicity 특징을 보인다.

  > 단계: Leader(Propose) → Follower(Ack) → Leader(Commit)

    - **Quorum**
      Leader가 새로운 트랜잭션을 수행하기 위해서는 자신을 포함하여 과반수 이상의 서버의 합의를 얻어야 한다. 과반수의 합의를 위해 필요한 서버들을 Quorum이라고 한다. Ensemble을 구성하는 서버의 수가 5개라면, Quorum은 3개의 서버로 구성이 된다.
      - Propose: Quorum을 구성하는 서버들에게 트랙잭션을 수행해도 되는지 여부를 요청하는 과정을 의미한다.
      - Ack: Quorum을 구성하는 (팔로워) 서버들은 Leader로 부터 Propose 요청을 받으면, 트랙잭션을 수행해도 된다는 Ack 응답을 Leader에게 전송한다.
      - Commit: 모든 Quorum으로 부터 Ack를 받으면, Leader는 트랙잭션을 처리하라는 Commit 명령을 broadcast 형태로 모든 Follower에 전파한다. ZooKeeper에서는 Commit 명령을 전달할 때, ZAB(ZooKeeper Atomic Broadcast) 알고리즘을 사용한다. Atomic Broadcast는 broacast 방식 중 하나로, 멀티 프로세스 시스템에서 모든 프로세스에게 동일한 순서로 메시지가 전달된다는 것을 의미한다. 과반수(쿼럼) 노드에서 변경을 저장하면 리더는 업데이트 연산을 commit하고, 클라이언트는 업데이트가 성공했다는 이벤트를 받게 된다.
- **In-memory DB**: ZNode 정보가 저장되어 있다. Local Filesystem에 In-memory DB의 Replication을 구성할 수 있다.

분산 애플리케이션들이 각각 클라이언트가 되어 주키퍼 서버들과 커넥션 맺은 후 **상태 정보** 등을 주고 받는다.

**상태 정보**

- 주키퍼의 ZNode라고 불리는 곳에 key-value 형태로 저장한다. 지노드에 저장된 것을 이용해 분산 애플리케이션들이 데이터를 주고 받는다.
- Znode: 자식 노드를 가지고 있는 계층형 구조로 구성되어 있다. ← 일반 컴퓨터의 파일이나 폴더 개념

  ![Untitled](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1b7d433a-6443-4049-8495-def1a467c849%2FUntitled.png?id=5b40c011-f320-4663-ae6a-b1ff4a9bad4c&table=block&spaceId=b897da56-853b-41fc-a775-9d2efbbfb74a&width=2000&userId=09d07b42-3885-4a64-9d43-62494b058a86&cache=v2)

    - **Persistent Node (영속)**: 영구 저장소
    - **Ephermeral Node(임시)** : Client가 종료되면 사라진다.
    - **Sequence Node(연속형)**: 생성 시 뒤에 숫자가 붙는다.
- 데이터 변경 등에 대한 유효성 검사 등을 위해 버전 번호를 관리한다. (데이터가 변동될 때마다 지노드의 버전 번호가 증가)
- 주키퍼에 저장되는 데이터는 모두 메모리에 저장되어 처리량이 매우 크고 속도 또한 빠름

### 주키퍼의 특징

- 단순하다.
- 기능이 풍부하다.
- 고가용성을 제공한다.
- 느슨하게 연결된 상호작용을 할 수 있도록 해준다.
- 오픈소스 라이브러리다.

**카프카에서 주키퍼의 역할은?**

- 브로커를 관리하고 조정하는데 사용한다.
- 서버의 상태를 감지하기 위해 사용되며 새로운 토픽이 생성되었을 때, 토픽의 생성과 소비에 대한 상태를 저장한다.
- 주키퍼와 카프카는 대규모 환경에서는 다른 서버에 두는 편이 좋다. 주키퍼 앙상블은 홀수로 구성되어 과반수 이상이 장애가 발생하면 중단되고, 카프카는 그렇지 않아도 되기 때문에 다른 서버에 두는 것을 권장한다.

![[링크](https://blog.neonkid.xyz/302)](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fcc9d397d-37bb-42f2-a9d9-5b0baab125bf%2FUntitled.png?id=0b94a200-645d-4934-a463-00fa201279d7&table=block&spaceId=b897da56-853b-41fc-a775-9d2efbbfb74a&width=2000&userId=09d07b42-3885-4a64-9d43-62494b058a86&cache=v2)

[링크](https://blog.neonkid.xyz/302)

카프카에서는 메시지를 주고 받는 것 외에는 아무것도 하지 않아, 브로커 상태를 저장하지 않아 상태 관리를 위한 주키퍼를 사용하는 것이다.

Producer와 Consumer는 카프카의 브로커 정보를 가지고 있다. 동적으로 브로커의 상태가 변경(scale-out 등)되는 경우, 이를 주키퍼가 Producer와 Consumer에게 알려준다.

보통 카프카 메시지 브로커와 주키퍼는 1:1로 구성되며 만약 Producer가 특정 카프카 브로커로 메시지를 생산했을 때 이를 실패하면 주키퍼가 다른 정상 서버로 Producer와 Consumer에게 브로커를 알리는 방식으로 운영된다.

카프카는 대용량 데이터 분산 처리 플랫폼이지만 오직 데이터 분산 처리에만 그 역할을 하고 있다. 그래서 분산 시스템 설계 시 필요한 데이터 공유나 락 등은 카프카가 아닌 주키퍼를 통해 별도로 관리되어지고 있다라고 보면 된다.

Producer는 Zookeeper를 통해 Multi Node Message Broker의 ID를 전달 받는다.

![Untitled](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F7c458a00-d621-4290-9233-f1620e15c70e%2FUntitled.png?id=0119e1bd-a5f5-4d88-8ddc-9748998c4c03&table=block&spaceId=b897da56-853b-41fc-a775-9d2efbbfb74a&width=2000&userId=09d07b42-3885-4a64-9d43-62494b058a86&cache=v2)

ZooKeeper는 Kafka 브로커를 관리하고 조정하는데 사용된다. Kafka Message Broker와 1:1로 구성된다. 즉 3 Node Message Broker를 구성할 경우 마찬가지로 3 Node Zookeeper를 구성한다.

ZooKeeper 서비스는 주로 생산자와 소비자에게 Kafka Message Broker의 신규 생성 여부 또는 Kafka Message Broker의 실패를 알리는데 사용된다. 브로커의 존재 또는 실패와 관련하여 Zookeeper에 받은 알림에 따라 Producer와 Consumer는 결정을 내리고 다른 브로커와 작업을 조정한다.

## 🔗 참고 링크
- [ZooKeeper](https://ssup2.github.io/theory_analysis/ZooKeeper/)
- [Kafka(Zookeeper) 아키텍처](https://waspro.tistory.com/645)
- [[Kafka] Kafka broker, Zookeeper의 역할](https://velog.io/@anjinwoong/Kafka-Kafka-broker-Zookeeper%EC%9D%98-%EC%97%AD%ED%95%A0)