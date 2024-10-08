# 컨피그 & 스토리지 API 카테고리

- 컨테이너에 대해 설정 파일이나 기밀 정보 등을 추가하거나 영고 볼륨을 제공하기 위한 리소스
  - Secret
  - ConfigMap
  - PersistentVolumeClaim(PVC)
---

## 쿠버네티스의 환경 변수

- 파드 템플릿에 `env`, `envForm`을 지정

### 정적 설정

- 정적인 key-value를 설정
  - [예시](./sample-env.yaml)

### 파드 정보

- `fieldRef`
  - 파드가 기동하고 있는 노드 정보
  - 파드 자신의 IP 주소
  - 파드의 기동 시간 등
- `kubectl get po sample-env -o yaml` 명령어로 확인 가능
- `fieldRef`로 관련 정보를 컨테이너 환경 변수로 주입 가능
    - [예시](./sample-env-pod.yaml)

### 컨테이너 정보

- `resourceFieldRef`
  - 사용 방식은 `fieldRef`와 동일
  - [예시](./sample-env-container.yaml)

### 주의사항

- 쿠버네티스에서 `command`나 `args`로 명령어를 지정할 때는 환경 변수 사용 방법이 다르다.
  - [실패 예시](./sample-env-fail.yaml)
- 파드 매니페스트 내부에 정의된 환경 변수는 `${}`가 아닌 `$()`를 사용해야 한다.
  - [성공 예시](./sample-env-fail2.yaml)
- `fieldRef`, `resourceFieldRef`로 참조한 환경 변수도 마찬가지이다.
  - [예시](./sample-env-fail3.yaml)
---

## Secret

- 사용자명, 패스워드 등의 기밀 정보를 별도 리소스에 정의하고 파드에서 읽어들일 수 있게 하는 리소스
- <strong>시크릿 매니페스트 자체는 base64 인코딩되지만 암호화되지는 않기 때문에 매니페스트를 그대로 외부 저장소에 업로드하는 것은 불가능</strong>
  - `kubesec`, `SealedSecret`, `ExternalSecret` 등의 오픈 소스 소프트웨어에서 암호화 지원

### 시크릿 분류

| 종류                                  | 개요              |
|-------------------------------------|-----------------|
| Opaque                              | 범용              |
| kubernetes.io/tls                   | TLS 인증서용        |
| kubernetes.io/basic-auth            | 기본 인증용          |
| kubernetes.io/dockerconfigjson      | 도커 레지스트리 인증 정보용 |
| kubernetes.io/ssh-auth              | SSH 인증 정보용      |
| kubernetes.io/service-account-token | 서비스 어카운트 토큰용    |
| bootstrap.kubernetes.io/token       | Bootstrap 토큰용   |
- Opaque 타입 이외의 리소스는 고정된 스키마가 있음

### Opaque

- 시크릿 생성 패턴
  - `--from-file`
  - `--from-env-file`
  - `--from-literal`
  - `-f`

#### `--from-file`

- 파일명이 그대로 키가 되기 때문에 확장자는 붙이지 않는 것을 권장
- 개행 코드가 들어가지 않도록 주의
```shell
$ echo -n "root" > ./username
$ echo -n "rootpassword" > ./password
$ kubectl create secret generic --save-config sample-db-auth \
--from-file=./username --from-file=./password
```

#### `--from-env-file`

- [envfile 예시](./env-secret.txt)
```shell
$ kubectl create secret generic --save-config sample-db-auth \
--from-env-file ./env-secret.txt
```

#### `--from-literal`

```shell
$ kubectl create secret generic --save-config sample-db-auth \
--from-literal=username=root --from-literal=password=rootpassword
```

#### `-f`

- base64로 인코딩한 값을 매니페스트에 추가
  - [예시](./sample-db-auth.yaml)
- base64로 인코딩하지 않은 값을 사용하려면 `data` 대신 `stringData` 사용
  - [예시](./sample-db-auth-nobase64.yaml)

### TLS 시크릿

