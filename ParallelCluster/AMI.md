
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
단계 1: 클러스터 설정 파일 작성

cluster-config.yaml:
Region: us-east-1
Image:
Os: alinux2  # Amazon Linux 2

HeadNode:
InstanceType: c6i.xlarge
Networking:
SubnetId: subnet-12345678
Ssh:
KeyName: my-keypair
Iam:
AdditionalIamPolicies:
- Policy: arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

Scheduling:
Scheduler: slurm
SlurmSettings:
ScaledownIdleTime: 10  # 10분 유휴 후 노드 종료

SlurmQueues:
- Name: gpu-queue
CapacityType: ONDEMAND

ComputeResources:
- Name: p5-nodes
InstanceType: p5.48xlarge
MinCount: 0      # 최소 노드 수
MaxCount: 10     # 최대 노드 수

# 🔑 핵심: Enroot/Pyxis 설치 스크립트
CustomActions:
OnNodeConfigured:
Script: s3://my-bucket/scripts/install-enroot.sh
Args:
- arg1
- arg2

Networking:
SubnetIds:
- subnet-12345678
PlacementGroup:
Enabled: true  # 네트워크 성능 최적화

SharedStorage:
- MountDir: /fsx
Name: fsx-storage
StorageType: FsxLustre
FsxLustreSettings:
StorageCapacity: 1200
DeploymentType: PERSISTENT_2

단계 2: Enroot 설치 스크립트 작성

s3://my-bucket/scripts/install-enroot.sh:
#!/bin/bash
# install-enroot.sh
# 이 스크립트는 각 Compute Node가 부팅될 때 자동 실행됨

set -ex  # 에러 발생 시 중단, 모든 명령 출력

# 로그 파일 설정
LOGFILE="/var/log/enroot-install.log"
exec > >(tee -a ${LOGFILE})
exec 2>&1

echo "=========================================="
echo "Starting Enroot and Pyxis installation"
echo "Date: $(date)"
echo "Hostname: $(hostname)"
echo "=========================================="

# 1. 필수 패키지 설치
echo "Installing prerequisites..."
yum install -y \
jq \
squashfs-tools \
parallel \
libnvidia-container-tools

# 2. Enroot 다운로드 및 설치
echo "Installing Enroot..."
ENROOT_VERSION=3.4.1
cd /tmp

wget -q https://github.com/NVIDIA/enroot/releases/download/v${ENROOT_VERSION}/enroot-${ENROOT_VERSION}-1.x86_64.rpm
wget -q https://github.com/NVIDIA/enroot/releases/download/v${ENROOT_VERSION}/enroot+caps-${ENROOT_VERSION}-1.x86_64.rpm

yum install -y ./enroot-${ENROOT_VERSION}-1.x86_64.rpm
yum install -y ./enroot+caps-${ENROOT_VERSION}-1.x86_64.rpm

# 3. Enroot 설정
echo "Configuring Enroot..."
mkdir -p /etc/enroot
cat > /etc/enroot/enroot.conf <<EOF
# Enroot 설정
ENROOT_RUNTIME_PATH /run/enroot/user-\$(id -u)
ENROOT_CACHE_PATH /tmp/enroot-cache/user-\$(id -u)
ENROOT_DATA_PATH /tmp/enroot-data/user-\$(id -u)
ENROOT_TEMP_PATH /tmp
EOF

# 캐시 디렉토리 생성 (모든 사용자가 접근 가능)
mkdir -p /tmp/enroot-cache
chmod 1777 /tmp/enroot-cache

# 4. Pyxis 빌드 및 설치
echo "Installing Pyxis..."
PYXIS_VERSION=0.16.1
cd /tmp

# Pyxis 소스 다운로드
wget -q https://github.com/NVIDIA/pyxis/archive/refs/tags/v${PYXIS_VERSION}.tar.gz
tar xzf v${PYXIS_VERSION}.tar.gz
cd pyxis-${PYXIS_VERSION}

# Pyxis 빌드 (Slurm 경로 지정)
make install SLURM_DIR=/opt/slurm

# 5. Slurm plugstack 설정
echo "Configuring Slurm plugstack for Pyxis..."

# plugstack 디렉토리 생성
mkdir -p /etc/slurm/plugstack.conf.d

# Pyxis 플러그인 설정 파일 생성
cat > /etc/slurm/plugstack.conf.d/pyxis.conf <<EOF
required /usr/local/lib/slurm/spank_pyxis.so
EOF

# plugstack.conf에 include 추가
PLUGSTACK_CONF="/opt/slurm/etc/plugstack.conf"
if [ ! -f "${PLUGSTACK_CONF}" ]; then
touch "${PLUGSTACK_CONF}"
fi

if ! grep -q "plugstack.conf.d" "${PLUGSTACK_CONF}"; then
echo "include /etc/slurm/plugstack.conf.d/*.conf" >> "${PLUGSTACK_CONF}"
fi

