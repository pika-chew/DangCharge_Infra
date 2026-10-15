# 당충전 Ansible

Ansible로 Kubernetes 클러스터를 구축/관리합니다.

- Ansible Control Node: jenkins-1
- Control Plane: control-plane
- Workers: worker-1 ~ worker-4

---
<br>

## Control node setup

Ansible을 설치하고, 이 디렉토리를 통해 플레이북 실행

```bash
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install ansible
ansible-galaxy collection install -r collections/requirements.yml

ansible all -m ping
ansible-playbook playbooks/site.yml
```
플레이북 실행하기 전, 모든 Kubernetes 노드의 호스트 키를 `~/.ssh/known_hosts`에 등록하고 확인.

---
<br>

## Jenkins setup

```
ansible-playbook playbooks/jenkins-docker.yml
```
`jenkins-docker.yml` 플레이북 실행 후 Docker 그룹권한을 적용하기 위해 jenkins-1 VM에 다시 로그인해야 함.

---
<br>

## Argo CD setup

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook playbooks/argocd.yml
```
control-plane에서 `admin.conf`로 Helm 설치, PriorityClass 생성, argo-cd 차트 설치 수행
Helm과 Python kubernetes 라이브러리(`/opt/argocd/venv`)는 role에서 설치

- 버전: `roles/argocd/defaults/main.yml` (차트, Helm, Python kubernetes)
- PriorityClass / 리소스 / Service: `roles/argocd/templates/values.yaml.j2`
- VIP: `roles/argocd/defaults/main.yml`의 `argocd_server_vip` (MetalLB `argocd-vip` 풀은 GitOps repo에서 관리)
- UI 접속: `https://<Argo CD VIP의 Floating IP>`

NodePort에서 LoadBalancer로 처음 바꿀 때는 기존 NodePort(30080/30443)가 Service에 그대로 남는다
Traefik NodePort와 겹치지 않도록 한 번만 Service를 지우고 playbook을 다시 실행해 새로 만든다
```bash
kubectl -n argocd delete service argocd-server
ansible-playbook playbooks/argocd.yml
```

### 설치 후 할 일

1. **설치 확인 및 admin 비밀번호 조회** (control-plane에서 실행)
   ```bash
   ssh ubuntu@10.20.0.110
   kubectl -n argocd get pods
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
   ```
   jenkins-1에는 kubeconfig가 없으므로 control-plane에서 실행
   `ubuntu` 사용자의 `~/.kube/config`(control-plane role이 `admin.conf`를 복사해 둔 것)를 사용

   UI(`https://<Argo CD VIP의 Floating IP>`)에 `admin`으로 로그인 후 비밀번호 변경
   (추후 이 단계를 role의 task로 추가 가능)

2. **GitOps repo 등록 및 Root Application 생성**
   아래의 별도 GitOps bootstrap 플레이북으로 실행

3. **앱 네임스페이스 ResourceQuota**
   이 role이 아니라 GitOps repo에서 관리할 항목

### GitOps bootstrap

기존 `playbooks/argocd.yml`은 설치 전용으로 유지합니다.
별도 `playbooks/argocd-bootstrap.yml`은 설치된 Argo CD에 GitOps의 시작점만 등록합니다.
Ansible이 적용하는 리소스는 AppProject 3개, 선택적인 저장소 인증 Secret, root Application입니다.
하위 Application 생성, 서비스 배포, 지속적인 동기화는 Argo CD가 담당합니다.

```bash
# ansible 디렉터리에서 실행
ansible-playbook playbooks/argocd-bootstrap.yml --syntax-check
ansible-playbook playbooks/argocd-bootstrap.yml --check
ansible-playbook playbooks/argocd-bootstrap.yml

# root를 적용하지 않고 프로젝트만 준비
ansible-playbook playbooks/argocd-bootstrap.yml -e '{"argocd_bootstrap_apply_root": false}'
```

#### 실행 위치와 기본 설정

현재 인벤토리는 `jenkins-1`을 Ansible 실행 머신으로 가정합니다.
다른 머신에서도 Git 접근과 control plane에 대한 SSH·sudo 접근을 준비하면 실행할 수 있습니다.
인벤토리의 SSH 키 경로도 실행 머신에 맞아야 합니다.
부트스트랩은 이번 실행에 포함된 control plane 중 첫 번째 노드에서만 실행합니다.
특정 노드를 선택하려면 `--limit <control-plane 인벤토리 이름>`을 사용합니다.
기존 설치 플레이북은 변경하지 않습니다.

기본값은 `roles/argocd_bootstrap/defaults/main.yml`에서 관리합니다.

- control plane의 kubeconfig: `/etc/kubernetes/admin.conf`
- control plane의 Kubernetes 모듈용 Python: `/opt/argocd/venv/bin/python`
- Argo CD 네임스페이스: `argocd`
- GitOps 저장소: `https://github.com/pika-chew/DangCharge_GitOps.git`, 조회 브랜치: `main`

기존 설치 role의 defaults는 자동으로 로드되지 않습니다.
인벤토리나 실행 인자로 지정한 `argocd_kubeconfig`, `argocd_venv_dir`, `argocd_namespace`는
부트스트랩 기본값에도 반영됩니다. 필요하면 `argocd_bootstrap_*` 변수를 직접 지정합니다.
root의 추적 브랜치와 대상 클러스터(`https://kubernetes.default.svc`)는 GitOps 원본에서 관리합니다.
조회 브랜치를 바꾼다면 root가 추적하는 브랜치와도 맞춰야 합니다.

#### 실행 흐름

