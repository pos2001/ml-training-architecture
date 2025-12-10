
### ParallelCluster UI에서 maxcount 증가(2 ->4)는 바로 가능
<img width="572" height="108" alt="image" src="https://github.com/user-attachments/assets/0e7998c5-0e4b-4940-8cf3-914cb97f29d1" />




### ParallelCluster UI에서 maxcount 증가는 바로 가능하지만, 
### 감소는 아래와 같은 에러 메시지 발생

<img width="960" height="775" alt="image" src="https://github.com/user-attachments/assets/04c29922-9df1-466c-89bd-bf7ecd126026" />


### 그래서 아래와 같이 compute fleet stop 시킴
<img width="1014" height="277" alt="image" src="https://github.com/user-attachments/assets/d0a84b56-1b1a-4549-a8ca-b2ef92a979b1" />

### compute fleet stop 상태에서, 아래와 같이 edit cluster 클릭
<img width="1004" height="237" alt="image" src="https://github.com/user-attachments/assets/7a60d489-10f8-4485-bbb0-84fe9b128105" />

### 다시 4개에서 2개로 감소된 것을 확인할 수 있다
<img width="585" height="197" alt="image" src="https://github.com/user-attachments/assets/f80cb7ae-15d1-4b2f-864e-722c4d359990" />


## 클러스터 크기를 줄일 때
### 방법 A: Compute Fleet 중지


```
# 1. 컴퓨팅 노드 중지
pcluster update-compute-fleet --cluster-name my-cluster --status STOP_REQUESTED

# 2. 설정 변경 (크기 축소)
pcluster update-cluster --cluster-name my-cluster --config-file new-config.yaml
```

### QueueUpdateStrategy를 TERMINATE로 설정
```
Scheduling:
QueueUpdateStrategy: TERMINATE  # 설정 파일에 추가

# 바로 업데이트 가능
pcluster update-cluster --cluster-name my-cluster --config-file new-config.yaml

```


### TERMINATE 동작 방식
### 노드 목록 뒤에서부터 종료 (앞쪽 노드는 유지)
```
초기 상태:
- MinCount = 5, MaxCount = 10
- 노드: st-[1-5], dy-[1-5]

축소 후:
- MinCount = 3, MaxCount = 5
- 유지되는 노드: st-[1-3], dy-[1-2]  ✅ 그대로 유지
- 종료되는 노드: st-[4-5], dy-[3-5]  ❌ 삭제됨

```

###  Compute Fleet 중지 없이 가능한 변경
```
이런 변경은 즉시 적용 가능 (노드 중지 불필요):

✅ 새 큐(SlurmQueue) 추가 ✅ 새 컴퓨팅 리소스(ComputeResource) 추가 ✅ MaxCount 증가 ✅ MinCount와 MaxCount 동시에 같은 양 이상 증가

# 이런 변경은 바로 가능
Scheduling:
SlurmQueues:
- Name: queue1
ComputeResources:
- MinCount: 5 → 8     # +3
MaxCount: 10 → 15   # +5 (MinCount 증가분 이상)

```




