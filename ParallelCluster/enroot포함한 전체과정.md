

## 전체 워크플로우 비교
### 시나리오 B: ECR 사용 (실제 프로덕션 권장)
```
┌──────────────────────────────────────────────────────────────┐
│ 1. 사전 준비: 컨테이너 이미지를 ECR에 푸시                    │
└──────────────────────────────────────────────────────────────┘
↓
┌────────────────────────────────────────────────┐
│ 개발자 워크스테이션 또는 CI/CD 파이프라인      │
│                                                 │
│ # NGC에서 베이스 이미지 가져오기               │
│ docker pull nvcr.io/nvidia/pytorch:24.01-py3   │
│                                                 │
│ # (선택) Dockerfile로 커스터마이징             │
│ docker build -t my-training-image .            │
│                                                 │
│ # ECR 로그인                                    │
│ aws ecr get-login-password --region us-east-1 \│
│   | docker login --username AWS \               │
│     --password-stdin 123456789012.dkr.ecr...   │
│                                                 │
│ # ECR에 태그 및 푸시                            │
│ docker tag my-training-image \                  │
│   123456789012.dkr.ecr.us-east-1.amazonaws.com/│
│   my-training:v1.0                              │
│                                                 │
│ docker push 123456789012.dkr.ecr.us-east-1...  │
└────────────────────────────────────────────────┘
↓
┌──────────────────────────────────────────────────────────────┐
│ Amazon ECR (Private Registry)                                │
│ └─ 123456789012.dkr.ecr.us-east-1.amazonaws.com/             │
│    my-training:v1.0                                          │
│                                                               │
│ ✅ VPC 내부 통신 (빠름)                                      │
│ ✅ IAM 기반 인증                                             │
│ ✅ 버전 관리                                                 │
└──────────────────────────────────────────────────────────────┘
↓
┌──────────────────────────────────────────────────────────────┐
│ 2. 작업 실행 시: Compute Node에서 ECR 이미지 사용            │
└──────────────────────────────────────────────────────────────┘
↓
┌────────────────────────────────────────────────┐
│ Compute Node (p5.48xlarge)                     │
│                                                 │
│ ① Enroot가 ECR에서 이미지 가져오기            │
│    enroot import \                              │
│      docker://123456789012.dkr.ecr...          │
│                                                 │
│ ② Docker 이미지 → Enroot sqsh 변환            │
│    위치: /tmp/enroot-cache/my-training.sqsh    │
│    크기: ~10-20GB (압축됨)                     │
│                                                 │
│ ③ sqsh 파일에서 컨테이너 생성                 │
│    enroot create my-training.sqsh              │
│                                                 │
│ ④ 컨테이너 시작 및 작업 실행                  │
│    enroot start my-training python train.py    │
└────────────────────────────────────────────────┘
```
## 상세 단계별 가이드
### 단계 1: ECR 리포지토리 생성
```
# ECR 리포지토리 생성
aws ecr create-repository \
--repository-name my-training \
--region us-east-1 \
--image-scanning-configuration scanOnPush=true

# 출력:
# {
#     "repository": {
#         "repositoryArn": "arn:aws:ecr:us-east-1:123456789012:repository/my-training",
#         "registryId": "123456789012",
#         "repositoryName": "my-training",
#         "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training"
#     }
# }
```


### 단계 2: 컨테이너 이미지 준비 및 푸시
옵션 A: NGC 이미지를 그대로 ECR에 복사
```
# 1. NGC에서 이미지 다운로드
docker pull nvcr.io/nvidia/pytorch:24.01-py3

# 2. ECR 로그인
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin \
123456789012.dkr.ecr.us-east-1.amazonaws.com

# 3. 이미지 태그
docker tag nvcr.io/nvidia/pytorch:24.01-py3 \
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training:pytorch-24.01

# 4. ECR에 푸시
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training:pytorch-24.01
```


### 옵션 B: 커스텀 이미지 빌드 후 ECR에 푸시
```
# Dockerfile
FROM nvcr.io/nvidia/pytorch:24.01-py3

# 추가 패키지 설치
RUN pip install --no-cache-dir \
transformers==4.36.0 \
datasets==2.16.0 \
wandb==0.16.0 \
deepspeed==0.12.6

# 학습 스크립트 복사
COPY train.py /workspace/train.py
COPY requirements.txt /workspace/requirements.txt

# 작업 디렉토리 설정
WORKDIR /workspace

# 환경 변수 설정
ENV NCCL_DEBUG=INFO
ENV PYTHONUNBUFFERED=1

# 기본 명령어
CMD ["python", "train.py"]
```

