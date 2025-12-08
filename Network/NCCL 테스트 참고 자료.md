https://swsmith.cc/posts/efa-best-practices.html
https://github.com/aws-samples/awsome-distributed-training/blob/main/1.architectures/efa-cheatsheet.md
https://aws.amazon.com/blogs/hpc/large-scale-training-with-nemo-megatron-on-aws-parallelcluster-using-p5-instances/



NCCL+AWS OFI NCCL+EFA installer만 잘 맞는 버전으로 설치하시면 자동으로 EFA를 탑니다.

export NCCL_PROTO=simple
export NCCL_SOCKET_IFNAME=ens5
export GLOO_SOCKET_IFNAME=ens5 이런건 다 빼도 되고 최신 버전에선 제가 테스트해봤을 땐 export FI_PROVIDER=efa (libfabric/EFA 선택) 이것도 필요 없었습니다.
디버그 용도로 export NCCL_DEBUG=INFO는 테스트용으로 하나 넣어두고 EFA cheatsheet 정도 내용만 보셔도 될 거에요
위에 제가 언급한 세개는 Dockerfile에 추가해서 테스트해보셔야 해요
NCCL, AWS-OFI-NCCL, EFA INSTALLER
9:43
참고로 최근까지 NCCL v2.28.3-1, AWS-OFI-NCCL v1.17.1, EFA-INSTALLER v1.44.0 이 세가지 조합으로 테스트했을 때 EFA 잘 타는 거 확인했습니다
