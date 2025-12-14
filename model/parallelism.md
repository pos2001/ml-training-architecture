





### Data Parallelism (데이터 병렬화)
```
모든 라이브러리 지원 ✓


# 각 GPU가 모델 전체 복사본을 가짐
# 데이터를 배치로 나누어 병렬 처리

GPU 0: Model Copy → Batch 1 → Gradients
GPU 1: Model Copy → Batch 2 → Gradients
GPU 2: Model Copy → Batch 3 → Gradients
GPU 3: Model Copy → Batch 4 → Gradients
         ↓
    All-Reduce (그래디언트 평균)
         ↓
    모든 GPU에서 동일하게 파라미터 업데이트

```





### Tensor Parallelism (텐서 병렬화)
```
Megatron ✓, DeepSpeed ✓
# 모델의 레이어를 GPU들에 분할
# 각 GPU가 레이어의 일부분을 처리

        Input
          ↓
┌─────────────────────┐
│   Linear Layer      │
│   (4096 x 4096)     │
└─────────────────────┘
     Split by columns
          ↓
┌────┬────┬────┬────┐
│GPU0│GPU1│GPU2│GPU3│
│1024│1024│1024│1024│
└────┴────┴────┴────┘
     All-Gather
          ↓
       Output


```






### Pipeline Parallelism (파이프라인 병렬화)
```
Megatron ✓, DeepSpeed ✓

# 모델을 레이어 단위로 여러 GPU에 분할
# 마이크로배치로 파이프라인 처리

GPU 0: Layers 1-3   →  Micro-batch 1 → Micro-batch 2 → ...
         ↓
GPU 1: Layers 4-6   →  Micro-batch 1 → Micro-batch 2 → ...
         ↓
GPU 2: Layers 7-9   →  Micro-batch 1 → Micro-batch 2 → ...
         ↓
GPU 3: Layers 10-12 →  Micro-batch 1 → Micro-batch 2 → ...

```






### ZeRO (Zero Redundancy Optimizer)
```
DeepSpeed만 지원 ✓

# 옵티마이저 상태, 그래디언트, 파라미터를 분할하여 메모리 절약

┌─────────────────────────────────────────────────────────┐
│  ZeRO Stage 1: Optimizer State Partitioning             │
│  - 옵티마이저 상태를 GPU들에 분할 (4x 메모리 절약)         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  ZeRO Stage 2: + Gradient Partitioning                  │
│  - 그래디언트도 분할 (8x 메모리 절약)                      │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  ZeRO Stage 3: + Parameter Partitioning                 │
│  - 모델 파라미터도 분할 (Nx 메모리 절약, N=GPU 수)        │
└─────────────────────────────────────────────────────────┘

```







### 
```

```
