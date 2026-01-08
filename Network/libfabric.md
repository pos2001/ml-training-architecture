
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
└─────────────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│   libibverbs     │    │   libfabric      │
│                  │    │                  │
│ • ibv_post_send()│    │ • fi_send()      │
└──────────────────┘    └──────────────────┘
        │                       │
        │                       ▼
        │              ┌──────────────────┐
        │              │  libfabric       │
        │              │  EFA Provider    │
        │              │                  │
        │              │ libibverbs를     │
        │              │ 사용하지 않음! ✅ │
        │              └──────────────────┘
        │                       │
        │                       │ Fast Path:
        │                       │ doorbell 직접 쓰기
        │                       │
        │                       │ Control Path:
        │                       │ ioctl()
        │                       │
        └───────────┬───────────┘
                    ▼
          ┌──────────────────────┐
          │    rdma-core         │
          │  (사용자 공간)       │
          │                      │
          │ • EFA provider       │
          │ • verbs 인터페이스   │
          │ • doorbell 매핑 관리 │
          └──────────────────────┘
                    │
                    ▼
          ┌──────────────────────┐
          │ /dev/infiniband/     │
          │      uverbsX         │
          └──────────────────────┘
                    │
                    ▼
          ┌──────────────────────┐
          │  EFA 커널 드라이버    │
          │    (efa.ko)          │
          └──────────────────────┘
                    │
                    ▼
          ┌──────────────────────┐
          │   EFA 하드웨어        │
          │                      │
          │ • Doorbell BAR       │
          │ • Send/Recv Queues   │
          │ • DMA 엔진           │
          └──────────────────────┘

```
