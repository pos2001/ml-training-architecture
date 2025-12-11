

### 컨테이너 이미지 생성 및 변환 프로세스:
```
┌─────────────────────────────────────────────────────────────┐
│              Container Image Lifecycle                       │
└─────────────────────────────────────────────────────────────┘

Step 1: Build/Pull Docker Image
┌──────────────────────────────────┐
│  Docker Registry                 │
│  nvcr.io/nvidia/pytorch:23.10    │
└──────────────────────────────────┘
↓
docker pull / build
↓
┌──────────────────────────────────┐
│  Local Docker Image              │
│  pytorch:23.10-py3               │
│  Size: ~15 GB (layers)           │
└──────────────────────────────────┘
↓
Step 2: Import to Enroot
↓
enroot import docker://...
↓
┌──────────────────────────────────┐
│  Enroot Squashfs Image           │
│  /shared/enroot/                 │
│    pytorch-23.10.sqsh            │
│  Size: ~8 GB (compressed)        │
│  Format: Read-only, immutable    │
└──────────────────────────────────┘
↓
Step 3: Use in Slurm Jobs
↓
┌──────────────────────────────────┐
│  Pyxis loads image per task      │
│  • Fast: Squashfs = direct read │
│  • Efficient: Shared across jobs │
│  • Secure: Read-only filesystem  │
└──────────────────────────────────┘

```



### Pyxis + Enroot 작동 흐름:
```
┌─────────────────────────────────────────────────────────────┐
│                 Job Execution Flow                           │
└─────────────────────────────────────────────────────────────┘

User submits job:
┌──────────────────────────────────────────────────────────────┐
│  $ srun --nodes=4 --ntasks-per-node=8 \                      │
│         --container-image=/shared/enroot/pytorch.sqsh \      │
│         --container-mounts=/lustre:/data \                   │
│         python train.py                                      │
└──────────────────────────────────────────────────────────────┘
↓
┌──────────────────────────────────────────────────────────────┐
│  Slurm Controller (slurmctld)                                │
│  • Parse job requirements                                    │
│  • Allocate 4 nodes × 8 tasks = 32 tasks                     │
│  • Assign GPUs: 8 per node                                   │
└──────────────────────────────────────────────────────────────┘
↓
┌──────────────────────────────────────────────────────────────┐
│  Slurm Daemon (slurmd) on each node                          │
│  • Receives job allocation                                   │
│  • Invokes Pyxis plugin (spank_pyxis.so)                     │
└──────────────────────────────────────────────────────────────┘
↓
┌──────────────────────────────────────────────────────────────┐
│  Pyxis Plugin (per node)                                     │
│  • Reads --container-image path                              │
│  • Parses --container-mounts options                         │
│  • Prepares container configuration                          │
│  • Calls Enroot for each task                                │
└──────────────────────────────────────────────────────────────┘
↓
┌──────────────────────────────────────────────────────────────┐
│  Enroot Runtime (per task)                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Task 0 (GPU 0)                                        │  │
│  │  1. Mount squashfs image as root filesystem           │  │
│  │  2. Create namespace (mount, PID, network)            │  │
│  │  3. Bind mount: /dev/nvidia0 → container              │  │
│  │  4. Bind mount: /lustre → /data in container          │  │
│  │  5. Set environment variables (CUDA_VISIBLE_DEVICES)  │  │
│  │  6. chroot into container                             │  │
│  │  7. exec: python train.py                             │  │
│  └────────────────────────────────────────────────────────┘  │
│  ... (repeat for tasks 1-7)                                  │
└──────────────────────────────────────────────────────────────┘
↓
┌──────────────────────────────────────────────────────────────┐
│  Training Process (inside container)                         │
│  • Python interpreter from container                         │
│  • PyTorch/CUDA libraries from container                     │
│  • Access to mounted /data (Lustre)                          │
│  • GPU access via passthrough                                │
│  • NCCL communication across nodes                           │
└──────────────────────────────────────────────────────────────┘

```



### 기본 작업 제출:
```
#!/bin/bash
#SBATCH --job-name=gpt-training
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=8
#SBATCH --gpus-per-task=1
#SBATCH --cpus-per-task=12
#SBATCH --time=48:00:00

# Pyxis 컨테이너 옵션
#SBATCH --container-image=/shared/containers/pytorch-23.10.sqsh
#SBATCH --container-mounts=/lustre/datasets:/data:ro,/lustre/checkpoints:/results:rw
#SBATCH --container-writable

# 환경 변수 (컨테이너 내부로 전달)
export NCCL_DEBUG=INFO
export NCCL_IB_DISABLE=0
export MASTER_ADDR=$(scontrol show hostname $SLURM_NODELIST | head -n 1)
export MASTER_PORT=29500

# 학습 실행
srun python train.py \
--model gpt3-175b \
--data-path /data/pile \
--checkpoint-path /results \
--tensor-parallel 8 \
--pipeline-parallel 4

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
