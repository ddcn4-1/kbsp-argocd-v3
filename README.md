## 0. 문서 정보
- 버전 / 작성일 : v1 / 2025.11.26
- 작성자 : 윤효정
- 환경 : MacOS
- 목차 :
  1. Overview
  2. 아키텍처 & Flow
  3. Requirements
  4. 소스 코드 / 레포지토리 구조
  5. Setup & Installation
  6. Troubleshooting
  7. Appendix

---

## 1. Overview (개요)

### 목표
- 백엔드 애플리케이션의 ECR 이미지 버전이 변경되면, 해당 변경을 자동으로 감지해 Kubernetes Deployment의 이미지를 업데이트한다.
- 변경된 이미지 태그를 GitOps 레포지토리에 자동 커밋(write-back)하여 GitOps 상태와 클러스터 상태를 동기화한다.
  
### 실사용 영상
- 목표 달성 영상 추가 예정

---

## 2. 아키텍쳐 & Flow
### 파이프라인 흐름 
- 개발자가 GitHub Actions를 통해 새 Docker 이미지를 ECR에 push
- ArgoCD Image Updater가 ECR 레지스트리를 모니터링
- 최신 이미지 태그를 감지하면 Application의 kustomize(deployment) 파일 내 이미지 태그를 업데이트
- 업데이트된 kustomize 파일을 GitOps 레포지토리에 자동 커밋(write-back)
- ArgoCD가 변경된 GitOps 레포지토리를 감지하여 Kubernetes에 자동 배포(syncPolicy.automated)

### 아키텍쳐 다이어그램
- 전체 CD 플로우 다이어그램
- ArgoCD ↔ GitOps ↔ Kubernetes 구조

---

## 3. Requirements (사전 준비 사항)
### 시스템 요구사항
- Control Plane EC2에 다음 도구 설치
    - git
    - helm (Version: v3.18.4, GoVersion: go1.24.4)
