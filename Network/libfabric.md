
### libibverbs 직접 사용 ✅
```
애플리케이션
    ↓ ibv_post_send()
libibverbs (사용자 공간 라이브러리)
    ↓ ioctl() 또는 doorbell 쓰기
rdma-core 커널 인터페이스
    ↓ /dev/infiniband/uverbsX
EFA 커널 드라이버 (efa.ko)
    ↓ DMA
EFA 하드웨어

```

### libfabric 사용 (정정됨
```
애플리케이션
    ↓ fi_send()
libfabric (추상화 계층)
    ↓ 내부 호출
libfabric EFA Provider
    ↓ 직접 doorbell 쓰기 (fast path)
    ↓ 또는 ioctl() (control path)
rdma-core의 EFA provider (verbs 인터페이스)
    ↓ /dev/infiniband/uverbsX
EFA 커널 드라이버 (efa.ko)
    ↓ DMA
EFA 하드웨어

```


```
┌─────────────────────────────────────────────────┐
│            애플리케이션 계층                      │
│        (MPI, NCCL, 사용자 애플리케이션)           │
└─────────────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│   libibverbs     │    │   libfabric      │
│   (기본 RDMA)    │    │  (고급 기능)     │
│                  │    │                  │
│ • ibv_post_send()│    │ • fi_send()      │
│ • ibv_post_recv()│    │ • fi_recv()      │
└──────────────────┘    │ • fi_read()      │
        │               └──────────────────┘
        │                       │
        │                       │ (내부 호출)
        │                       ▼
        │              ┌──────────────────┐
        │              │  libfabric의     │
        │              │  EFA Provider    │
        │              │                  │
        │              │ • MR 캐싱        │
        │              │ • 세그멘테이션   │
        │              │ • 순서 보장      │
        │              └──────────────────┘
        │                       │
        │                       │ (rdma-core API 호출)
        │                       │
        └───────────┬───────────┘
                    ▼
          ┌──────────────────────┐
          │    rdma-core         │
          │  (공통 인프라)       │
          │                      │
          │ • EFA provider 포함  │
          │ • /dev/infiniband/   │
          │   uverbsX            │
          └──────────────────────┘
                    │ (ioctl)
                    ▼
          ┌──────────────────────┐
          │  EFA 커널 드라이버    │
          │    (efa.ko)          │
          │                      │
          │ • drivers/infiniband/│
          │   hw/efa/            │
          └──────────────────────┘
                    │ (DMA)
                    ▼
          ┌──────────────────────┐
          │   EFA 하드웨어        │
          │   (NIC)              │
          └──────────────────────┘
```