- 미리 생성해 둔 인증서와 비밀키 파일로 생성 가능
```shell
$ kubectl create secret tls --save-config tls-sample --key ./tls.key --cert ./tls.crt
```

### 도커 레지스트리 시크릿

- 프라이빗 도커 레지스트리에서 이미지를 가져오기 위한 인증을 수행할 수 있는 시크릿
- 보통 kubectl로 직접 생성
- `~/.docker/config.json` 파일을 대체
```shell
$ kubectl create secret docker-registry --save-config sample-registry-auth \
--docker-server=REGISTRY_SERVER \
--docker-username=REGISTRY_USER \
--docker-password=REGISTRY_USER_PASSWORD \
--docker-email=REGISTRY_USER_EMAIL
```

- 시크릿 생성 후, 해당 시크릿을 사용해 프라이빗 도커 레지스트리에서 이미지를 가져오려면 `spec.imagePullSecrets`에 생성한 시크릿 정보를 지정
  - [예시](./sample-pull-secret.yaml)

### 기본 인증 시크릿

- 사용자명, 패스워드로 인증

#### kubectl로 생성(`--from-literal`)

```shell
$ kubectl create secret generic --save-config sample-basic-auth \
--type kubernetes.io/basic-auth \
--from-literal=username=root --from-literal=password=rootpassword
```

#### 매니페스트에서 생성(`-f`)

- [예시](./sample-basic-auth.yaml)

### SSH 인증 시크릿

#### kubectl로 생성(`--from-file)
```shell
$ kubectl create secret generic --save-config sample-ssh-auth \
--type kubernetes.io/ssh-auth \
--from-file=ssh-privatekey=./sample-key
```

#### 매니페스트에서 생성(`-f`)

- [예시](./sample-ssh-auth.yaml)

### 시크릿 사용

- 환경 변수로 전달 or 볼륨으로 마운트
- 특정 키만 or 전체 키 모두

#### 환경 변수로 전달

- 특정 키만 전달
  - `spec.containers[].env[].valueFrom.secretKeyRef`
    - [예시](./sample-secret-single-env.yaml)
    - 하나씩 지정해서 넘기기 때문에 별도의 환경 변수명을 지정 가능
- 모든 키 전달
  - `spec.containers[].envFrom[].secretRef`
    - [예시](./sample-secret-multi-env.yaml)
  - prefix를 붙여 키 충돌 방지
    - [예시](./sample-secret-prefix-env.yaml)

#### 볼륨으로 마운트

- 특정 키만 마운트
  - `spec.volumes[].secret.items[]`
    - [예시](./sample-secret-single-volume.yaml)
    - 하나씩 지정해서 마운트하기 때문에 별도의 파일명을 지정 가능
- 모든 키 마운트
  - `spec.volumes[].secret`
    - [예시](./sample-secret-multi-volume.yaml)

#### 동적 시크릿 업데이트

- 볼륨 마운트한 시크릿은 kubelet의 sync loop 주기마다 kube-apiserver로 변경을 확인하고 변경이 있을 경우 파일을 교체
  - 기본 sync loop 주기: 60초
  - `--sync-frequency`로 조정 가능

---

## ConfigMap

- 설정 정보 등을 키-밸류 값으로 저장할 수 있는 데이터 저장 리소스
  - 키-밸류 이외에 설정 파일 자체도 저장 가능

### 컨피그맵 생성

- 생성 방법
  - kubectl로 파일에서 값을 참조하여 생성(`--from-file`) 
  - kubectl로 직접 값을 전달하여 생성(`--from-literal`)
  - 매니페스트로 생성(`-f`)
- 하나의 컨피그맵에 저장 가능한 사이즈는 총 1MB

#### `--from-file`

```shell
$ kubectl create configmap --save-config sample-configmap --from-file=./nginx.conf
```
- `data` 대신 `binaryData`를 사용하면 UTF-8 이외의 데이터를 저장 가능
  -  base64로 인코드한 값이 등록

#### `--from-literal`