- Kubernetes 클러스터 및 AWS 인프라는 terraform-v3 repo 기준으로 구성되어 있어야 한다.
    - 인프라 구조는 다음 repo 참고  
    - [https://github.com/ddcn4-1/terraform-v3/tree/kbsp-jy](https://github.com/ddcn4-1/terraform-v3/tree/kbsp-jy)
### 인증 정보 / 접근 권한
- - EC2 인스턴스 프로필(IAM Role)에 `AmazonEC2ContainerRegistryReadOnly` 필요
    - ArgoCD Image Updater가 ECR 이미지 목록을 조회하는데 필요
- GitHub → GitOps Write-Back을 위한 PAT
    - 권한: `repo` (read/write 모두 필요)

---

## 4. 소스 코드 / 레포지토리 구조
### 레포지토리 구조


```
.
├── root # ArgoCD App of Apps 구조
│   └── root-apps.yaml
├── apps 
│   ├── argocd-image-updater-app.yaml
│   └── aws-kubernetes-cluster
│       ├── *-app.yaml # admin, queue, main에 해당하는 Application manifest
├── argocd-image-updater 
│   └── values.yaml  # (참고용, 실제 배포에는 포함되지 않음)
└── values-arogcd.yaml # ArgoCD Helm 설치 설정 파일
├── k8s
│   └── aws-kubernetes-cluster # 쿠버네티스 리소스 파일들
│       ├── ...
│       └── ticket-app # ArgoCD Image updater 사용을 위한 kustomize wrapper
│           ├── admin
│           ├── main
│           └── queue
│               ├── config.yaml
│               ├── deployment.yaml
│               ├── kustomization.yaml
│               └── service.yaml
```

### 핵심 구조 설명
- App of Apps 방식으로 root-apps.yaml이 하위 모든 Application을 관리한다.
- argocd-image-updater/values.yaml은 원래 Helm values override 용도지만, remote Helm repo와 local values 병행 적용이 정상 동작하지 않아 현재는 `argocd-image-updater-app.yaml` 내부에 values를 포함하여 사용하고 있다.
- ticket-app 하위 admin/main/queue 폴더는 기존 directory 타입을 kustomize 타입으로 변환하여 Image Updater가 정상적으로 이미지를 추적할 수 있도록 구성했다

---

## 5. Setup & Installation

### step 1 Control Plane에 ArgoCD 설치

설치 버전:
- argo-cd chart: argo-cd-9.1.3 (app version v3.2.0)
- argocd-image-updater chart: argocd-image-updater-1.0.1 (app version v1.0.1)


Image Updater는 App of Apps 구조로 배포될 예정이므로 초기에는 ArgoCD만 설치한다.


**본 레포지토리를 controlplane에 git clone**
```
  git clone https://github.com/ddcn4-1/kbsp-argocd-v3.git
```    

**helm repo 추가**
```
    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update
```

**namespace 가 없다면 생성 **
```
    kubectl create namespace argocd
```

**ArgoCD 설치**
```
    helm install argocd argo/argo-cd \
      -n argocd \
      -f values-argocd.yaml # git clone 한 레포지토리 루트 폴더에 위치
```

**결과 확인**
```
    kubectl get pods -n argocd
    argocd-server            1/1 Running
    argocd-repo-server       1/1 Running
    argocd-application-controller 1/1 Running
    argocd-dex-server        1/1 Running
    argocd-redis             1/1 Running
```

**ArgoCD 업데이트가 필요한 경우**
```
    helm upgrade argo-cd argo/argo-cd -n argocd -f values-arogcd.yaml
```

### step 2 ArgoCD UI를 localhost:8080에서 확인
목표:
- Control Plane(Private)에 있는 kubeconfig(`admin.conf`)를 Bastion을 통해 로컬로 복사
- 로컬 PC에서 kubectl이 Control Plane에 접근할 수 있도록 SSH 터널링
- 로컬 PC에서 ArgoCD port-forward 실행.

**Control Plane에서 kubeconfig 위치 확인 (대부분 위치한 경로)**
```
    ls -l /etc/kubernetes/admin.conf
```

**(로컬 PC에서 실행) SSH config 작성 (~/.ssh/config)**
```
    Host bastion
        HostName <bastion-host-publicIP(elasticIP)>
        User <bastion ec2 username>
        IdentityFile ~/.ssh/<example-key.pem>

    Host controlplane
        HostName <controlplane-host-privateIP>
        User <controlplane ec2 username>
        IdentityFile ~/.ssh/<example-key.pem>
        ProxyJump bastion
```

**~./ssh 아래에 ec2 key 파일 추가 후 권한 부여**

`chmod 600 ~/.ssh/<example-key.pem>`


**(로컬 PC에서 실행) kubeconfig 다운로드 진행**
```
    scp controlplane:/etc/kubernetes/admin.conf ~/.kube/config
```

> 주의 :  Permission denied 가 뜨는 경우 Control Plane에서 파일 접근에 root 권한이 필요한 것일 수도 있습니다. 그 경우 
1. Control Plane에서 아래를 실행합니다.
```
    sudo cp /etc/kubernetes/admin.conf /home/ubuntu/admin.conf
    sudo chown ubuntu:ubuntu /home/ubuntu/admin.conf
```

2. 로컬에서 아래를 실행합니다
```
    scp controlplane:/home/ubuntu/admin.conf ~/.kube/config
```

**(로컬 PC에서 실행 ) Kubeconfig 권한 부여**
`chmod 600 ~/.kube/config`

**(로컬 PC에서 실행) 로컬 PC → Control Plane 6443 포트 터널 만들기**
이후 이 창에서 터널이 계속 열려 있어야 합니다
```
    ssh -i <BASTION_KEY.pem> -L 6443:<CONTROL_PLANE_PRIVATE_IP>:6443 <bastion-ec2-username>@<BASTION_PUBLIC_IP>
```

**(로컬 PC에서 실행) 다운받은 kubeconfig의 server 주소 변경 (~/.kube/config)**
```
    기존 :
    server: https://<CONTROL_PLANE_PRIVATE_IP>:6443
    
    변경:
    server: https://localhost:6443
```

> 주소 변경 후 로컬에서 kubectl get nodes 정상 출력 시 연결 성공된 것

**(로컬 PC에서 실행) ArgoCD port-forward**
```
    kubectl port-forward svc/argocd-server -n argocd 8080:443
```

> 접속 :  http://localhost:8080

### Step 3 ArgoCD 로그인

**controlplane에서 초기 비밀번호 확인 & username = admin**
```
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

### Step 4 ArgoCD에 github Write-Back을 위한 PAT 추가
- UI 상에서 setting > Repositories > connect repo 
- http/https 연결 방식으로 진행

<img width="1624" height="1056" alt="Pasted image 20251127002130" src="https://github.com/user-attachments/assets/d02088e9-6e94-43dc-8652-40b7a30617e4" />


### step 5 Controlplane에서 Root app 등록

**git repo의 root/root-apps.yaml을 사용**
```
    kubectl apply -f root-apps.yaml -n argocd
```


배포 후:
- Image Updater Pod 로그에서 write-back 정상 여부 확인
- 각 Application의 이미지 태그 변경 감지 확인

  
결과 :
<img width="2048" height="1172" alt="Pasted image 20251127010437" src="https://github.com/user-attachments/assets/27d7f0ea-2509-4632-bc18-780c7e3e247c" />

<img width="3216" height="2082" alt="Pasted image 20251126235910" src="https://github.com/user-attachments/assets/90683741-4f69-4eee-9e72-f3cddfd44227" />


## 6. Troubleshooting

### Image Updater가 directory 타입을 인식하지 못하는 문제

<img width="2048" height="652" alt="Pasted image 20251127011727" src="https://github.com/user-attachments/assets/d09ef7bd-696a-4b3f-b50e-96243227132f" />

문제 상황:
- 공식 문서(v1.0.1 기준)에서는 directory 타입 지원을 명시하지만, 실제 로그에서 type이 공백("")으로 인식되는 문제가 확인됨.

원인:
- Image Updater는 기본적으로 Helm, Kustomize, Plain YAML을 모두 지원하지만, Plain directory 내부에 여러 yaml이 있으면 어떤 파일이 이미지 패치를 위한 대상으로 사용될지 불명확해 오류가 발생할 수 있다. (공식 issue 참고 가능)
    
해결:
- directory → kustomize 타입으로 전환
- 각 서비스 폴더에 kustomization.yaml 추가


예시:
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

### ArgoCD가 Root App 아래 app들의 하위 리소스를 인지하지 못하는 문제

문제:
- UI에서 root → app 까지만 보이고, 하위 Kubernetes 리소스를 감지하지 못함
<img width="1608" height="1041" alt="Pasted image 20251127012824" src="https://github.com/user-attachments/assets/0435799a-c002-4df7-bb94-a3c4807d061a" />


원인:
- Application manifest에 `metadata.namespace: argocd`가 아닌 경우 ArgoCD Application이 정상적으로 컨트롤되지 않음  
    (ArgoCD는 Application CRD를 반드시 argocd namespace에서 관리해야 리소스를 추적함)

해결:
- apps/ 파일명 재확인 (`*-app.yaml`)
- Application manifest 구조 점검
```
# apps/*-app.yaml 형식
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
    name: # <new app name>
    namespace: argocd # 반드시 argocd로 유지해야 하위 리소스를 감지함
    annotations:
        argocd-image-updater.argoproj.io/image-list: |
            <alias>=<ECR Url>
        argocd-image-updater.argoproj.io/<alias>.update-strategy: newest-build
        argocd-image-updater.argoproj.io/write-back-method: git
spec:
    project: default 
    source:
        repoURL: 'https://github.com/ddcn4-1/kbsp-argocd-v3.git'
        targetRevision: main
        path: '<k8s/ 에서 앱의 deployment 파일이 존재하는 폴더경로>'
    destination:
        server: 'https://kubernetes.default.svc'
        namespace: default
    syncPolicy:
        automated:
            prune: true
            selfHeal: true

```

## 7. Appendix
### 참고 레퍼런스
- ArgoCD Image Updater v1.0.1  
    [https://argocd-image-updater.readthedocs.io/en/stable/](https://argocd-image-updater.readthedocs.io/en/stable/)    
- ArgoCD 공식 문서  
    [https://argo-cd.readthedocs.io/en/stable/](https://argo-cd.readthedocs.io/en/stable/)
  
### 용어 설명
- GitOps: Git 저장소를 단일 소스 오브 트루스로 사용하여 인프라/애플리케이션 상태를 선언적으로 관리하는 방식
- Image Updater: ArgoCD 확장 기능. Container Registry의 최신 이미지를 감지해 Application manifest를 자동으로 업데이트
- Write-back: 감지된 최신 이미지 태그를 GitOps repo에 자동 커밋
- App of Apps: 하나의 Root Application이 여러 Application을 생성/관리하는 ArgoCD 패턴
- Kustomize Wrapper: plain yaml 폴더를 kustomization.yaml 기반 구조로 변환하여 도구가 인식할 수 있도록 만든 구성
