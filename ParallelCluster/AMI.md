
## Head Node AMI 옵션
<img width="883" height="184" alt="image" src="https://github.com/user-attachments/assets/c36e363b-b081-4d21-b6b5-f822b002925d" />

```
HeadNode:
InstanceType: c6i.xlarge
Networking:
SubnetId: subnet-xxxxx
# 옵션 1: 표준 AMI (기본값, 명시 불필요)
# ParallelCluster가 자동으로 최신 표준 AMI 선택

# 옵션 2: Custom AMI 지정
Image:
CustomAmi: ami-0123456789abcdef0

```


## Compute Node AMI 옵션
<img width="878" height="223" alt="image" src="https://github.com/user-attachments/assets/d3735ce7-10f1-4d5f-9263-ea0baae33e47" />

### ParallelCluster 표준 AMI (기본 방식)
```
특징:

    AWS가 공식 제공 및 관리
    ParallelCluster 버전과 호환성 보장
    Slurm, PBS Pro, SGE 등 스케줄러 사전 설치
    EFA, FSx Lustre 등 AWS 서비스 통합

Scheduling:
Scheduler: slurm
SlurmQueues:
- Name: compute
ComputeResources:
- Name: gpu-nodes
InstanceType: p5.48xlarge
MinCount: 0
MaxCount: 10
# AMI 지정 안 함 = 표준 AMI 자동 사용

```




### DLAMI (Deep Learning AMI) 사용
```
언제 사용하나:

    PyTorch, TensorFlow 등 프레임워크 사전 설치 필요
    CUDA, cuDNN 최신 버전 필요
    GPU 최적화 라이브러리 필요

Scheduling:
SlurmQueues:
- Name: gpu-queue
ComputeResources:
- Name: gpu-compute
InstanceType: p5.48xlarge
MinCount: 0
MaxCount: 8
# DLAMI 지정
Image:
CustomAmi: ami-0a1b2c3d4e5f6g7h8  # DLAMI ID

주의 사항
# DLAMI를 ParallelCluster와 함께 사용 시 호환성 확인 필요
# ParallelCluster 버전과 DLAMI OS 버전 매칭 중요

# 예: ParallelCluster 3.x는 Ubuntu 20.04/22.04, Amazon Linux 2 지원

```



### Custom AMI
```
사용 시나리오:

    특정 소프트웨어 라이선스 사전 설치
    회사 내부 보안 정책 적용
    특수한 커널 설정 또는 드라이버
    부팅 시간 단축 (모든 소프트웨어 사전 설치)

# 1. ParallelCluster 표준 AMI 기반으로 시작
pcluster build-image \
--image-id my-custom-image \
--image-configuration image-config.yaml

# 2. 또는 기존 인스턴스에서 AMI 생성
aws ec2 create-image \
--instance-id i-1234567890abcdef0 \
--name "my-parallelcluster-custom-ami" \
--description "Custom AMI with pre-installed software"


Image:
Os: alinux2  # 또는 ubuntu2004, centos7 등
CustomAmi: ami-custom123456

HeadNode:
Image:
CustomAmi: ami-custom-head123

Scheduling:
SlurmQueues:
- Name: compute
ComputeResources:
- Name: custom-nodes
Image:
CustomAmi: ami-custom-compute456

```




### 표준 AMI + 컨테이너 이미지 (최신 권장 방식)
```
왜 권장되는가:

    이식성: 동일한 컨테이너를 다양한 환경에서 실행
    재현성: 환경 일관성 보장
    유지보수: AMI 관리 부담 감소
    속도: AMI 빌드보다 컨테이너 빌드가 빠름

설정 예시 (Enroot/Pyxis 사용):
Scheduling:
Scheduler: slurm
SlurmSettings:
EnableMemoryBasedScheduling: true
SlurmQueues:
- Name: gpu-queue
ComputeResources:
- Name: gpu-nodes
InstanceType: p5.48xlarge
MinCount: 0
MaxCount: 10
# 표준 AMI 사용 (기본값)
CustomActions:
OnNodeConfigured:
Script: s3://my-bucket/install-enroot.sh

컨테이너 실행:
# Slurm 작업에서 컨테이너 사용
srun --container-image=nvcr.io/nvidia/pytorch:24.01-py3 \
--nodes=4 \
--ntasks-per-node=8 \
python train.py

# 또는 Singularity/Apptainer 사용
srun singularity exec \
docker://nvcr.io/nvidia/pytorch:24.01-py3 \
python train.py


```





```

```





```

```