```shell
$ kubectl create configmap --save-config web-config \
--from-literal=connection.max=100 --from-literal=connection.min=10
```

#### `-f`

- 밸류를 여러 행으로 전달할 경우 | 사용
- 숫자는 큰따옴표
- [예시](./sample-configmap.yaml)

### 컨피그맵 사용

- 환경 변수로 전달 or 볼륨으로 마운트
- 특정 키만 or 전체 키 모두

#### 환경 변수로 전달

- 특정 키만 전달
  - `spec.containers[].env[].valueFrom.configMapKeyRef`
  - [예시](./sample-configmap-single-env.yaml)
- 모든 키 전달
  - `spec.containers[].envFrom[].configMapRef`
  - [예시](./sample-configmap-multi-env.yaml) 

#### 볼륨으로 마운트

- 특정 키만 마운트
  - `spec.volumes[].configMap.items[]`
  - [예시](./sample-configmap-single-volume.yaml)
- 모든 키 마운트
  - `spec.volumes[].configMap`
  - [예시](./sample-configmap-multi-volume.yaml)

### 시크릿, 컨피그맵의 공통 주제

#### 시크릿에서 기밀 정보를 취급하기 위한 구조

- etcd에 저장된 시크릿 데이터는 시크릿을 사용하는 파드가 있을 경우에만 노드에 데이터를 전송
- 또한, 데이터가 영구적으로 남지 않도록 노드의 tmpfs 영역에 저장

#### 시크릿과 컨피그맵으로 볼륨을 생성하고 마운트할 경우 데이터에 대한 퍼미션 설정 가능
- 기본값은 `0644`
- 8진수 표기를 10진수 표기로 변환한 형태를 사용해야 함
- [예시 1: ConfigMap의 shell script를 실행할 수 있도록 변경](./sample-configmap-scripts.yaml)
- [예시 2: Secret에 owner readonly로 접근 권한을 변경](./sample-secret-secure.yaml)

#### 동적 컨피그맵 업데이트

- 볼륨 마운트한 시크릿은 kubelet의 sync loop 주기마다 kube-apiserver로 변경을 확인하고 변경이 있을 경우 파일을 교체
  - 기본 sync loop 주기: 60초
  - `--sync-frequency`로 조정 가능
- 환경 변수를 사용한 컨피그맵은 기동할 때 환경 변수가 정해지기 때문에 동적으로 업데이트를 할 수 없음
  - 컨피그맵을 사용중인 파드 재기동 필요

#### 시크릿, 컨피그맵의 데이터 변경 거부(immutable)

- 데이티 변경을 방지함으로써 예상치 못한 시스템 변경도 방지
- 데이터를 변경하려면 리소스를 삭제하고 나서 다시 생성해야 함
  - 볼륨 마운트 중인 경우에는 파드 재생성도 필요
- [예시](./sample-secret-immutable.yaml)

---

## 영구 볼륨 클레임(PersistentVolumeClaim)

### 볼륨, 영구 볼륨, 영구 볼륨 클레임의 차이

- 볼륨
  - 미리 준비된 사용 가능한 볼륨을 매니페스트에서 지정하여 사용
  - 이미 준비된 볼륨은 사용 가능하지만, 새로 만들거나 삭제하는 작업은 불가능
- 영구 볼륨
  - 외부 영구 볼륨을 제공하는 시스템과 연계하여 볼륨 생성과 삭제가 가능
  - 매니페스트에서 영구 볼륨 리소스를 별도로 생성하는 형태
- 영구 볼륨 클레임
  - 생성된 영구 볼륨 리소스를 할당
  - 영구 볼륨 클레임을 사용해야 클러스텅 등록된 볼륨을 파드에서 사용할 수 있음

---

## 볼륨

- 쿠버네티스에서 제공되는 볼륨 플러그인
  - emptyDir
  - hostPath
  - downwardAPI
  - projected
  - nfs
  - iscsi
  - cephfs

### emptyDir

- 파드용 임시 디스크 영역
  - 파드가 종료되면 삭제