```
# 1. 이미지 빌드
docker build -t my-training:v1.0 .

# 2. ECR 로그인
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin \
123456789012.dkr.ecr.us-east-1.amazonaws.com

# 3. 태그 및 푸시
docker tag my-training:v1.0 \
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training:v1.0

docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training:v1.0
```



### 단계 3: ParallelCluster 설정 (ECR 인증 포함)
```
# cluster-config.yaml
Region: us-east-1
Image:
Os: alinux2

HeadNode:
InstanceType: c6i.xlarge
Networking:
SubnetId: subnet-12345678
Ssh:
KeyName: my-keypair
Iam:
AdditionalIamPolicies:
# ECR 접근 권한 추가
- Policy: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
- Policy: arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

Scheduling:
Scheduler: slurm
SlurmSettings:
ScaledownIdleTime: 10

SlurmQueues:
- Name: gpu-queue
CapacityType: ONDEMAND

ComputeResources:
- Name: p5-nodes
InstanceType: p5.48xlarge
MinCount: 0
MaxCount: 10

# Enroot + ECR 인증 설정 스크립트
CustomActions:
OnNodeConfigured:
Script: s3://my-bucket/scripts/install-enroot-ecr.sh

# ECR 접근 권한
Iam:
AdditionalIamPolicies:
- Policy: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

Networking:
SubnetIds:
- subnet-12345678
PlacementGroup:
Enabled: true

SharedStorage:
- MountDir: /fsx
Name: fsx-storage
StorageType: FsxLustre
FsxLustreSettings:
StorageCapacity: 1200
DeploymentType: PERSISTENT_2
```


