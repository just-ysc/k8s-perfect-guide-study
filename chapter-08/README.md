# 클러스터 API 카테고리와 메타데이터 API 카테고리

- 클러스터 API 카테고리: 보안 관련 설정이나 쿼터 설정 등 클러스터 동작 제어
  - 노드
  - 네임스페이스
  - 영구 볼륨
  - 리소스 쿼터
  - 서비스 어카운트
  - 롤
  - 클러스터롤
  - 롤바인딩
  - 클러스터롤바인딩
  - 네트워크 정책
- 메타데이터 API 카테고리: 클러스터에 컨테이너를 기동하는 데 사용하는 리소스
  - LimitRange
  - HorizontalPodAutoscaler
  - PodDisruptionBudget
  - CustomResourceDefinition

---

## 노드

- 사용자가 생성하거나 삭제하는 리소스는 아니지만 쿠버네티스에 리소스로 등록
- `status.capacity`: 노드가 소유하고 있는 CPU, 메모리의 실제 용량
- `status.allocatable`: capacity에서 쿠버네티스가 시스템 리소스로 할당하고 있는 만큼을 제외한 실제 파드에 할당 가능한 리소스 용량
- `kubectl describe node` 또는 `kubectl get nodes -o yaml(or json)`으로 노드에 관한 대부분의 정보를 확인할 수 있음