- 호스트 임의 영역을 마운트할 수 없음
  - 호스트에 있는 파일 참조 불가
- [예시](./sample-emptydir.yaml)
- `emptyDir.sizeLimit`으로 리소스 제한도 가능
  - [예시](./sample-emptydir-limit.yaml)
- 디스크 영역 이외에 tmpfs 메모리 영역을 사용할 수도 있음
  - [예시](./sample-emptydir-memory.yaml)
  - 메모리 영역을 사용할 경우 `resources.limits.memory` 제한에도 영향을 받기 때문에 주의가 필요함
    - [예시](./sample-emptydir-memory-with-memory-limits.yaml)

### hostPath

- 쿠버네티스 노드상의 영역을 컨테이너상에 매핑
- [예시](./sample-hostpath.yaml)

### downwardAPI

- 파드 정보 등을 파일로 배치하기 위한 플러그인
- [예시](./sample-downward-api.yaml)

### projected

- 시크릿, 컨피그맵, downwardAPI, serviceAccountToken의 볼륨 마운트를 하나의 디렉터리에 통합하는 플러그인
- [예시](./sample-projected.yaml)

---

## 영구 볼륨(PV)

- 영속성 영역으로 확보된 볼륨
- 개별 리소스로 생성한 후 사용

### 영구 볼륨 종류

- 네트워크를 통해 디스크를 attach하는 디스크 타입
- 플러그인 종류
  - GCE Persistent Disk
  - AWS Elastic Block Store
  - Azure File
  - nfs
  - iSCSI
  - Ceph(RBD, CephFS)
  - OpenStack Cinder
  - GlusterFS
  - Container Storage Interface(CSI)

#### Container Storage Interface

- 컨테이너 오케스트레이션 엔진과 스토리지 시스템을 연결하는 인터페이스
- CSI를 사용하면 쿠버네티스부터 공통화된 CSI 인터페이스를 통해 다양한 프로바이더를 사용할 수 있음

### 영구 볼륨 생성

- 설정 가능한 항목
  - 레이블
    - type, environment, speed 등
    - 영구 볼륨 클레임에서 레이블로 볼륨을 지정할 수 있음
    - 쉽게 원하는 볼륨을 선택할 수 있도록 설정해 두는 것이 좋음
  - 용량
  - 접근 모드
    - ReadWriteOnce(RWO): 단일 노드에서 Read/Write 가능
    - ReadOnlyMany(ROX): 여러 노드에서 Read 가능
      - 파드에서 영구 볼륨 클레임을 지정할 때 `spec.volumes[].persistentVolumeClaim.readOnly`로 지정
      - [예시](./sample-pv-readonlymany.yaml)
    - ReadWriteMany(RWX): 여러 노드에서 Read/Write 가능
      - nfs 등에서 사용 가능
    - 영구 볼륨에 따라 지원하는 접근 모드가 다름
  - Reclaim Policy: 영구 볼륨을 사용한 후 처리 방법을 제어하는 정책
    - `spec.persistentVolumeReclaimPolicy`
      - Delete: 영구 볼륨 자체를 삭제
      - Retain: 영구 볼륨 자체를 삭제하지 않고 유지
        - 또 다른 영구 볼륨 클레임에 의해 이 영구 볼륨이 다시 마운트되지 않음
      - ~~Recycle: 영구 볼륨의 데이터를 삭제하고 재사용 가능한 상태로 전환~~
        - ~~다른 영구 볼륨 클레임에 의해 마운트 가능~~
        - <b>더 이상 사용하지 않기 때문에, 대신 동적 프로비저닝 사용 필요</b>
  - 마운트 옵션
    - `spec.mountOptions`
  - 스토리지클래스
    - [GKE에서 동적 프로비저닝을 막는 스토리지클래스 예시](./sample-storageclass-manual.yaml)
  - 각 플러그인 특유의 설정
- [GCE 영구 디스크용 플러그인을 사용한 예시](./sample-pv.yaml)