### 단계 4: Enroot + ECR 인증 설치 스크립트
```
#!/bin/bash
# install-enroot-ecr.sh
# ECR 인증을 포함한 Enroot/Pyxis 설치

set -ex

LOGFILE="/var/log/enroot-install.log"
exec > >(tee -a ${LOGFILE})
exec 2>&1

echo "=========================================="
echo "Starting Enroot, Pyxis, and ECR setup"
echo "Date: $(date)"
echo "=========================================="

# 1. AWS CLI 설치 (Amazon Linux 2는 기본 설치됨)
echo "Checking AWS CLI..."
aws --version

# 2. 필수 패키지 설치
echo "Installing prerequisites..."
yum install -y \
jq \
squashfs-tools \
parallel \
libnvidia-container-tools

# 3. Enroot 설치
echo "Installing Enroot..."
ENROOT_VERSION=3.4.1
cd /tmp

wget -q https://github.com/NVIDIA/enroot/releases/download/v${ENROOT_VERSION}/enroot-${ENROOT_VERSION}-1.x86_64.rpm
wget -q https://github.com/NVIDIA/enroot/releases/download/v${ENROOT_VERSION}/enroot+caps-${ENROOT_VERSION}-1.x86_64.rpm

yum install -y ./enroot-${ENROOT_VERSION}-1.x86_64.rpm
yum install -y ./enroot+caps-${ENROOT_VERSION}-1.x86_64.rpm

# 4. Enroot 설정
echo "Configuring Enroot..."
mkdir -p /etc/enroot

cat > /etc/enroot/enroot.conf <<EOF
ENROOT_RUNTIME_PATH /run/enroot/user-\$(id -u)
ENROOT_CACHE_PATH /tmp/enroot-cache/user-\$(id -u)
ENROOT_DATA_PATH /tmp/enroot-data/user-\$(id -u)
ENROOT_TEMP_PATH /tmp
EOF

# 5. 🔑 ECR 인증 설정 (핵심!)
echo "Configuring ECR authentication for Enroot..."

# ECR 리전 설정
ECR_REGION="us-east-1"
ECR_REGISTRY="123456789012.dkr.ecr.${ECR_REGION}.amazonaws.com"

# Enroot credentials 디렉토리 생성
mkdir -p /root/.config/enroot

# ECR 로그인 헬퍼 스크립트 생성
cat > /usr/local/bin/enroot-ecr-login.sh <<'EOFSCRIPT'
#!/bin/bash
# ECR 인증 토큰 가져오기
ECR_REGION="${1:-us-east-1}"
ECR_REGISTRY="${2}"

# AWS CLI로 ECR 토큰 가져오기
ECR_PASSWORD=$(aws ecr get-login-password --region ${ECR_REGION})

# Enroot credentials 파일에 저장
mkdir -p ~/.config/enroot
cat > ~/.config/enroot/.credentials <<EOF
machine ${ECR_REGISTRY}
login AWS
password ${ECR_PASSWORD}
EOF

chmod 600 ~/.config/enroot/.credentials
echo "ECR credentials updated for ${ECR_REGISTRY}"
EOFSCRIPT

chmod +x /usr/local/bin/enroot-ecr-login.sh

# 초기 ECR 로그인 실행
/usr/local/bin/enroot-ecr-login.sh ${ECR_REGION} ${ECR_REGISTRY}

# 6. ECR 토큰 자동 갱신 (12시간마다)
echo "Setting up ECR token refresh cron job..."
cat > /etc/cron.d/enroot-ecr-refresh <<EOF
# ECR 토큰은 12시간 유효, 6시간마다 갱신
0 */6 * * * root /usr/local/bin/enroot-ecr-login.sh ${ECR_REGION} ${ECR_REGISTRY} >> /var/log/enroot-ecr-refresh.log 2>&1
EOF

# 7. Pyxis 설치
echo "Installing Pyxis..."
PYXIS_VERSION=0.16.1
cd /tmp

wget -q https://github.com/NVIDIA/pyxis/archive/refs/tags/v${PYXIS_VERSION}.tar.gz
tar xzf v${PYXIS_VERSION}.tar.gz
cd pyxis-${PYXIS_VERSION}

make install SLURM_DIR=/opt/slurm

# 8. Slurm plugstack 설정
echo "Configuring Slurm plugstack..."
mkdir -p /etc/slurm/plugstack.conf.d

cat > /etc/slurm/plugstack.conf.d/pyxis.conf <<EOF
required /usr/local/lib/slurm/spank_pyxis.so
EOF

PLUGSTACK_CONF="/opt/slurm/etc/plugstack.conf"
if [ ! -f "${PLUGSTACK_CONF}" ]; then
touch "${PLUGSTACK_CONF}"
fi

if ! grep -q "plugstack.conf.d" "${PLUGSTACK_CONF}"; then
echo "include /etc/slurm/plugstack.conf.d/*.conf" >> "${PLUGSTACK_CONF}"
fi

# 9. Slurm 환경 변수 설정 (ECR 인증 전파)
cat >> /opt/slurm/etc/slurm.conf <<EOF
# Enroot/ECR 설정
PropagateResourceLimitsExcept=MEMLOCK
EOF

# 10. Slurm 재시작
echo "Restarting Slurm daemon..."
systemctl restart slurmd

# 11. 설치 확인
echo "Verifying installation..."

if command -v enroot &> /dev/null; then
echo "✓ Enroot installed: $(enroot version)"
else
echo "✗ Enroot installation failed"
exit 1
fi

if [ -f /usr/local/lib/slurm/spank_pyxis.so ]; then
echo "✓ Pyxis plugin installed"
else
echo "✗ Pyxis installation failed"
exit 1
fi

if [ -f ~/.config/enroot/.credentials ]; then
echo "✓ ECR credentials configured"
else
echo "✗ ECR credentials not configured"
exit 1
fi

echo "=========================================="
echo "Installation completed successfully!"
echo "Date: $(date)"
echo "=========================================="

exit 0
```


단계 5: ECR 이미지를 사용한 작업 제출
# Head Node에서 실행

# 방법 1: srun으로 직접 실행
srun --partition=gpu-queue \
--nodes=4 \
--ntasks-per-node=8 \
--gpus-per-task=1 \
--container-image=123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training:v1.0 \
--container-mounts=/fsx:/fsx \
python /fsx/scripts/train.py

# 방법 2: sbatch 스크립트
cat > submit_ecr_job.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=ecr-training
#SBATCH --partition=gpu-queue
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=8
#SBATCH --gpus-per-task=1
#SBATCH --time=02:00:00
#SBATCH --output=training_%j.out
#SBATCH --error=training_%j.err

