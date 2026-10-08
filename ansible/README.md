# 당충전 Ansible

Ansible로 Kubernetes 클러스터를 구축/관리합니다.

- Ansible Control Node: jenkins-1
- Control Plane: control-plane
- Workers: worker-1 ~ worker-4

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
