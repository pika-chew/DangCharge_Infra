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
   GitOps repo 준비 후 추가 예정

3. **앱 네임스페이스 ResourceQuota**
   이 role이 아니라 GitOps repo에서 관리할 항목

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