# ECR 이미지 URI
ECR_IMAGE="123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training:v1.0"

# 작업 실행
srun --container-image=${ECR_IMAGE} \
--container-mounts=/fsx:/fsx \
python /fsx/scripts/train.py
EOF

sbatch submit_ecr_job.sh

Enroot의 이미지 변환 과정 상세
Docker 이미지 → sqsh 파일 변환
┌──────────────────────────────────────────────────────────────┐
│ 1. Enroot import 명령 실행                                    │
└──────────────────────────────────────────────────────────────┘
enroot import docker://123456789012.dkr.ecr.../my-training:v1.0
↓
┌──────────────────────────────────────────────────────────────┐
│ 2. ECR에서 Docker 이미지 레이어 다운로드                      │
└──────────────────────────────────────────────────────────────┘
ECR Registry
├─ Layer 1: base OS (Ubuntu 20.04)          - 100 MB
├─ Layer 2: CUDA 12.3                       - 2 GB
├─ Layer 3: PyTorch 2.2                     - 3 GB
├─ Layer 4: 추가 패키지                     - 500 MB
└─ Layer 5: 학습 스크립트                   - 10 MB

총 크기: ~5.6 GB (압축됨)
↓
┌──────────────────────────────────────────────────────────────┐
│ 3. 레이어 압축 해제 및 병합                                   │
└──────────────────────────────────────────────────────────────┘
/tmp/enroot-import-XXXXXX/
├─ rootfs/                    ← 모든 레이어 병합
│  ├─ bin/
│  ├─ lib/
│  ├─ usr/
│  ├─ opt/conda/              ← PyTorch 설치
│  └─ workspace/              ← 학습 스크립트
└─ config.json                ← 메타데이터
↓
┌──────────────────────────────────────────────────────────────┐
│ 4. SquashFS 파일 시스템으로 압축                              │
└──────────────────────────────────────────────────────────────┘
mksquashfs rootfs/ my-training.sqsh \
-comp zstd \              ← Zstandard 압축
-Xcompression-level 3

결과: /tmp/enroot-cache/user-0/my-training+v1.0.sqsh
크기: ~8-10 GB (압축률에 따라)
↓
┌──────────────────────────────────────────────────────────────┐
│ 5. sqsh 파일 캐싱                                             │
└──────────────────────────────────────────────────────────────┘
/tmp/enroot-cache/user-0/
└─ my-training+v1.0.sqsh      ← 다음 실행 시 재사용

sqsh 파일에서 컨테이너 실행
┌──────────────────────────────────────────────────────────────┐
│ 1. Enroot create 명령 (Pyxis가 자동 호출)                     │
└──────────────────────────────────────────────────────────────┘
enroot create --name my-container my-training+v1.0.sqsh
↓
┌──────────────────────────────────────────────────────────────┐
│ 2. sqsh 파일을 읽기 전용 파일 시스템으로 마운트               │
└──────────────────────────────────────────────────────────────┘
/run/enroot/user-0/my-container/
├─ rootfs/                    ← sqsh 마운트 (읽기 전용)
│  ├─ bin/
│  ├─ lib/
│  └─ ...
└─ overlay/                   ← 쓰기 가능 레이어 (tmpfs)
└─ upper/
↓
┌──────────────────────────────────────────────────────────────┐
│ 3. Enroot start 명령으로 컨테이너 실행                        │
└──────────────────────────────────────────────────────────────┘
enroot start \
--mount /fsx:/fsx \                    ← FSx 마운트
--env CUDA_VISIBLE_DEVICES=0 \         ← GPU 할당
--env NCCL_SOCKET_IFNAME=eth0 \
my-container \
python /fsx/scripts/train.py
↓
┌──────────────────────────────────────────────────────────────┐
│ 4. 컨테이너 내부에서 프로세스 실행                            │
└──────────────────────────────────────────────────────────────┘
Namespace 격리:
├─ PID namespace     (프로세스 격리)
├─ Mount namespace   (파일 시스템 격리)
├─ Network namespace (네트워크 격리)
└─ User namespace    (사용자 격리)

GPU 접근:
└─ /dev/nvidia0 → GPU 0 직접 접근
