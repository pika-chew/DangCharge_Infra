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

## Remove worker-4 from Kubernetes

`worker-4`를 다른 용도로 전환하기 전 클러스터에서 제외하고 Kubernetes 구성 요소를 정리

```bash
ansible-playbook playbooks/decommission-worker.yml -e worker_reset_reboot=true
```
worker-4를 대상으로 kube-proxy 이미지 준비, cordon, drain, 노드 삭제, `kubeadm reset`, Kubernetes·containerd·CNI 설정 및 패키지 제거 수행
kube-proxy 이미지가 없으면 Node 삭제 전에 내려받습니다

Node 삭제 뒤에 실패하더라도 같은 명령으로 재실행 가능
`--limit worker-4`로 첫 플레이를 생략한 경우에는 클러스터에 worker-4가 남아 있으면 초기화 전에 중지됩니다
