# 메인터넌스와 노드 정지

## 노드 정지와 파드 정지

- 쿠버네티스 노드를 정지하기 위해서는 노드에 스케줄링된 파드를 정지해야 함
  - SIGTERM, SIGKILL 시그널에 안정적으로 반응하도록 애플리케이션을 구성해야 함
  - `terminationGracePeriodSeconds`를 적절하게 설정해야 함

---