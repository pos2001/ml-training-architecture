
## 프레임워크와 병렬화 라이브러리의 관계
### 딥러닝 프레임워크와 분산 학습 라이브러리들의 계층 구조를 시각화
### Application = 운전자가 작성한 경로 (여러분의 코드)
```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│              (train.py - Your Training Script)               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              Distributed Training Libraries                  │
├──────────────┬──────────────┬──────────────┬────────────────┤
│     DDP      │   Megatron   │  DeepSpeed   │   Horovod      │
│  (PyTorch)   │   (NVIDIA)   │ (Microsoft)  │    (Uber)      │
├──────────────┼──────────────┼──────────────┼────────────────┤
│  Framework   │  Framework   │  Framework   │  Framework     │
│  PyTorch     │  PyTorch     │  PyTorch     │  Multi-        │
│  Only        │  Only        │  Primary     │  Framework     │
│              │              │              │  (PyTorch, TF, │
│              │              │              │   JAX, Keras)  │
├──────────────┼──────────────┼──────────────┼────────────────┤
│  Strategies  │  Strategies  │  Strategies  │  Strategies    │
│              │              │              │                │
│  ✓ Data      │  ✓ Data      │  ✓ Data      │  ✓ Data        │
│    Parallel  │    Parallel  │    Parallel  │    Parallel    │
│              │              │              │                │
│  ✗ Tensor    │  ✓ Tensor    │  ✓ Tensor    │  ✗ Tensor      │
│    Parallel  │    Parallel  │    Parallel  │    Parallel    │
│              │              │              │                │
│  ✗ Pipeline  │  ✓ Pipeline  │  ✓ Pipeline  │  ✗ Pipeline    │
│    Parallel  │    Parallel  │    Parallel  │    Parallel    │
│              │              │              │                │
│  ✗ ZeRO      │  ✗ ZeRO      │  ✓ ZeRO      │  ✗ ZeRO        │
│              │              │  (Stage 1-3) │                │
│              │              │              │                │
│  ✗ 3D        │  ✓ 3D        │  ✓ 3D        │  ✗ 3D          │
│    Parallel  │    Parallel  │    Parallel  │    Parallel    │
│              │  (DP+TP+PP)  │  (DP+TP+PP)  │                │
└──────────────┴──────────────┴──────────────┴────────────────┘

       │              │              │            │   │   │
       ↓              ↓              ↓            ↓   ↓   ↓
┌─────────────────────────────────────────────────────────────┐
│              Deep Learning Frameworks                        │
├──────────────┬──────────────┬──────────────┬────────────────┤
│   PyTorch    │ TensorFlow   │     JAX      │   Keras/MXNet  │
│              │              │              │                │
│  • DDP       │  • Horovod   │  • Horovod   │  • Horovod     │
│  • Megatron  │              │              │                │
│  • DeepSpeed │              │              │                │
│  • Horovod   │              │              │                │
└──────────────┴──────────────┴──────────────┴────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│            Communication Backends                            │
├──────────────┬──────────────┬──────────────┬────────────────┤
│     NCCL     │     Gloo     │     MPI      │   Framework    │
│   (NVIDIA)   │  (PyTorch)   │  (Standard)  │   Built-in     │
│              │              │              │                │
│  • GPU comm  │  • CPU comm  │  • Horovod's │  • TF's gRPC   │
│  • DDP uses  │  • DDP uses  │    primary   │  • Custom      │
│  • Horovod   │              │    backend   │    backends    │
│    can use   │              │  • NCCL for  │                │
│              │              │    GPU ops   │                │
└──────────────┴──────────────┴──────────────┴────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                 Hardware Layer                               │
│  GPUs (CUDA)  │  TPUs  │  CPUs  │  Network (IB/EFA/RoCE)   │
└─────────────────────────────────────────────────────────────┘


```


## 각 계층의 역할:

