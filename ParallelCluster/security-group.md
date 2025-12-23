
### ParallelCluster가 자동으로 설정하는 3가지 주요 통신 경로:
```



1. 사용자 → HeadNode 접근 🔓
외부 사용자
    ↓ SSH (포트 22)
HeadNode Security Group

    자동 설정: SSH 포트 22 오픈
    기본값: 0.0.0.0/0 (모든 IP 허용)
    커스터마이징 가능: Ssh.AllowedIps로 특정 IP만 허용

2. HeadNode ↔ ComputeNode 양방향 통신 🔄
HeadNode
    ↕ (모든 포트)
ComputeNode

    자동 설정: 양방향 모든 트래픽 허용
    포함 내용:
        Slurm 스케줄러 통신
        NFS 서버/클라이언트 (포트 2049, 111, 20048 등)
        SSH (HeadNode에서 ComputeNode로 작업 제출)
        모니터링 및 관리 트래픽

3. ComputeNode ↔ ComputeNode 통신 🔄
ComputeNode 1
    ↕ (모든 포트)
ComputeNode 2, 3, 4...

    자동 설정: ComputeNode끼리 모든 트래픽 허용
    필요한 이유:
        MPI 병렬 처리
        분산 컴퓨팅 작업
        노드 간 데이터 교환
```
