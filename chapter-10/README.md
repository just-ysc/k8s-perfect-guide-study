# 헬스 체크와 컨테이너 라이프사이클

## 헬스 체크

### Liveness, Readiness, Startup Probe

- Liveness Probe
  - 파드 내부의 컨테이너가 정상 동작 중인지 확인
  - 실패 시 컨테이너 재기동
  - ex) 메모리 누수로 인한 OOM
- Readiness Probe
  - 파드가 요청을 받아들일 수 있는지 확인
  - 실패 시 트래픽 차단
  - ex) 데이터베이스 접속 장애, 캐시 장애, 초기화 프로세스 기동 완료 여부 확인
- Startup Probe
  - 파드의 첫 번째 기동이 완료되었는지 확인
  - 실패 시 다른 Probe 실행을 시작하지 않음

### 세 가지 헬스 체크 방식

- exec
  - 명령어를 실행하고 종료 코드가 0이 아니면 실패
  - 명령어는 컨테이너별로 실행
```yaml
livenessProbe:
  exec:
    command: ["test", "-e", "/ok.txt"]
```
- httpGet
  - HTTP GET 요청을 실행하고 Status Code가 200~399가 아니면 실패
  - kubelet에서 요청을 수행
    - 애플리케이션에서 IP 기반 접근제어를 할 경우 kubelet의 네트워크 인터페이스 IP 주소를 허용해야 함
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80
    scheme: HTTP
    host: web.example.com
    httpHeaders:
      - name: Authorization
        value: Bearer TOKEN
```
- tcpSocket
  - TCP 세션이 연결되지 않으면 실패
```yaml
livenessProbe:
  tcpSocket:
    port: 80
```

### 헬스 체크 간격

- `initialDelaySeconds`: 첫 번째 헬스 체크 시작까지의 지연
- `periodSeconds`: 헬스 체크 간격 시간(초)
- `timeoutSeconds`: 타임아웃까지의 시간(초)
- `successThreshold`: 성공이라고 판단하기까지의 체크 횟수
- `failureThreshold`: 실패라고 판단하기까지의 체크 횟수

### 헬스 체크 생성

- [예시](./sample-healthcheck.yaml)

### Liveness Probe 실패

- [예시](./sample-liveness.yaml)
  - 체크가 실패하도록 `index.html` 삭제
```shell
$ kubectl exec -it sample-liveness -- rm -f /usr/share/nginx/html/index.html
```
- `restartPolicy`에 따라 컨테이너를 재시작

### Readiness Probe 실패

- [예시](./sample-readiness.yaml)
  - 체크가 실패하도록 파일 삭제
```shell
$ kubectl exec -it sample-readiness -- rm -f /usr/share/nginx/html/50x.html
```

#### 파드 ready++(ReadinessGate)

- 파드가 정말 Ready 상태인지를 추가로 체크
  - 클라우드 외부 로드 밸런서와의 연계에 시간이 걸리는 경우
  - 일반적인 파드의 Ready 판단만으로는 롤링 업데이트 시 기존 파드가 한 번에 사라져 안전하게 업데이트할 수 없는 경우
- `spec.readinessGates` 설정
  - 통과할 때까지 서비스 전송 대상에 추가되지 않고, 롤링 업데이트 시 다음 파드 기동으로 이동하지 않음
- [예시](./sample-readinessgate.yaml)
- 여러 조건을 설정할 경우 모든 조건이 Ready가 되지 않으면 파드는 Ready가 되지 않음
  - 이 조건들은 커스텀 컨트롤러 등을 사용하여 구현하는 것이 일반적

#### Readiness Probe를 무시한 서비스 생성

- 스테이트풀셋에서 Headless 서비스를 사용할 때는 파드가 Ready 상태가 되지 않아도 클러스터를 구성하기 위해 각 파드의 이름 해석이 필요한 경우 존재
- Readiness Probe가 실패한 경우에도 서비스에 연결되게 하려면 `spec.publishNotReadyAddresses` 설정
- [예시](./sample-publish-notready.yaml)

### Startup Probe를 사용한 지연 체크와 실패

- [예시](./sample-startup.yaml)
- `failureThreshold` 횟수만큼 실패한 경우에는 Liveness Probe와 마찬가지로 파드가 정지되고 `restartPolicy`에 따라 재기동

---

## 컨테이너 라이프사이클과 재기동

- 컨테이너 재기동 정책은 `spec.restartPolicy`로 지정
  - Always: 항상 파드를 재기동
  - OnFailure: 0이 아닌 종료 코드로 파드가 정지한 경우 재기동
  - Never: 재기동하지 않음

### `Always`

- [예시](./sample-restart-always.yaml)

### `OnFailure`

- [예시](./sample-restart-onfailure.yaml)

### `Never`

- [예시](./sample-restart-never.yaml)

---

## 초기화 컨테이너

- 파드 내부에서 메인 컨테이너를 기동하기 전에 별도로 기동하는 컨테이너
  - 저장소에서 파일 로드
  - 컨테이너 기동 지연
  - 설정 파일 동적 생성
- `spec.initContainers[]`
  - 위에서부터 순차적으로 기동
- [예시](./sample-initcontainer.yaml)

---

## `postStart` / `preStop`

- `postStart`: 컨테이너 기동 후 임의의 명령어 실행
  - `spec.containers[].command`와 거의 같은 타이밍에 비동기로 실행
- `preStop`: 컨테이너 정지 전 임의의 명령어 실행
- [예시](./sample-lifecycle-exec.yaml)
- `exec`외에 `httpGet`도 사용 가능
  - [예시](./sample-lifecycle-httpget.yaml)
- `postStart`, `preStop`은 <b>최소 한 번</b> 실행
  - 두 번 이상 실행될 수 있음
- `postStart`는 타임아웃 설정이 불가
  - 실행 중에는 Probe도 실행되지 않기 때문에 오래 걸리는 명령을 실행할 경우 주의

---

