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
- PriorityClass / 리소스 / NodePort: `roles/argocd/templates/values.yaml.j2`
- UI 접속: `https://<노드 IP>:30443`

### 설치 후 할 일

1. **설치 확인 및 admin 비밀번호 조회** (control-plane에서 실행)
   ```bash
   ssh ubuntu@10.20.0.110
   kubectl -n argocd get pods
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
   ```
   jenkins-1에는 kubeconfig가 없으므로 control-plane에서 실행
   `ubuntu` 사용자의 `~/.kube/config`(control-plane role이 `admin.conf`를 복사해 둔 것)를 사용

   UI(`https://<노드 IP>:30443`)에 `admin`으로 로그인 후 비밀번호 변경
   (추후 이 단계를 role의 task로 추가 가능)

2. **GitOps repo 등록 및 Root Application 생성**
   GitOps repo 준비 후 추가 예정

3. **앱 네임스페이스 ResourceQuota**
   이 role이 아니라 GitOps repo에서 관리할 항목

### GitOps bootstrap (구현 2단계)

기존 `playbooks/argocd.yml`은 설치 전용으로 유지합니다.
별도 `playbooks/argocd-bootstrap.yml`이 `argocd_bootstrap` role을 실행합니다.
현재는 변수 검증과 실행 구조만 구현되어 있으며, Kubernetes 접속이나 리소스 적용은 수행하지 않습니다.
준비 상태 확인, 프로젝트 적용, root 적용 전 사전 점검은 다음 단계에서 추가합니다.

```bash
# 기존 설치 플레이북
ansible-playbook playbooks/argocd.yml

# 설치된 Argo CD의 부트스트랩 전용 플레이북 (현재는 변수 검증만 수행)
ansible-playbook playbooks/argocd-bootstrap.yml
```

부트스트랩은 이번 실행에 포함된 control plane 중 첫 번째 노드에서만 실행합니다.
특정 노드를 선택하려면 `--limit <control-plane 인벤토리 이름>`을 사용합니다.
기존 설치 플레이북에는 이 제한을 추가하지 않습니다.

기본값은 `roles/argocd_bootstrap/defaults/main.yml`에서 관리합니다.

- kubeconfig: `/etc/kubernetes/admin.conf`
- Kubernetes 모듈용 Python: `/opt/argocd/venv/bin/python`
- Argo CD 네임스페이스: `argocd`
- GitOps 저장소: `https://github.com/pika-chew/DangCharge_GitOps.git`, 브랜치: `main`
- 대상 클러스터: `https://kubernetes.default.svc`

기존 설치 role의 defaults는 자동으로 로드되지 않습니다.
인벤토리나 실행 인자로 지정한 `argocd_kubeconfig`, `argocd_venv_dir`, `argocd_namespace`는
부트스트랩 기본값에도 반영됩니다. 필요하면 `argocd_bootstrap_*` 변수를 직접 지정할 수 있습니다.
Python과 kubeconfig 경로는 선택된 control plane 안의 경로입니다.
GitOps YAML 복사본과 저장소 인증값은 이 role에 저장하지 않습니다.

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
