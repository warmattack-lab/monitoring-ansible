# monitoring-ansible

Terraform 없이 Ansible로 Prometheus, Grafana, Node Exporter 스택을 배포하는 프로젝트입니다.

## 🚀 사용법 한눈에 보기
아래 순서대로 실행하면 로컬에서 제공하는 ansible-control 컨테이너를 통해 모니터링 노드를 자동 구성할 수 있습니다.

> 📌 **사전 준비**: Ansible을 실행할 호스트에 Docker가 설치되어 있어야 합니다.

### 1) 컨테이너 이미지 준비
```bash
# ansible-control 이미지를 로컬에서 빌드
docker build -t ansible-control:local ansible-control/

# docker-compose.yml 의 image 태그도 동일한 이름으로 맞춰둡니다.
```

### 2) Ansible 컨테이너 시작
```bash
docker compose up -d
docker exec -it ansible-control bash
```

### 3) 인벤토리 설정
컨테이너 내부(`/workspace`)에서 인벤토리 파일을 편집합니다:
```bash
vi ansible/inventory/hosts.ini
```

**인벤토리 예시:**
```ini
[monitoring_server]
monitoring ansible_host=10.0.0.10 monitoring_compose_dir=/app/monitoring prometheus_data_dir=/data/prometheus grafana_data_dir=/data/grafana

[monitored_nodes]
node1 ansible_host=10.0.0.11
node2 ansible_host=10.0.0.12

[gpu_nodes]
gpu1 ansible_host=10.0.0.21 dcgm_exporter_enabled=true
gpu2 ansible_host=10.0.0.22 dcgm_exporter_enabled=true
```

- `monitoring_server`: Prometheus + Grafana가 설치될 모니터링 서버
- `monitored_nodes`: Node Exporter가 설치되어 메트릭을 수집할 대상 노드들
- `gpu_nodes`: NVIDIA GPU가 있는 노드 (dcgm_exporter_enabled=true 설정 시 DCGM-Exporter 배포)

### 4) 플레이북 실행
```bash
cd ansible
ansible-playbook -i inventory/hosts.ini playbooks/deploy-monitoring.yml
```

### 5) 서비스 접속
- **Prometheus**: `http://<monitoring 노드 IP>:9090`
- **Grafana**: `http://<monitoring 노드 IP>:3000`
  - 기본 로그인: `admin / admin`
- **Node Exporter**(각 노드): `http://<노드 IP>:9100/metrics`

| 서비스 | 기본 포트 | 비고 |
| --- | --- | --- |
| Prometheus | 9090 | 메트릭 수집/탐색 |
| Grafana | 3000 | Node Exporter Full 대시보드 자동 등록 |
| Node Exporter | 9100 | 시스템 메트릭 수집 |
| DCGM-Exporter | 9400 | NVIDIA GPU 메트릭 수집 (GPU 노드에만 배포) |

> 💡 **대시보드**: Grafana에는 Node Exporter Full 대시보드가 provisioning으로 자동 등록됩니다.

---

## 🧭 프로젝트 구조

```
monitoring-ansible/
├── docker-compose.yml              # Ansible 컨트롤 컨테이너 정의
├── ansible-control/               # Ansible 컨트롤 컨테이너 Dockerfile
│   └── Dockerfile
└── ansible/                       # Ansible 설정 및 플레이북
    ├── ansible.cfg                # Ansible 설정 (roles_path 등)
    ├── inventory/
    │   ├── hosts.ini              # 인벤토리 (서버/노드 정의)
    │   └── group_vars/
    │       └── all.yml            # 전역 변수 (Prometheus URL, 디렉터리 경로 등)
    ├── playbooks/
    │   └── deploy-monitoring.yml  # 메인 배포 플레이북
    └── roles/
        ├── monitoring_server/     # Prometheus + Grafana 역할
        │   ├── tasks/
        │   │   └── main.yml
        │   ├── templates/
        │   │   ├── prometheus.yml.j2
        │   │   ├── docker-compose.yml.j2
        │   │   ├── datasource.yml.j2
        │   │   └── dashboards.yml.j2
        │   └── files/
        │       └── node-exporter-full.json
        ├── node_exporter/         # Node Exporter 역할
        │   └── tasks/
        │       └── main.yml
        └── dcgm_exporter/         # DCGM-Exporter 역할 (GPU 모니터링)
            └── tasks/
                └── main.yml
```

### 주요 파일 설명

- **`ansible/ansible.cfg`**: Ansible 설정 파일. roles 경로를 `./roles`로 지정하여 상대 경로로 역할을 찾을 수 있도록 합니다.
- **`ansible/inventory/hosts.ini`**: 배포 대상 서버 및 노드 정의
- **`ansible/inventory/group_vars/all.yml`**: 모든 호스트에 적용되는 전역 변수 (Prometheus URL, 데이터 디렉터리 경로, Grafana 인증 정보 등)
- **`ansible/roles/monitoring_server/`**: Prometheus와 Grafana를 Docker Compose로 배포하는 역할
  - `prometheus.yml.j2`: Prometheus 설정 템플릿 (동적으로 monitored_nodes와 gpu_nodes를 targets에 추가)
  - `docker-compose.yml.j2`: Prometheus, Grafana, Node Exporter 컨테이너 정의
