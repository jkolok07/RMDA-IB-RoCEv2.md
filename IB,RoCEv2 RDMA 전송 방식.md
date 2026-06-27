# RDMA를 전송하는 방식 : InfiniBand vs RoCEv2

RDMA(Remote Direct Memory Access)는 CPU를 거치지 않고 원격 서버의 메모리에 직접 접근할 수 있도록 하는 통신 기술이다. 하지만 RDMA 자체는 네트워크 프로토콜이 아니기 때문에, 실제 네트워크를 통해 데이터를 전달하기 위한 전송 방식이 필요하다.

대표적인 방식이 **InfiniBand**와 **RoCEv2(RDMA over Converged Ethernet v2)** 이다.

---

## InfiniBand

InfiniBand는 RDMA를 위해 설계된 전용 네트워크이다. RDMA 패킷을 별도의 변환 없이 InfiniBand Fabric으로 전달하며, 스위치 역시 RDMA 패킷을 직접 처리할 수 있다.

```text
Application → libibverbs → RNIC(HCA) → InfiniBand Fabric → Remote RNIC → Remote Memory
```

애플리케이션은 `libibverbs`를 통해 RDMA 요청을 생성하고, RNIC(HCA)가 이를 처리하여 InfiniBand Fabric으로 전송한다. 목적지 RNIC는 CPU를 거치지 않고 등록된 메모리에 직접 데이터를 읽거나 쓸 수 있다.

---

## RoCEv2

RoCEv2는 Ethernet 환경에서 RDMA를 사용할 수 있도록 만든 기술이다. RDMA Transport는 그대로 유지하면서 Ethernet에서 전달할 수 있도록 UDP/IP 헤더를 추가한다.

```text
Application → libibverbs → RNIC → UDP/IP → Ethernet Fabric → Remote RNIC → Remote Memory
```

애플리케이션은 InfiniBand와 동일한 RDMA API를 사용한다. 차이점은 RNIC가 RDMA 패킷을 UDP/IP로 캡슐화하여 Ethernet 네트워크로 전송한다는 점이다. 목적지 RNIC는 UDP/IP 헤더를 제거한 뒤 RDMA 작업을 수행한다.

---

## RoCEv2가 TCP 대신 UDP를 사용하는 이유

RoCEv2는 TCP 대신 UDP를 사용한다. 이유는 RDMA의 **Reliable Connection(RC)** 이 이미 자체적으로 신뢰성을 제공하기 때문이다.

RC는 다음과 같은 기능을 제공한다.

- Packet Sequence Number (PSN)
- ACK
- Retry
- Ordering

이러한 기능은 TCP가 제공하는 신뢰성 기능과 동일한 역할을 수행한다. 따라서 TCP 위에서 RDMA를 사용하면 신뢰성 처리가 중복되어 오버헤드와 지연 시간이 증가할 수 있다.

반면 UDP는 연결 상태나 재전송을 관리하지 않는 단순한 전송 프로토콜이다. RoCEv2에서는 UDP를 RDMA 패킷을 Ethernet에서 전달하기 위한 **캡슐화 계층(Encapsulation Layer)** 으로만 사용하며, 실제 데이터의 신뢰성은 RNIC와 RDMA Transport가 담당한다.

---

## UDP를 사용하는 이유

RoCEv2가 UDP를 사용하는 이유는 단순히 가볍기 때문만은 아니다.

### Layer 3 Routing 지원

RoCEv2는 IP 기반으로 동작하기 때문에 Layer 3 환경에서도 RDMA를 사용할 수 있다. 이를 통해 여러 Rack이나 Subnet으로 구성된 대규모 AI Cluster에서도 RDMA 통신이 가능하다.

### ECMP(Equal-Cost Multi-Path)

UDP 헤더를 사용하면 ECMP를 통해 여러 경로로 트래픽을 분산할 수 있다. Leaf-Spine 구조에서는 이러한 경로 분산이 네트워크 대역폭을 효율적으로 활용하는 데 중요한 역할을 한다.

### 기존 Ethernet 인프라 활용

RoCEv2는 기존 Ethernet 스위치와 네트워크를 그대로 사용할 수 있다. 따라서 별도의 InfiniBand Fabric을 구축하지 않고도 RDMA 기반 AI Cluster를 구성할 수 있다.

---

## InfiniBand vs RoCEv2

| 구분 | InfiniBand | RoCEv2 |
|------|------------|---------|
| Network | InfiniBand Fabric | Ethernet Fabric |
| RDMA 전송 방식 | Native RDMA | RDMA를 UDP/IP로 캡슐화 |
| Routing | InfiniBand Routing | IP Routing |
| Transport | InfiniBand Transport | UDP/IP |
| 신뢰성 | RDMA(RC) | RDMA(RC) |
| TCP 사용 여부 | 사용하지 않음 | 사용하지 않음 |
| 주요 특징 | RDMA 전용 네트워크 | Ethernet 기반 RDMA 네트워크 |