---

## 영구 볼륨 클레임(PVC)

- 영구 볼륨 클레임에서 지정된 조건을 기반으로 영구 볼륨에 대한 요청이 들어오면 스케줄러는 현재 가지고 있는 영구 볼륨에서 적당한 볼륨을 할당

### 영구 볼륨 클레임 설정

- 설정 항목
  - 레이블 셀렉터
  - 용량
  - 접근 모드
  - 스토리지클래스
- [기본 예시](./sample-pvc.yaml)
  - PVC 생성 시 PV 확보에 실패하면 Pending 상태로 유지

#### 파드에서 사용

- `spec.volumes[].persistentVolumeClaim.claimName`
  - [예시](./sample-pvc-pod.yaml)

### 동적 프로비저닝

- 영구 볼륨 클레임이 생성되는 타이밍에 동적으로 영구 볼륨을 생성하고 할당
  - 사전에 미리 영구 볼륨을 생성할 필요가 없어짐
  - 용량 낭비가 발생하지 않음
- 동적 프로비저닝을 사용하려면 스토리지클래스 생성이 필요
  - 설정한 프로비저너로 동적 프로비저닝이 실행
  - [스토리지클래스 예시](./sample-storageclass.yaml)
  - [동적 프로비저닝 영구 볼륨 클레임 예시](./sample-pvc-dynamic.yaml)
  - [동적 프로비저닝된 PVC에 마운트하는 파드 예제](./sample-pvc-dynamic-pod.yaml)

#### 영구 볼륨 할당 타이밍 제어

- 기본적으로 PVC를 생성하면 바로 PV가 생성
  - 파드에서 실제로 PVC를 attach하지 않아도 PV를 생성하기 때문에 낭비가 발생
- 스토리지클래스의 `volumeBindingMode`를 통해 이를 제어 가능
  - `WaitForFirstConsumer` (기본값: `Immediate`)
  - [예시](./sample-storageclass-wait.yaml)

### 영구 볼륨을 블록 장치로 사용

- PV를 파일 시스템으로 attach하지 않고 블록 장치로 attach하는 것도 가능
  - `spec.volumeMode: Block` (기본값: `Filesystem`)
  - [예시](./sample-pvc-block.yaml)
- 블록 장치로 PVC를 사용할 때에는 `spec.containers[].volumeMounts` 대신 `spec.containers[].volumeDevices` 사용
  - [예시](./sample-pvc-block-pod.yaml)

### 영구 볼륨 클레임 조정을 사용한 볼륨 확장

- 일부 플러그인에서는 영구 볼륨 클레임 확장(리사이즈)을 지원
  - 스토리지클래스에 `allowVolumeExpansion: true` 설정 필요
    - [예시](./sample-storageclass-resize.yaml)
  - 영구 볼륨 클레임은 별도의 설정이 필요없이 확장 가능한 스토리지클래스를 지정하기만 하면 됨
    - [예시](./sample-pvc-resize.yaml)
    - [파드 예시](./sample-pvc-resize-pod.yaml)
- <b>볼륨 확장은 가능하지만 축소는 불가</b>

### 스테이트풀셋에서 영구 볼륨 클레임(volumeClaimTemplate)

- 스테이트풀셋은 영구 데이터 영역을 사용하는 경우가 많기 때문에 `spec.volumeClaimTemplate`으로 자동으로 영구 볼륨 클레임을 생성 가능
- [예시](./sample-statefulset-with-pvc.yaml)

## volumeMounts에서 사용 가능한 옵션

### 읽기 전용 마운트

- [예시](./sample-readonly-volumemount.yaml)

### subPath

- 볼륨을 마운트할 때 특정 디렉터리를 루트로 마운트하는 기능
- [예시](./sample-subpath.yaml)
- 서로 다른 파드들이 동일한 경로로 마운트하더라도 호스트 레벨에서는 서로 영향을 주지 않도록 하고, 컨테이너 내에서도 동일한 동작을 공유할 수 있게 하기 위함
