# 온프레미스 3-Tier 고가용성 인프라 구축

> Ansible 자동화 기반 온프렘 3-tier 인프라. HAProxy 로드밸런싱, Keepalived VIP 페일오버, 이중화·장애복구 실습 포함.

리눅스 서버 중심 시스템 엔지니어를 목표로, 클라우드가 추상화해주던 계층(로드밸런싱, VIP 페일오버, 헬스체크)을 **직접 구축·이중화·자동화·장애복구**한 온프레미스 인프라 프로젝트입니다. 모든 구성은 Ansible 플레이북으로 코드화되어 있어 재현 가능합니다.

---

## 아키텍처
                사용자
                  │
         VIP (192.168.56.20)          ← 단일 진입점 (가상 IP)
          ╱                ╲
 lb1 (.21) MASTER  ⇄  lb2 (.22) BACKUP  ← Keepalived VRRP 페일오버
   HAProxy              HAProxy          ← L7 로드밸런싱 + 헬스체크
          ╲                ╱
    web1 (.31)        web2 (.32)         ← Nginx

    ansible (192.168.56.10)             ← 전 노드를 Ansible로 관리

**두 겹의 이중화**로 단일 장애점(SPOF)을 제거했습니다.

| 계층 | 이중화 메커니즘 | 해결하는 문제 |
|---|---|---|
| 웹 | HAProxy 헬스체크 | 웹 서버 한 대가 죽어도 무중단 |
| 로드밸런서 | Keepalived VIP 페일오버 | LB 한 대가 죽어도 약 1초 내 무중단 |

---

## 기술 스택

| 구분 | 사용 기술 |
|---|---|
| OS | Rocky Linux 9.8 (RHEL 9 계열) |
| 가상화 | VirtualBox |
| 자동화 (IaC) | Ansible (ansible-core, ansible.posix) |
| 웹 서버 | Nginx |
| 로드밸런서 | HAProxy (L7, roundrobin) |
| 이중화 | Keepalived (VRRP) |
| 보안/시스템 | SELinux (enforcing), firewalld, systemd, LVM |

---

## 노드 구성

| 호스트 | IP | 역할 |
|---|---|---|
| ansible | 192.168.56.10 | Ansible 컨트롤 노드 |
| **VIP** | **192.168.56.20** | Keepalived 가상 IP (대표 주소) |
| lb1 | 192.168.56.21 | HAProxy + Keepalived (MASTER) |
| lb2 | 192.168.56.22 | HAProxy + Keepalived (BACKUP) |
| web1 | 192.168.56.31 | Nginx |
| web2 | 192.168.56.32 | Nginx |

네트워크는 어댑터 2개(NAT + 호스트전용)로 구성해, 인터넷 출구와 내부 통신을 분리하고 VIP·페일오버 실험을 호스트전용 대역(192.168.56.0/24) 안에 격리했습니다.

---

## 프로젝트 구조

onprem-3tier/
├── ansible.cfg
├── inventory/hosts.ini # 관리 대상 노드 그룹 정의
├── group_vars/lb.yml # lb 그룹 공통 변수 (VIP, 인터페이스 등)
├── host_vars/
│ ├── lb1.yml # lb1: MASTER, priority 110
│ └── lb2.yml # lb2: BACKUP, priority 100
├── roles/
│ ├── nginx/ # 웹 서버 배포
│ ├── haproxy/ # 로드밸런서 (검증·SELinux·방화벽 포함)
│ └── keepalived/ # VIP 페일오버 (VRRP 방화벽 포함)
├── web.yml # 웹 계층 플레이북
└── lb.yml # 로드밸런서 계층 플레이북


---

## 실행 방법

Ansible 컨트롤 노드에서 SSH 키 배포 및 인벤토리 구성 후:

```bash
# 웹 계층 배포 (Nginx)
ansible-playbook web.yml

# 로드밸런서 계층 배포 (HAProxy + Keepalived)
ansible-playbook lb.yml
```

배포 검증:

```bash
# VIP를 통한 로드밸런싱 확인 (web1/web2 번갈아 응답)
for i in $(seq 1 6); do curl -s http://192.168.56.20 | grep "Served by"; done

# HAProxy 상태 대시보드
# 브라우저: http://192.168.56.20:8404/stats
```

---

## 주요 설계 결정

**왜 VM인가 (컨테이너 대신)**
시스템 엔지니어 관점의 프로젝트로, 부팅 순서·네트워크 스택·방화벽·SELinux 등 호스트 계층을 직접 다루기 위해 VM을 선택했습니다. 컨테이너는 이러한 계층을 커널/호스트에 위임하므로 학습 목표와 맞지 않습니다.

**왜 Ansible인가 (셸 스크립트 대신)**
멱등성(idempotency) 때문입니다. 셸 스크립트는 재실행 시 상태가 깨지지만, Ansible은 몇 번을 실행해도 결과가 동일합니다. `ansible-playbook lb.yml` 한 번으로 이미 구성된 노드는 건드리지 않고 신규 노드만 설정하는 것을 확인했습니다.

**왜 HAProxy인가 (Nginx LB 대신)**
로드밸런싱 전용 도구로, 헬스체크·분산 알고리즘·통계 대시보드가 정교합니다. 특히 stats 대시보드로 백엔드 UP/DOWN 상태를 시각적으로 확인할 수 있어 헬스체크 검증에 유리합니다.

**왜 Keepalived VIP인가**
HAProxy로 웹 이중화를 해도 LB가 1대면 LB 자체가 새로운 SPOF가 됩니다. VIP를 두 LB가 공유하고 VRRP로 페일오버시켜, 사용자는 단일 대표 주소만 바라보게 했습니다.

---

## 장애 시나리오 & 복구

각 계층 구축 후 의도적으로 장애를 유발해 복구 과정을 검증했습니다. 상세 트러블슈팅 로그는 [`docs/troubleshooting.md`](docs/troubleshooting.md) 참조.

| 시나리오 | 결과 |
|---|---|
| 웹 서버 1대 정지 | HAProxy 헬스체크가 감지 → 살아있는 서버로만 분산, 무중단 |
| LB MASTER 정지 | VIP가 BACKUP으로 페일오버 → 실측 다운타임 약 1초 |
| Keepalived split-brain | VRRP 차단이 원인 → firewalld rich_rule로 해결 |

---

## 진행 현황

- [x] 토대: Rocky 9 골든 이미지, Ansible 컨트롤 노드
- [x] 웹 계층: Nginx (Ansible 롤)
- [x] 로드밸런서 계층: HAProxy + Keepalived VIP 페일오버
- [ ] DB 계층: PostgreSQL 스트리밍 복제 + 자동 백업
- [ ] 관측 계층: Prometheus + Grafana + Loki
- [ ] 네트워크 심화 & 통합 문서화

> 실습 환경 특성상 `group_vars/lb.yml`의 인증 값은 평문으로 두었으며, 실무에서는 Ansible Vault로 암호화가 필요합니다.
