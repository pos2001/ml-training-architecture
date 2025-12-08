
### 귀하께서 직접 컴파일하지 않았는데도 NCCL 컴파일 오류가 발생한 것은 사전 설치된 라이브러리와 CUDA 버전 간의 불일치 때문입니다.
```
클러스터 환경
├── 설치된 CUDA 버전: 예) CUDA 12.x
├── 사전 컴파일된 NCCL: 예) CUDA 11.x용으로 빌드됨
└── 런타임 검증 시도 → 버전 불일치 감지 → 오류 발생

```




### ParallelCluster 특화 팁
```
1. AMI 선택이 중요
# Deep Learning AMI 사용 (사전 구성된 환경)
aws ec2 describe-images \
--owners amazon \
--filters "Name=name,Values=Deep Learning AMI GPU PyTorch*"

2. 커스텀 부트스트랩 스크립트
#!/bin/bash
# 특정 버전 조합 강제 설치
pip uninstall torch torchvision torchaudio -y
pip install torch==2.0.1+cu118 torchvision==0.15.2+cu118 \
--extra-index-url https://download.pytorch.org/whl/cu118

3. 컨테이너 사용 권장
# 검증된 NVIDIA NGC 컨테이너 사용
FROM nvcr.io/nvidia/pytorch:23.08-py3
# 이미 호환되는 CUDA, NCCL, PyTorch 포함

```





```

```