### Deep Learning Frameworks (기반): 역할: 딥러닝의 기본 빌딩 블록 제공
```
PyTorch                          TensorFlow
│                                 │
├─ 텐서 연산                      ├─ 텐서 연산
├─ 자동 미분                      ├─ 자동 미분
├─ 신경망 모듈                    ├─ Keras API
└─ 기본 분산 학습 지원             └─ tf.distribute API

제공하는 것:

    텐서 연산 (CPU/GPU)
    자동 미분 (Autograd)
    신경망 레이어 (nn.Module)
    옵티마이저 (SGD, Adam 등)
    단일 GPU 학습


Deep Learning Framework:

    "어떻게 신경망을 만들고 학습할까?"
    텐서, 레이어, 옵티마이저 제공
    단일 디바이스에서 완전히 동작

Framework = 자동차 엔진 (기본 동력)
```





### Distributed Training Libraries (상위):Framework 위에서 분산 학습 전략 구현
```
┌─────────────────────────────────────────────────────────┐
│                    호환성 매트릭스                        │
├──────────────┬──────────────┬──────────────┬────────────┤
│   Library    │   PyTorch    │  TensorFlow  │   Jax      │
├──────────────┼──────────────┼──────────────┼────────────┤
│     DDP      │      ✓       │      ✗       │     ✗      │
├──────────────┼──────────────┼──────────────┼────────────┤
│   Megatron   │      ✓       │      ✗       │     ✗      │
├──────────────┼──────────────┼──────────────┼────────────┤
│  DeepSpeed   │      ✓       │   Limited    │     ✗      │
├──────────────┼──────────────┼──────────────┼────────────┤
│   Horovod    │      ✓       │      ✓       │     ✓      │
└──────────────┴──────────────┴──────────────┴────────────┘

Distributed Training Library:

    "어떻게 여러 GPU/노드에서 효율적으로 학습할까?"
    Framework 위에서 동작
    분산 전략과 통신 패턴 구현
    Framework의 기능을 확장

Distributed Library = 터보차저 (성능 향상 장치)
```


## 기술 스택 조합 예시:
### 예시 1: 중형 모델 (BERT 규모)
```
┌─────────────────────────────────────┐
│      Your Training Script           │
│   (model.py, train.py)              │
└─────────────────────────────────────┘
↓
┌─────────────────────────────────────┐
│    torch.nn.parallel.DDP            │
│    (데이터 병렬화)                   │
└─────────────────────────────────────┘
↓
┌─────────────────────────────────────┐
│         PyTorch Core                │
└─────────────────────────────────────┘
↓
┌─────────────────────────────────────┐
│      NCCL (All-Reduce)              │
└─────────────────────────────────────┘

```



### 예시 2: 대형 모델 (GPT-3 규모)
```
┌─────────────────────────────────────┐
│      Your Training Script           │
│   (megatron/train.py)               │
└─────────────────────────────────────┘
↓
┌─────────────────────────────────────┐
│         Megatron-LM                 │
│  ├─ Tensor Parallelism              │
│  ├─ Pipeline Parallelism            │
│  └─ Data Parallelism                │
└─────────────────────────────────────┘
↓
┌─────────────────────────────────────┐
│         PyTorch Core                │
└─────────────────────────────────────┘
↓
┌─────────────────────────────────────┐
│  NCCL (intra-node) + Custom         │
│  (inter-node communication)         │
└─────────────────────────────────────┘

```



### 메모리 최적화 필요 (DeepSpeed)
```
┌─────────────────────────────────────┐
│      Your Training Script           │
│   (deepspeed.initialize())          │
└─────────────────────────────────────┘
↓
┌─────────────────────────────────────┐
│         DeepSpeed Engine            │
│  ├─ ZeRO Stage 1/2/3                │
│  ├─ Mixed Precision (FP16)          │
│  ├─ Gradient Checkpointing          │
│  ├─ Pipeline Parallelism            │
│  └─ Tensor Parallelism (optional)   │
└─────────────────────────────────────┐
↓
┌─────────────────────────────────────┐
│         PyTorch Core                │
└─────────────────────────────────────┘
↓
┌─────────────────────────────────────┐
│      NCCL + Custom Kernels          │
└─────────────────────────────────────┘

```



