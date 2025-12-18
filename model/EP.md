
### EP의 의미, EP = Expert Parallelism (전문가 병렬화)
### MoE (Mixture-of-Experts) 모델에서 Expert를 여러 GPU에 분산하는 병렬화 전략입니다.
```
EP8 = Expert Parallelism with 8 GPUs
     = 8개 GPU에 Expert를 분산

예시:
총 64개 Expert가 있을 때

EP8:
- 8개 GPU 사용
- 각 GPU에 8개 Expert (64 ÷ 8 = 8)
- GPU 0: Expert 1-8
- GPU 1: Expert 9-16
- GPU 2: Expert 17-24
- ...
- GPU 7: Expert 57-64

EP16:
- 16개 GPU 사용
- 각 GPU에 4개 Expert (64 ÷ 16 = 4)
- GPU 0: Expert 1-4
- GPU 1: Expert 5-8
- ...
- GPU 15: Expert 61-64

EP32:
- 32개 GPU 사용
- 각 GPU에 2개 Expert (64 ÷ 32 = 2)
- GPU 0: Expert 1-2
- GPU 1: Expert 3-4
- ...
- GPU 31: Expert 63-64

EP64:
- 64개 GPU 사용
- 각 GPU에 1개 Expert (64 ÷ 64 = 1)
- GPU 0: Expert 1
- GPU 1: Expert 2
- ...
- GPU 63: Expert 64

```






### 
```

```







### 
```

```







### 
```

```






### 
```

```






### 
```

```







### 
```

```







### 
```

```






### 
```

```






### 
```

```






### 
```

```

