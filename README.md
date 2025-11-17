# monitoring-ansible

Terraform 없이 Ansible로 Prometheus, Grafana 스택을 배포하는 프로젝트입니다.

## 🚀 사용법 한눈에 보기
아래 순서대로 실행하면 로컬에서 제공하는 ansible-control 컨테이너를 통해 모니터링 노드를 자동 구성할 수 있습니다.

> 📌 **사전 준비**: Ansible을 실행할 호스트에 Docker가 설치되어 있어야 합니다.

### 1) 컨테이너 이미지 준비
```bash
# ansible-control 이미지를 로컬에서 빌드
docker build -t ansible-control:local .

# docker-compose.yml 의 image 태그도 동일한 이름으로 맞춰둡니다.
```

### 2) Ansible 컨테이너 시작
```bash
docker compose up -d
docker exec -it ansible-control bash
```

### 3) 컨테이너 내부에서 플레이북 실행
```bash
cd /workspace
# inventory/hosts.ini 에 모니터링 노드 IP/유저/비밀번호 수정
ansible-playbook -i inventory/hosts.ini playbooks/deploy-monitoring.yml
```

### 4) 서비스 접속
- Prometheus: `http://<monitoring 노드 IP>:9090`
- Grafana: `http://<monitoring 노드 IP>:3000`

| 서비스 | 기본 포트 | 비고 |
| --- | --- | --- |
| Prometheus | 9090 | 메트릭 수집/탐색 |
| Grafana | 3000 | 사전 등록된 Node Exporter Full 대시보드 포함 |

> 💡 **대시보드**: Grafana에는 Node Exporter Full 대시보드가 provisioning으로 자동 등록됩니다.

---

## 🧭 구조 요약
- `docker-compose.yml`: Ansible 컨트롤 컨테이너 정의
- `ansible/`: 인벤토리, 플레이북, 역할 및 템플릿이 위치한 디렉터리
- `ansible/inventory/hosts.ini`: 배포 대상 노드 정보와 데이터 디렉터리 설정
