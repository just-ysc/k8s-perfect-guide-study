# 리소스 관리와 오토 스케일링

## 리소스 제한

- 컨테이너 단위로 리소스 제한 설정 가능
  - CPU
  - 메모리
  - Ephemeral 스토리지
- Device Plugins를 사용하면 GPU 등 다른 리소스에 대해서도 제한 설정 가능

### CPU/메모리 리소스 제한

- CPU: 1vCPU를 1000m(millicores) 단위로 지정
- `spec.containers[].resources.requests`, `spec.containers[].resources.limits`
  - [예시](./sample-resource.yaml)
- Requests: 사용할 리소스 최솟값
  - 설정되지 않을 경우 Limits와 같은 값이 설정
- Limits: 사용할 리소스 최댓값
  - 노드에 Limits 만큼의 리소스가 남아 있지 않아도 스케줄링됨
  - 설정되지 않을 경우 호스트 측의 부하가 최대로 상승할 떄까지 리소스를 계속 소비
    - CPU의 경우 thrashing 발생 가능
    - 메모리의 경우 OOM 발생 가능

### Ephemeral 스토리지 리소스 제어

- 쿠버네티스 노드에서 기동 중인 kubelet이 디스크 사용 현황을 정기적으로 모니터링하고 초과한 경우 pod를 evict
  - 컨테이너 로그
  - emptyDir에 기록된 데이터
  - 쓰기 가능한 레이어에 기록된 데이터
    - PV/hostPath/nfs 마운트 영역을 제외한 모든 영역에 대한 쓰기
- [예시](./sample-ephemeral-storage.yaml)
- 동일한 emptyDir 볼륨을 여러 파드나 컨테이너 사용할 경우 각각의 limits는 해당 볼륨을 사용하는 모든 컨테이너의 limits을 합산하여 적용
  - [예시](./sapmle-ephemeral-storage-multi.yaml)
  - 위 예시에서 하나의 컨테이너에서 emptyDir 볼륨에 3GB 이상을 쓰더라도 두 컨테이너 모두 evict되지 않음

### 시스템에 할당된 리소스와 Eviction 매니저

- 리소스의 완전 고갈로 인한 쿠버네티스 장애나 노드 장애를 막기 위해 kube-reserved, system-reserved라는 두 가지 리소스가 확보되어 있음
  - kube-reserved: 쿠버네티스 시스템 구성 요소나 컨테이너 런타임에 확보된 리소스
  - system-reserved: OS에 깊이 관련된 데몬 등에 확보된 리소스
- 쿠버네티스 내부에서는 Eviction 매니저라는 구성 요소가 동작
  - Allocatable, system-reserved, kube-reserved 영역에서 실제로 사용되는 리소스 합계가 Eviction Threshold를 넘지 않는지 확인하고, 넘을 경우 파드를 Evict
  - soft 제한과 hard 제한이 존재
    - soft 제한: SIGTERM 시그널로 파드 정지 시도
      - 파드의 `terminationGracePeriodSeconds`와 kubelet의 `--eviction-soft-grace-period` 중 짧은 쪽을 선택
    - hard 제한: SIGKILL 시그널로 파드 즉시 정지
  - Evict 우선순위
    1. Requests에 할당된 양보다 초과하여 리소스를 소비하고 있는 것
    2. PodPriority가 더 낮은 것
    3. Requests에 할당된 양보다 초과하여 소비하고 있는 리소스 양이 더 많은 것

### GPU 등의 리소스 제한

#### 엔비디아 GPU

```yaml
resources:
  requests:
    nvidia.com/gpu: 2
  limits:
    nvidia.com/gpu: 2
```

### 오버커밋과 리소스 부족

- 오버커밋: 클러스터에 배포하려는 쿠버네티스 리소스가 전체 클러스터 리소스보다 큰 상황

### 여러 컨테이너 사용 시 리소스 할당

- `max(sum(containers[*]), max(initContainers[*]))`

---

## Cluster Autoscaler와 리소스 부족

- Cluster Autoscaler: 수요에 따라 쿠버네티스 노드를 자동으로 추가하는 기능
  - Pending 상태의 파드가 생기는 타이밍에 처음으로 동작
    - 따라서 Requests, Limits를 명확하게 설정해서 배포해야 함
- 명심해야 할 2가지
  - Request와 Limits에 너무 큰 차이를 주지 않을 것
  - Requests를 너무 크게 설정하지 않을 것

---

## LimitRange를 사용한 리소스 제한

- 파드/컨테이너/영구 볼륨 클레임에 대해 CPU나 메모리 리소스의 최솟값과 최댓값, 기본값 등을 설정
  - default: 기본 Limits
  - defaultRequest: 기본 Requests
  - max: 최대 리소스
  - min: 최소 리소스
  - maxLimitRequestRatio: Limits/Requests의 비율
- 신규 파드를 생성할 때 사용되므로 기존 파드에는 영향을 주지 않음

### 기본으로 생성되는 LimitRange

- 일부 환경에서는 사용자가 Request 설정 없이 컨테이너를 지속적으로 배포해 부하가 높아져 장애가 발생하는 것을 막기 위해 LimitRange를 통해 기본 CPU Requests를 설정하기도 함

### 컨테이너에 대한 LimitRange

- `type: Container`
- [예시](./sample-limitrange-container.yaml)

### 파드에 대한 LimitRange

- `type: Pod`
- 컨테이너에서 사용하는 리소스 합계로 최대/최소 리소스 제한
- [예시](./sample-limitrange-pod.yaml)

### 영구 볼륨 클레임에 대한 LimitRange

- `type: PersistentVolumeClaim`
- [예시](./sample-limitrange-pvc.yaml)

> <strong>LimitRange는 namespace별로 하나만 생성할 수 있는가?</strong>  
> <strong>여러 개 생성이 가능하다면 충돌 시에는 어떻게 되는가?</strong>

---