
## 그럼 컨테이너가 노드당 8개 , 즉 프로세스 하나당 컨테이너 하나를 실행시키는 거야?
### 네, 정확합니다! 
### 1 프로세스 = 1 컨테이너 인스턴스 = 1 GPU

### node1
```
┌─────────────────────────────────────────────────────────┐
│  HOST OS (Ubuntu)                                        │
│                                                          │
│  프로세스 1 (PID: 10001) ◄─► Enroot Container #1        │
│  └─ Rank 0, GPU 0                                       │
│                                                          │
│  프로세스 2 (PID: 10002) ◄─► Enroot Container #2        │
│  └─ Rank 1, GPU 1                                       │
│                                                          │
│  프로세스 3 (PID: 10003) ◄─► Enroot Container #3        │
│  └─ Rank 2, GPU 2                                       │
│                                                          │
│  프로세스 4 (PID: 10004) ◄─► Enroot Container #4        │
│  └─ Rank 3, GPU 3                                       │
│                                                          │
│  프로세스 5 (PID: 10005) ◄─► Enroot Container #5        │
│  └─ Rank 4, GPU 4                                       │
│                                                          │
│  프로세스 6 (PID: 10006) ◄─► Enroot Container #6        │
│  └─ Rank 5, GPU 5                                       │
│                                                          │
│  프로세스 7 (PID: 10007) ◄─► Enroot Container #7        │
│  └─ Rank 6, GPU 6                                       │
│                                                          │
│  프로세스 8 (PID: 10008) ◄─► Enroot Container #8        │
│  └─ Rank 7, GPU 7                                       │
│                                                          │
│  총: 8개 프로세스 = 8개 컨테이너 = 8개 GPU              │
└─────────────────────────────────────────────────────────┘

```



```

```


```

```




```

```