### 통합 아키텍처 (최대 규모 모델):
```
┌───────────────────────────────────────────────────────────┐
│                  Training Application                      │
└───────────────────────────────────────────────────────────┘
↓
┌───────────────────────────────────────────────────────────┐
│              DeepSpeed + Megatron-DeepSpeed               │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Megatron Tensor Parallelism (intra-node)           │  │
│  │  Megatron Pipeline Parallelism (inter-node)         │  │
│  │  DeepSpeed ZeRO-3 (optimizer state sharding)        │  │
│  │  DeepSpeed Mixed Precision & Gradient Accumulation  │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
↓
┌───────────────────────────────────────────────────────────┐
│                     PyTorch                                │
└───────────────────────────────────────────────────────────┘
↓
┌───────────────────────────────────────────────────────────┐
│            NCCL (GPU) + Custom Kernels                     │
└───────────────────────────────────────────────────────────┘
↓
┌───────────────────────────────────────────────────────────┐
│  Hardware: Multi-node GPU Cluster with NVLink/InfiniBand  │
└───────────────────────────────────────────────────────────┘

```



### 의존성 관계도:
```
Your Code
│
┌─────────────┼─────────────┐
↓             ↓             ↓
DDP         Megatron      DeepSpeed
│             │             │
└─────────────┼─────────────┘
↓
PyTorch
│
┌─────────────┼─────────────┐
↓             ↓             ↓
NCCL          Gloo           MPI
│             │             │
└─────────────┼─────────────┘
↓
GPU/Network Hardware

```



### 핵심 포인트:
```
1. 프레임워크 = 기반

    PyTorch/TensorFlow는 기본 텐서 연산과 자동 미분 제공
    기본적인 분산 학습 기능 포함

2. 분산 라이브러리 = 확장

    DDP: PyTorch의 공식 데이터 병렬화
    Megatron: 모델 병렬화 전문 (PyTorch 기반)
    DeepSpeed: 종합 최적화 플랫폼 (PyTorch 기반)

3. 조합 가능
# DeepSpeed + Megatron 조합 예시
model = MegatronModel(...)  # Megatron의 텐서 병렬화
engine = deepspeed.initialize(
model=model,
config=ds_config  # DeepSpeed의 ZeRO 최적화
)

```



### 통신 백엔드 = 하부 구조
```

    NCCL: GPU 간 통신 (NVIDIA 최적화)
    Gloo: CPU 통신
    MPI: 범용 병렬 통신

```