1. 기존 kubeconfig·Python, Application/AppProject CRD의 `Established=True`와
   Argo CD server/repo-server/application-controller/redis Pod Ready를 확인합니다.
   각 확인은 기본 재시도 60회, 간격 5초입니다.
   redis는 업무용 DB가 아니라 Argo CD 내부 캐시입니다. 외부 Redis나 HA 구성은 검사 조정이 필요합니다.
2. 실행 머신의 임시 디렉터리에 원격 브랜치를 내려받습니다.
   GitOps YAML 복사본을 Infra에 유지하지 않으며, 로컬 미커밋 변경은 사용하지 않습니다.
   임시 체크아웃은 성공·실패 후 삭제합니다. 실행 머신에는 Git이 필요합니다.
3. 부트스트랩 YAML의 종류·이름·네임스페이스를 확인하고, 명시적으로 제공한 저장소 인증을 준비합니다.
4. GitOps의 `bootstrap/root-project.yaml`, `infra-project.yaml`, `workloads-project.yaml`을 순서대로 적용합니다.
5. `bootstrap/root-app.yaml`을 적용합니다. 이후 동기화와 리소스 배포는 Argo CD가 수행합니다.

완료 메시지는 부트스트랩 리소스 등록 성공만 의미합니다. root Synced나 전체 서비스 준비 완료를
기다리지 않습니다. `--check`는 예상 변경만 확인하며 Kubernetes 리소스를 쓰지 않습니다.
이 모드도 클러스터 읽기 권한과 Git 접근이 필요하고, 임시 체크아웃은 생성 후 삭제합니다.

#### 실행 전 준비 조건

root 적용 즉시 GitOps에 정의된 자동 동기화가 시작됩니다.
이 role은 앱 이름·개수, placeholder, 하위 앱 동기화 정책, NodePort 충돌을 검사하거나 차단하지 않습니다.
GitOps 검증과 클러스터 준비는 실행 전에 별도로 완료해야 합니다.

- 미완성 앱은 root 탐색에서 제외하거나 배포 가능한 상태로 완성한 뒤 원격 main에 반영합니다.
  `skip-reconcile`만으로 root의 Application 생성을 막을 수는 없습니다.
- Argo CD와 Traefik의 NodePort 충돌이 실제 클러스터에서도 해소됐는지 확인합니다.
- DB Secret·노드 저장 경로·PV·Neutron VIP 등 배포 전제 조건을 준비합니다.
  이 role은 이를 생성·초기화하거나 기존 DB 데이터·비밀번호를 변경하지 않습니다.

하위 앱의 자동·수동 동기화 정책은 GitOps 원본이 결정합니다. Ansible은 하위 앱을 직접 동기화하거나
자동 동기화로 변경하지 않습니다. 현재 수동 앱인 `metallb`, `metallb-config`, `mysql`, `mongodb`는
준비 상태를 확인한 뒤 별도로 동기화합니다. sync-wave만으로 하위 앱의 준비 완료 순서를 보장하지 않습니다.
MongoDB 완료는 전체 Sync의 bootstrap hook 성공과 1 PRIMARY + 2 SECONDARY까지 별도로 확인합니다.

프로젝트만 적용하는 옵션은 기존 root를 중지하지 않습니다. 도중에 실패해도 앞서 적용한 리소스는 유지됩니다.
기존 클러스터에 연결할 때는 GitOps 원본과 기존 Application 정의가 일치하는지 먼저 확인합니다.

#### 선택적인 HTTPS 저장소 인증

Ansible 실행 머신의 Git 인증과 Argo CD의 저장소 인증은 별개입니다.
실행 머신에는 Git credential helper 등을 별도로 준비하고, URL에 토큰을 넣지 않습니다.
이 role은 HTTPS 저장소 URL을 사용합니다.

공개 저장소 또는 이미 Argo CD에 인증이 등록된 저장소는 추가 인증 변수를 전달하지 않습니다.
아래 변수를 전달하면 repository Secret을 생성·갱신하고, 생략하면 기존 인증을 건드리지 않습니다.
이 role은 기존 인증이나 Argo CD에서의 저장소 연결 성공 여부를 검사하지 않습니다.

```yaml
# 외부 파일을 작성한 뒤 ansible-vault encrypt로 암호화합니다.
# 실제 비밀값은 Git에 커밋하거나 명령행 -e 인자로 전달하지 않습니다.
argocd_bootstrap_repository_credentials:
  username: <GitHub 사용자명>
  password: <저장소 읽기 권한 토큰>
```

```bash
ansible-playbook playbooks/argocd-bootstrap.yml \
  -e @/secure/gitops-vault.yml --ask-vault-pass
```

기본 Secret 이름은 `dangcharge-gitops`입니다. 비밀값 처리에는 `no_log`를 적용하고,
`data`로 전달해 평문 인증값이 last-applied annotation에 저장되지 않게 합니다.

---
<br>

## Remove worker-4 from Kubernetes

`worker-4`를 다른 용도로 전환하기 전 클러스터에서 제외하고 Kubernetes 구성 요소를 정리

```bash
ansible-playbook playbooks/decommission-worker.yml -e worker_reset_reboot=true
```
worker-4를 대상으로 kube-proxy 이미지 준비, cordon, drain, 노드 삭제, `kubeadm reset`, Kubernetes·containerd·CNI 설정 및 패키지 제거 수행
kube-proxy 이미지가 없으면 Node 삭제 전에 내려받습니다

Node 삭제 뒤에 실패하더라도 같은 명령으로 재실행 가능
`--limit worker-4`로 첫 플레이를 생략한 경우에는 클러스터에 worker-4가 남아 있으면 초기화 전에 중지됩니다