# 6. Slurm 데몬 재시작
echo "Restarting Slurm daemon..."
systemctl restart slurmd

# 7. 설치 확인
echo "Verifying installation..."

# Enroot 버전 확인
if command -v enroot &> /dev/null; then
echo "✓ Enroot installed: $(enroot version)"
else
echo "✗ Enroot installation failed"
exit 1
fi

# Pyxis 플러그인 확인
if [ -f /usr/local/lib/slurm/spank_pyxis.so ]; then
echo "✓ Pyxis plugin installed"
else
echo "✗ Pyxis plugin installation failed"
exit 1
fi

# Slurm 상태 확인
if systemctl is-active --quiet slurmd; then
echo "✓ Slurmd is running"
else
echo "✗ Slurmd is not running"
exit 1
fi

echo "=========================================="
echo "Enroot and Pyxis installation completed!"
echo "Date: $(date)"
echo "=========================================="

exit 0

단계 3: S3에 스크립트 업로드
# 로컬에서 실행
aws s3 cp install-enroot.sh s3://my-bucket/scripts/install-enroot.sh

# 실행 권한 확인 (선택사항)
aws s3api put-object-acl \
--bucket my-bucket \
--key scripts/install-enroot.sh \
--acl bucket-owner-full-control

단계 4: 클러스터 생성
# ParallelCluster 생성
pcluster create-cluster \
--cluster-name my-gpu-cluster \
--cluster-configuration cluster-config.yaml \
--region us-east-1

# 생성 상태 모니터링
pcluster describe-cluster \
--cluster-name my-gpu-cluster \
--region us-east-1

# 완료 대기 (약 10-15분)
# Status: CREATE_COMPLETE 확인

단계 5: Head Node 접속
# SSH 접속
pcluster ssh \
--cluster-name my-gpu-cluster \
--region us-east-1

# 또는 직접 SSH
ssh -i my-keypair.pem ec2-user@<head-node-ip>

단계 6: 학습 스크립트 준비

train.py (FSx Lustre에 저장):
#!/usr/bin/env python3
# /fsx/scripts/train.py

import torch
import torch.distributed as dist
import os

def main():
# 분산 환경 초기화
dist.init_process_group(backend='nccl')

# Rank 정보
rank = dist.get_rank()
world_size = dist.get_world_size()
local_rank = int(os.environ['SLURM_LOCALID'])

# GPU 설정
torch.cuda.set_device(local_rank)
device = torch.device(f'cuda:{local_rank}')

print(f"Rank {rank}/{world_size} - Local Rank {local_rank} - Device: {device}")

# 간단한 텐서 연산 테스트
tensor = torch.ones(10).to(device) * (rank + 1)
print(f"Rank {rank} - Before all_reduce: {tensor}")

# All-Reduce 테스트
dist.all_reduce(tensor, op=dist.ReduceOp.SUM)
print(f"Rank {rank} - After all_reduce: {tensor}")

# 실제 학습 코드는 여기에...

# 정리
dist.destroy_process_group()

if __name__ == '__main__':
main()

스크립트 업로드:
# Head Node에서 실행
mkdir -p /fsx/scripts
vi /fsx/scripts/train.py
# (위 코드 붙여넣기)

chmod +x /fsx/scripts/train.py

단계 7: 작업 제출 및 실행
# Head Node에서 실행

# 방법 1: 직접 srun 실행
srun --partition=gpu-queue \
--nodes=4 \
--ntasks-per-node=8 \
--gpus-per-task=1 \
--container-image=nvcr.io/nvidia/pytorch:24.01-py3 \
--container-mounts=/fsx:/fsx \
python /fsx/scripts/train.py

# 방법 2: sbatch 스크립트 사용
cat > submit_job.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=distributed-training
#SBATCH --partition=gpu-queue
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=8
#SBATCH --gpus-per-task=1
#SBATCH --time=02:00:00
#SBATCH --output=training_%j.out
#SBATCH --error=training_%j.err

# 컨테이너 이미지 지정
export CONTAINER_IMAGE=nvcr.io/nvidia/pytorch:24.01-py3

# 작업 실행
srun --container-image=${CONTAINER_IMAGE} \
--container-mounts=/fsx:/fsx \
python /fsx/scripts/train.py
EOF

# 작업 제출
sbatch submit_job.sh

# 작업 상태 확인
squeue
watch -n 1 squeue  # 실시간 모니터링

단계 8: 실행 과정 모니터링
# 터미널 1: 노드 상태 모니터링
watch -n 2 'sinfo -N -l'

# 터미널 2: 작업 로그 확인
tail -f training_*.out

# 터미널 3: 노드별 리소스 사용량
# (Compute Node에 SSH 접속 후)
nvidia-smi -l 1


```





```

```





```

```
