
## EFA의 두 가지 구성
```
1. EFA + ENA (Primary ENI)
   ├─► 일반 네트워크 통신 (SSH, HTTP, TCP/IP)
   └─► HPC 고속 통신 (SRD 프로토콜)

2. EFA only (Secondary EFA ENI)
   └─► HPC 고속 통신만 (SRD 프로토콜)

```




## 프로토콜 사용 구분
```
일반 관리 작업:
  SSH, SCP, HTTP, DNS, etc.
  └─► ENA 사용 (eth0만 가능)

HPC 워크로드:
  MPI, NCCL, OpenFabrics
  └─► EFA/SRD 사용 (eth0-eth31 모두 가능)

```


## 
```

```



## 
```
┌─────────────────────────────────────────────────────────┐
│              Network Stack 비교                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  EFA with ENA (efa):                                    │
│  ┌────────────────────────────────────────┐            │
│  │  Application                           │            │
│  └──────────┬─────────────────────────────┘            │
│             │                                           │
│             ├──► TCP/IP Stack (ENA) ──► 일반 통신      │
│             │    - Kernel overhead                     │
│             │    - Protocol processing                 │
│             │                                           │
│             └──► libfabric/SRD (EFA) ──► HPC 통신      │
│                  - OS-bypass                           │
│                  - Direct hardware access              │
│  └────────────────────────────────────────┘            │
│                                                          │
│  EFA-only (efa-only):                                   │
│  ┌────────────────────────────────────────┐            │
│  │  Application                           │            │
│  └──────────┬─────────────────────────────┘            │
│             │                                           │
│             └──► libfabric/SRD (EFA) ──► HPC 통신      │
│                  - OS-bypass                           │
│                  - Direct hardware access              │
│                  - No TCP/IP stack overhead            │
│  └────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘

```




## 옵션 1: 모두 EFA with ENA
```
┌──────────────────────────────────────────────────────────┐
│                  P5.48xlarge Instance                    │
│                                                          │
│  Nitro Card 0  ──► Primary ENI                          │
│                    NetworkCardIndex=0, DeviceIndex=0     │
│                    InterfaceType=efa (EFA with ENA)      │
│                                                          │
│  Nitro Card 1  ──► Secondary ENI #1                     │
│                    NetworkCardIndex=1, DeviceIndex=1     │
│                    InterfaceType=efa (EFA with ENA) ✅   │
│                                                          │
│  Nitro Card 2  ──► Secondary ENI #2                     │
│                    NetworkCardIndex=2, DeviceIndex=1     │
│                    InterfaceType=efa (EFA with ENA) ✅   │
│  ...                                                     │
│                                                          │
│  모든 카드: 일반 통신 + HPC 통신 가능                    │
└──────────────────────────────────────────────────────────┘

```




## 옵션 2: Primary만 EFA with ENA, 나머지 EFA-only
```
┌──────────────────────────────────────────────────────────┐
│                  P5.48xlarge Instance                    │
│                                                          │
│  Nitro Card 0  ──► Primary ENI                          │
│                    NetworkCardIndex=0, DeviceIndex=0     │
│                    InterfaceType=efa (EFA with ENA)      │
│                    - SSH, 일반 통신 가능 ✅              │
│                    - HPC 통신 가능 ✅                    │
│                                                          │
│  Nitro Card 1  ──► Secondary ENI #1                     │
│                    NetworkCardIndex=1, DeviceIndex=1     │
│                    InterfaceType=efa-only                │
│                    - HPC 통신만 가능 ✅                  │
│                                                          │
│  Nitro Card 2-31 ──► Secondary ENI #2-31                │
│                    InterfaceType=efa-only                │
│                    - HPC 통신만 가능 ✅                  │
│                                                          │
│  이것이 일반적인 HPC 구성!                               │
└──────────────────────────────────────────────────────────┘

```








## 
```

```
