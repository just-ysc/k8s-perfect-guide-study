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

## QoS Class

- 사용자가 직접 설정하지 않고, 파드의 Requests/Limits 설정에 따라 자동으로 설정되는 값
  - Guaranteed: Requests/Limits가 같고 CPU와 메모리 모두 지정되어 있음
  - Burstable: Guaranteed를 충족하지 못하고 한 개 이상의 Requests/Limits가 설정되어 있음
  - BestEffort: Requests/Limits 모두 미지정
- QoS Class는 OOM Killer의 우선순위인 oom score(-1000 ~ 1000)를 설정할 때 사용
  - Guaranteed: -998
  - Burstable: `min(max(2, 1000 - (1000 * 메모리 Request) / 머신 메모리 용량), 999)`
  - BestEffort: 1000
- 파드의 `status.qosClass`에서 확인 가능

### BestEffort

- [예시](./sample-qos-besteffort.yaml)

### Guaranteed

- 모든 파드의 QoS Class를 Guaranteed로 한다면 부하 증가에 따른 다른 파드로의 영향(noisy neighbor)을 피할 수 있는 장점
- 집약률이 낮아지고 리소스 비효율이 발생할 수 있는 단점
- [예시](./sample-qos-guaranteed.yaml)

### Burstable

- 변동 요소가 크기 때문에 최악의 경우 노드가 과부하를 받을 가능성
- [예시](./sample-qos-burstable.yaml)

---

## 리소스 쿼터를 사용한 네임스페이스 리소스 쿼터 제한

- 각 네임스페이스마다 사용 가능한 리소스 제한 가능
- 이미 생성된 리소스에는 영향을 주지 않음
- [예시](./sample-resourcequota.yaml)

### 생성 가능한 리소스 수 제한

- `count/RESROUCE.GROUP`
- [예시](./sample-resourcequota-count-new.yaml)
- 쿠버네티스 v1.9 이전의 방식으로는 서비스 type별/PVC storageClass별로 리소스 수를 제한하는 것도 가능
  - [예시](./sample-resourcequota-count-old.yaml)

### 리소스 사용량 제한

- 컨테이너에 할당 가능한 리소스 양을 제한
- 스토리지는 Requests만 지정 가능
- GPU 등의 확장 리소스에 대한 제한도 설정
- [예시](./sample-resourcequota-usable.yaml)

---

## HorizontalPodAutoscaler

- 디플로이먼트/레플리카셋/레플리케이션 컨트롤러의 레플리카 수를 CPU 부하 등에 따라 자동으로 스케일하는 리소스
- <b>파드에 Resource Requests가 설정되어 있지 않은 경우에는 동작하지 않음</b>
- 필요한 레플리카 수 = `ceil(sum(파드의 현재 CPU 사용률) / targetAverageUtilization)`
- 최대 3분에 1회 스케일 아웃 실행 / 최대 5분에 1회 스케일 인 실행
- 스케일 아웃 조건식
  - `avg(파드의 현재 CPU 사용률) / targetAverageUtilization > 1.1`
- 스케일 인 조건식
  - `avg(파드의 현재 CPU 사용률) / targetAverageUtilization < 0.9`
- [예시](./sample-hpa.yaml)
- CPU 이외의 리소스를 사용하여 오토 스케일링을 하는 경우에는 프로메테우스나 그 외의 메트릭 서버와 연계하기 위한 별도 설정 필요
  - Resource: CPU/메모리
  - Object: 쿠버네티스 Object 메트릭(Ingress hit 등)
  - Pods: 파드 메트릭
  - External: 외부 메트릭

### HorizontalPodAutoscaler 스케일링 동작 설정

- autoscaling/v2beta2부터 `spec.behavior`로 오토 스케일링 빈도나 증감 가능한 레플리카 수 등을 리소스 단위로 설정 가능
- [예시](./sample-hpa-behavior.yaml)

---
