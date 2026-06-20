# InfiniBand(RDMA), RoCEv2

## 배경

AI 학습 환경에서는 GPU가 대량의 데이터를 처리하고 GPU 간 Gradient를 지속적으로 교환해야 한다.

- CPU → Control Path
- DMA → 데이터 이동
- GPU → 연산
- RNIC → 네트워크 전송

즉 CPU는 전송을 설정하고 관리하며, 실제 데이터 이동은 DMA 엔진과 RNIC가 수행한다.

---

## 일반 TCP/IP 통신

RDMA가 없는 일반적인 통신에서는 CPU와 OS Kernel이 여러 번 개입한다.

```text
GPU 1
→ Host RAM 1
→ CPU/Kernel 1
→ Socket Buffer
→ NIC 1
→ Network
→ NIC 2
→ Kernel/Socket Buffer
→ Host RAM 2
→ GPU 2
```

일반 TCP 통신에서는 다음 과정이 발생할 수 있다.

- GPU VRAM → Host RAM
- User Buffer → Kernel Socket Buffer
- Kernel Network Stack 처리
- NIC 송수신
- Kernel Socket Buffer → User Buffer
- Host RAM → GPU VRAM

따라서 CPU 사용량과 지연시간이 증가한다.

---

## RDMA

RDMA(Remote Direct Memory Access)는 한 서버의 RNIC가 다른 서버의 등록된 메모리에 직접 접근하는 기술이다.

```text
Server 1 Application
→ RNIC 1
→ Network
→ RNIC 2
→ Server 2 Registered Memory
```

### 특징

- Kernel Bypass
- Zero-Copy
- Low Latency
- High Throughput

CPU는 전송을 설정하고 관리하며 실제 데이터 이동은 RNIC가 수행한다.

---

## RDMA에서 중요한 개념

### Memory Registration

RDMA에 사용할 메모리를 미리 등록한다.

등록 과정에서 다음 작업이 수행된다.

- 메모리 페이지 고정
- 주소 변환 정보 준비
- 접근 권한 생성
- Local Key(LKey) 발급
- Remote Key(RKey) 발급

### Queue Pair (QP)

RDMA 통신의 작업 큐

- Send Queue
- Receive Queue

### Completion Queue (CQ)

작업 완료 확인

### Work Request (WR)

Application이 RNIC에 제출하는 작업

- RDMA Write
- RDMA Read
- Send
- Receive
- Atomic Operation

---

## RDMA Operation

### RDMA Write

로컬 RNIC가 원격 메모리에 데이터를 쓴다.

```text
Server 1 Memory
→ RNIC 1
→ Network
→ RNIC 2
→ Server 2 Memory
```

원격 CPU는 데이터 이동에 직접 관여하지 않는다.

### RDMA Read

로컬 RNIC가 원격 메모리의 데이터를 읽는다.

```text
Server 2 Memory
→ RNIC 2
→ Network
→ RNIC 1
→ Server 1 Memory
```

### Send / Receive

상대방이 미리 Receive Buffer를 등록해야 하는 메시지 기반 통신 방식이다.

---

## InfiniBand

InfiniBand는 RDMA를 위해 처음부터 설계된 전용 네트워크 패브릭이다.

```text
RDMA = 통신 방식

InfiniBand = RDMA를 위한 전용 네트워크 패브릭
```

### 특징

- 매우 낮은 지연시간
- 높은 대역폭
- Lossless Fabric
- Adaptive Routing
- Congestion Control 내장
- SHARP 지원

### 장점

- 최고 수준의 RDMA 성능
- 대규모 AI/HPC 환경에 적합

### 단점

- 구축 비용이 높음
- 전용 스위치 및 운영 기술 필요

---

## RoCEv2

RoCE(RDMA over Converged Ethernet)는 Ethernet 위에서 RDMA를 사용하는 기술이다.

현재 AI 데이터센터에서 가장 널리 사용되는 RDMA 방식이다.

### 구조

```text
Ethernet
→ IP
→ UDP
→ InfiniBand Transport
→ RDMA
```

즉 Ethernet + IP + UDP 위에 InfiniBand RDMA Transport를 실은 방식이다.

### 특징

- Ethernet 기반 RDMA
- IP Routing 지원
- 기존 데이터센터 네트워크 활용 가능
- 구축 비용 절감

### 주요 기술

#### PFC (Priority Flow Control)

혼잡 시 특정 우선순위 트래픽을 일시적으로 중단한다.

#### ECN (Explicit Congestion Notification)

네트워크 혼잡 상태를 송신 측에 전달한다.

#### DCQCN

RoCE 환경에서 사용하는 대표적인 Congestion Control 메커니즘이다.

---

## Host Memory RDMA

GPU 메모리가 아닌 Host RAM 간 RDMA 통신이다.

```text
GPU 1 VRAM
→ Host RAM 1
→ RNIC 1
→ Network
→ RNIC 2
→ Host RAM 2
→ GPU 2 VRAM
```

일반 TCP보다 CPU 부담은 작지만 Host RAM을 경유한다.

---

## GPUDirect RDMA

GPUDirect RDMA를 사용하면 RNIC가 GPU VRAM에 직접 DMA 접근할 수 있다.

```text
GPU 1 VRAM
→ RNIC 1
→ Network
→ RNIC 2
→ GPU 2 VRAM
```

### 특징

- Host RAM 복사 제거
- CPU 개입 최소화
- PCIe 및 Memory Bandwidth 사용 감소
- 분산 학습 성능 향상

---

## AI 분산 학습에서의 활용

PyTorch Distributed와 NCCL은 GPU 간 Gradient를 교환하기 위해 RDMA 네트워크를 사용한다.

```text
GPU
→ NCCL
→ RDMA
→ InfiniBand / RoCEv2
→ GPU
```

NCCL은 환경에 따라 다음 경로를 선택할 수 있다.

- NVLink
- PCIe P2P
- Shared Memory
- TCP Socket
- InfiniBand Verbs
- RoCE
- GPUDirect RDMA

---

## Control Path vs Data Path

### Control Path

CPU가 담당한다.

- PyTorch 코드 실행
- CUDA Kernel 실행 요청
- DMA Descriptor 생성
- RDMA Queue Pair 관리
- 메모리 등록
- 오류 처리

### Data Path

장치가 담당한다.

- Disk → RAM DMA
- RAM → GPU VRAM DMA
- GPU Kernel 계산
- RNIC → RNIC 네트워크 전송
- GPU VRAM → GPU VRAM GPUDirect RDMA

---

## 정리

- RDMA → CPU와 Kernel 개입을 최소화하는 통신 기술
- InfiniBand → RDMA를 위해 처음부터 설계된 전용 네트워크 패브릭
- RoCEv2 → Ethernet + IP + UDP 기반 RDMA 기술
- GPUDirect RDMA → GPU VRAM 간 직접 데이터 전송
- 현대 AI 클러스터 → NCCL + RDMA + InfiniBand/RoCEv2 기반으로 동작
