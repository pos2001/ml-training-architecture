

## 대조: 일반 ENI vs EFA ENI

### 일반 ENI (ENA)
```
하나의 Network Card에 여러 개 가능:

Network Card 0
├─► ENI 1 (Device 0) ✅
├─► ENI 2 (Device 1) ✅
├─► ENI 3 (Device 2) ✅
└─► ENI 4 (Device 3) ✅

예: c5.xlarge = 1 Card, 4 ENI

```




## EFA ENI
```
하나의 Network Card에 1개만 가능:

Network Card 0
└─► EFA ENI 1 ✅

Network Card 1
└─► EFA ENI 2 ✅

Network Card 2
└─► EFA ENI 3 ✅

예: p5.48xlarge = 32 Cards, 32 EFA

```



## 시나리오 1: 일반 인스턴스 (예: c5n.18xlarge)
```
c5n.18xlarge:
- 1개 Network Card
- 최대 15개 ENI 지원
- EFA 지원 가능

가능한 구성:
┌─────────────────────────────────────┐
│    Network Card 0                   │
│                                     │
│    eth0 (Primary ENI - EFA)         │
│    ├─ Interface Type: efa           │
│    └─ EFA + ENA 모두 사용           │
│                                     │
│    eth1 (Secondary ENI - 일반)      │
│    ├─ Interface Type: interface     │
│    └─ ENA만 사용                    │
│                                     │
│    eth2 (Secondary ENI - 일반)      │
│    ├─ Interface Type: interface     │
│    └─ ENA만 사용                    │
│                                     │
│    ... (최대 15개까지)              │
│                                     │
│    1개 EFA + 14개 일반 ENI?         │
└─────────────────────────────────────┘

```







## 
```

```





## 
```

```




## 
```

```