- **`ansible/roles/node_exporter/`**: 각 모니터링 대상 노드에 Node Exporter 컨테이너를 배포하는 역할
- **`ansible/roles/dcgm_exporter/`**: NVIDIA GPU가 있는 노드에 DCGM-Exporter 컨테이너를 배포하는 역할 (dcgm_exporter_enabled=true인 경우에만)

---

## 🔧 주요 기능

### 동적 타겟 구성
Prometheus 설정은 Jinja2 템플릿으로 생성되며, `inventory/hosts.ini`에 정의된 호스트들을 자동으로 scrape targets에 추가합니다.

- **`[monitored_nodes]`**: Node Exporter 타겟으로 자동 추가
- **`[gpu_nodes]`**: `dcgm_exporter_enabled=true`인 호스트만 DCGM-Exporter 타겟으로 추가

**템플릿 예시 (`prometheus.yml.j2`):**
```jinja
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets:
        - 'localhost:9100'
{% for h in groups['monitored_nodes'] | default([]) %}
{% if h not in groups['monitoring_server'] | default([]) %}
        - '{{ hostvars[h].ansible_host | default(h) }}:{{ hostvars[h].node_exporter_port | default(9100) }}'
{% endif %}
{% endfor %}

  - job_name: 'dcgm_exporter'
    static_configs:
      - targets:
{% for h in groups['gpu_nodes'] | default([]) %}
{% if hostvars[h].dcgm_exporter_enabled | default(false) | bool %}
        - '{{ hostvars[h].ansible_host | default(h) }}:{{ hostvars[h].dcgm_exporter_port | default(9400) }}'
{% endif %}
{% endfor %}
```

### 권한 분리
- Prometheus 데이터 디렉터리: `65534:65534` (nobody 사용자)
- Grafana 데이터 디렉터리: `472:472` (Grafana 컨테이너 기본 사용자)

---

## 🐛 디버깅

### 템플릿 렌더링 테스트
Prometheus 설정 템플릿이 올바르게 렌더링되는지 확인:
```bash
cd ansible
ansible monitoring -i inventory/hosts.ini -m template \
  -a "src=roles/monitoring_server/templates/prometheus.yml.j2 dest=/tmp/test.yml" \
  -c local

cat /tmp/test.yml
```

### 변수 확인
특정 호스트의 변수를 확인:
```bash
ansible monitoring -i inventory/hosts.ini -m debug -a "var=prometheus_url"
```

### 플레이북 Dry-run
실제로 변경하지 않고 플레이북 실행 시뮬레이션:
```bash
ansible-playbook -i inventory/hosts.ini playbooks/deploy-monitoring.yml --check --diff
```

---

## 📝 커스터마이징

### 변수 수정
`ansible/inventory/group_vars/all.yml`에서 다음 변수를 수정할 수 있습니다:

```yaml
prometheus_data_dir: "/data/prometheus"
grafana_data_dir: "/data/grafana"
monitoring_compose_dir: "/app/monitoring"
grafana_admin_user: "admin"
grafana_admin_password: "admin"
prometheus_url: "http://{{ hostvars[groups['monitoring_server'][0]].ansible_host }}:9090"

# DCGM Exporter settings (GPU 모니터링)
dcgm_exporter_enabled: false  # 호스트별로 오버라이드 가능
dcgm_exporter_port: 9400
dcgm_exporter_image: "nvcr.io/nvidia/k8s/dcgm-exporter:4.4.2-4.7.0-ubuntu22.04"
```

### 포트 변경
Node Exporter 또는 DCGM-Exporter 포트를 변경하려면 인벤토리에서 호스트별로 변수를 추가:
```ini
[monitored_nodes]
node1 ansible_host=10.0.0.11 node_exporter_port=9101

[gpu_nodes]
gpu1 ansible_host=10.0.0.21 dcgm_exporter_enabled=true dcgm_exporter_port=9401
```

---

## 📦 배포 내용

### Monitoring Server (monitoring_server role)
- **Prometheus**: 메트릭 수집 및 저장
- **Grafana**: 시각화 및 대시보드
- **Node Exporter**: 모니터링 서버 자체의 시스템 메트릭

### Monitored Nodes (node_exporter role)
- **Node Exporter**: 각 노드의 시스템 메트릭 수집 (CPU, 메모리, 디스크, 네트워크 등)

### GPU Nodes (dcgm_exporter role)
- **DCGM-Exporter**: NVIDIA GPU 메트릭 수집 (GPU 사용률, 메모리, 온도, 전력 등)
  - 요구사항: NVIDIA GPU, NVIDIA Driver, Docker with `--gpus` 지원
  - `dcgm_exporter_enabled=true`로 설정된 호스트에만 배포됨