### Pyxis + Enroot 중심 아키텍처
```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│              (Your Training Script / Model Code)             │
│                      train.py / main.py                      │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│              Distributed Training Libraries                  │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌─────────────────────────┐   │
│  │   DDP    │  │ Megatron │  │      DeepSpeed          │   │
│  │ (PyTorch)│  │  (NVIDIA)│  │     (Microsoft)         │   │
│  └──────────┘  └──────────┘  └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│              Deep Learning Frameworks                        │
│         PyTorch / TensorFlow / JAX                           │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│            Communication Backends                            │
│  NCCL (GPU-GPU)  │  UCX  │  MPI  │  libfabric (EFA)        │
└─────────────────────────────────────────────────────────────┘
↓
┌═════════════════════════════════════════════════════════════┐
║           Container Runtime Layer (Enroot)                   ║
╠═════════════════════════════════════════════════════════════╣
║  ┌───────────────────────────────────────────────────────┐  ║
║  │          Container Environment                        │  ║
║  │  ┌─────────────────────────────────────────────────┐  │  ║
║  │  │  /usr/local/bin/python                          │  │  ║
║  │  │  /usr/local/lib/python3.10/site-packages/       │  │  ║
║  │  │    ├─ torch/                                     │  │  ║
║  │  │    ├─ deepspeed/                                 │  │  ║
║  │  │    ├─ transformers/                              │  │  ║
║  │  │    └─ ...                                        │  │  ║
║  │  │                                                   │  │  ║
║  │  │  /usr/local/cuda-12.1/                           │  │  ║
║  │  │    ├─ lib64/ (CUDA libraries)                    │  │  ║
║  │  │    ├─ bin/ (nvcc, nvidia-smi)                    │  │  ║
║  │  │    └─ include/                                   │  │  ║
║  │  └─────────────────────────────────────────────────┘  │  ║
║  │                                                         │  ║
║  │  Mounted Volumes:                                      │  ║
║  │  ┌─────────────────────────────────────────────────┐  │  ║
║  │  │  /data      → /lustre/datasets                  │  │  ║
║  │  │  /workspace → /home/user/projects               │  │  ║
║  │  │  /results   → /lustre/checkpoints               │  │  ║
║  │  └─────────────────────────────────────────────────┘  │  ║
║  └───────────────────────────────────────────────────────┘  ║
║                                                              ║
║  Container Image Format: Squashfs                            ║
║  Location: /shared/enroot/pytorch-23.10.sqsh                ║
╚═════════════════════════════════════════════════════════════╝
↓
┌═════════════════════════════════════════════════════════════┐
║              Pyxis (Slurm Container Plugin)                  ║
╠═════════════════════════════════════════════════════════════╣
║  Job Submission Interface:                                   ║
║  ┌───────────────────────────────────────────────────────┐  ║
║  │  srun --container-image=<image>                       │  ║
║  │       --container-mounts=<mounts>                     │  ║
║  │       --container-writable                            │  ║
║  │       --container-remap-root                          │  ║
║  └───────────────────────────────────────────────────────┘  ║
║                                                              ║
║  Responsibilities:                                           ║
║  • Parse container options from Slurm                        ║
║  • Pull/import container images                              ║
║  • Convert to Enroot format (if needed)                      ║
║  • Setup GPU device access                                   ║
║  • Configure network namespaces                              ║
║  • Mount volumes and bind paths                              ║
║  • Launch Enroot containers per task                         ║
╚═════════════════════════════════════════════════════════════╝
↓
┌═════════════════════════════════════════════════════════════┐
║                    Slurm Workload Manager                    ║
╠═════════════════════════════════════════════════════════════╣
║  ┌────────────────┬────────────────┬─────────────────────┐  ║
║  │  slurmctld     │   slurmd       │    Job Scheduler    │  ║
║  │  (Controller)  │   (Daemon)     │    (Resources)      │  ║
║  └────────────────┴────────────────┴─────────────────────┘  ║
║                                                              ║
║  Job Distribution:                                           ║
║  Node 1: Tasks 0-7  (8 GPUs)                                ║
║  Node 2: Tasks 8-15 (8 GPUs)                                ║
║  Node 3: Tasks 16-23 (8 GPUs)                               ║
║  Node 4: Tasks 24-31 (8 GPUs)                               ║
╚═════════════════════════════════════════════════════════════╝
↓
┌─────────────────────────────────────────────────────────────┐
│                   Host System (Minimal)                      │
├─────────────────────────────────────────────────────────────┤
│  Required Components ONLY:                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  • NVIDIA Driver (535.xx or later)                    │  │
│  │  • Slurm (slurmctld, slurmd)                          │  │
│  │  • Pyxis plugin (spank_pyxis.so)                      │  │
│  │  • Enroot runtime (/usr/bin/enroot)                   │  │
│  │  • Network drivers (Mellanox OFED / EFA)              │  │
│  │  • Lustre client (for shared storage)                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  NOT Required:                                               │
│  ✗ Python / pip                                              │
│  ✗ CUDA Toolkit                                              │
│  ✗ cuDNN / NCCL libraries                                    │
│  ✗ PyTorch / TensorFlow                                      │
│  ✗ Any ML frameworks                                         │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│                 Hardware & Network Layer                     │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐   │
│  │   GPUs       │  │   Network    │  │   Storage       │   │
│  │              │  │              │  │                 │   │
│  │ • NVIDIA     │  │ • InfiniBand │  │ • Lustre FS     │   │
│  │   A100/H100  │  │   (200 Gbps) │  │   (parallel)    │   │
│  │ • NVLink     │  │ • AWS EFA    │  │ • NVMe (local)  │   │
│  │ • 8 per node │  │ • RDMA       │  │ • /shared/      │   │
│  └──────────────┘  └──────────────┘  └─────────────────┘   │
└─────────────────────────────────────────────────────────────┘

```
