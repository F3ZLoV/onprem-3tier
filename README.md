# 온프레미스 3-Tier 고가용성 인프라 구축

> Rocky Linux 9 VM 7대로 로드밸런서–웹–DB 3계층을 구축하고, 계층별 이중화·복제·백업·관측·접근제어를 Ansible로 코드화한 프로젝트.

클라우드 관리형 서비스가 대신 처리하던 계층(로드밸런싱, VIP 페일오버, 헬스체크, DB 복제)을 직접 구성하고, 각 계층마다 의도적으로 장애를 유발해 복구 과정을 검증했다. 전체 구성은 플레이북 실행만으로 재현된다.

---

## 아키텍처

```
                    사용자
                      │
             VIP (192.168.56.20)          ← 단일 진입점
              ╱                ╲
     lb1 (.21) MASTER  ⇄  lb2 (.22) BACKUP  ← Keepalived VRRP
       HAProxy              HAProxy          ← L7 로드밸런싱 + 헬스체크
              ╲                ╱
        web1 (.31)        web2 (.32)         ← Nginx
              │
        db1 (.41) ──WAL──▶ db2 (.42)         ← 스트리밍 복제
         primary            replica + 자동 백업

   ansible (.10)          mon (.50)
   플레이북 배포            Prometheus · Grafana · Alertmanager · Loki
```

모든 계층 간 접근은 firewalld rich rule로 출발지가 제한된다.

### 이중화 구조

| 계층 | 메커니즘 | 해결하는 문제 | 검증 결과 |
|---|---|---|---|
| 웹 | HAProxy 헬스체크 | 웹 서버 장애 | 1대 정지 시 무중단 |
| 로드밸런서 | Keepalived VIP 페일오버 | LB 자체 장애 | 실측 다운타임 약 1초 |
| DB | PostgreSQL 스트리밍 복제 | DB 장애 | 실시간 반영 확인 |
| DB | 일일 논리 백업 | 데이터 손상·조작 실수 | 삭제 후 복구 검증 |

---

## 기술 스택

| 구분 | 사용 기술 |
|---|---|
| OS | Rocky Linux 9 (RHEL 9 계열) |
| 가상화 | VirtualBox |
| 자동화 (IaC) | Ansible — 롤 8개, 플레이북 6개 |
| 웹 | Nginx |
| 로드밸런싱 | HAProxy (L7, roundrobin, HTTP 헬스체크) |
| 고가용성 | Keepalived (VRRP) |
| 데이터베이스 | PostgreSQL (스트리밍 복제) |
| 백업 | pg_dump + systemd timer |
| 관측 | Prometheus, Grafana, Alertmanager, Loki, Promtail |
| 보안 | SELinux (enforcing), firewalld (출발지 제한), Ansible Vault |
| 스토리지 | LVM (DB 데이터 볼륨 분리) |

---

## 노드 구성

| 호스트 | IP | 역할 |
|---|---|---|
| ansible | .10 | Ansible 컨트롤 노드 |
| **VIP** | **.20** | Keepalived 가상 IP |
| lb1 / lb2 | .21 / .22 | HAProxy + Keepalived (MASTER / BACKUP) |
| web1 / web2 | .31 / .32 | Nginx |
| db1 / db2 | .41 / .42 | PostgreSQL primary / replica |
| mon | .50 | 관측 스택 |

네트워크는 어댑터 2개(NAT + 호스트전용)로 구성해 인터넷 출구와 내부 통신을 분리하고, VIP·페일오버 실험을 호스트전용 대역(192.168.56.0/24)에 격리했다.

---

## 프로젝트 구조

```
onprem-3tier/
├── inventory/hosts.ini              # 노드 그룹 정의
├── group_vars/, host_vars/          # 공통 / 노드별 변수 (Vault 암호화 포함)
├── roles/
│   ├── nginx/                       # 웹 서버
│   ├── haproxy/                     # L7 로드밸런싱
│   ├── keepalived/                  # VIP 페일오버
│   ├── postgres/                    # LVM 볼륨 + PostgreSQL
│   ├── postgres-replication/        # primary/replica 분기 구성
│   ├── pg-backup/                   # 백업 스크립트 + systemd timer
│   ├── node-exporter/, promtail/    # 전 노드 에이전트
│   ├── prometheus/, alertmanager/   # 메트릭 수집 및 알림
│   ├── loki/, grafana/              # 로그 저장 및 시각화
│   └── firewall/                    # 접근 제어 정책
├── web.yml  lb.yml  db.yml          # 계층별 플레이북
├── backup.yml  monitoring.yml  firewall.yml
└── docs/
    ├── progress.md                  # 주차별 진행 보고서
    ├── troubleshooting.md           # 장애 대응 기록 17건
    └── study-notes.md               # 학습 정리
```

---

## 실행 방법

```bash
# 계층별 배포
ansible-playbook web.yml          # Nginx
ansible-playbook lb.yml           # HAProxy + Keepalived
ansible-playbook db.yml           # LVM + PostgreSQL
ansible-playbook backup.yml       # 자동 백업
ansible-playbook monitoring.yml   # 관측 스택
ansible-playbook firewall.yml     # 접근 제어 정책
```

검증:

```bash
# 로드밸런싱 (web1/web2 교대 응답)
for i in $(seq 1 6); do curl -s http://192.168.56.20 | grep "Served by"; done

# 복제 상태
ansible db2 -m shell -a 'sudo -u postgres psql -c "SELECT pg_is_in_recovery()"' -b

# 메트릭 수집 대상
ansible mon -m shell -a "curl -s 'localhost:9090/api/v1/targets?state=active' | grep -o '\"health\":\"up\"' | wc -l" -b
```

대시보드: Grafana `http://192.168.56.50:3000`

---

## 주요 설계 결정

**왜 VM인가 (컨테이너 대신)**
부팅 순서, 네트워크 스택, 방화벽, SELinux 등 호스트 계층을 직접 다루기 위해서다. 컨테이너는 이러한 계층을 커널과 호스트에 위임하므로 시스템 엔지니어링 관점의 학습 목표와 맞지 않는다.

**왜 Ansible인가 (셸 스크립트 대신)**
멱등성 때문이다. 셸 스크립트는 재실행 시 상태가 깨지지만, Ansible은 현재 상태를 확인하고 목표 상태로 수렴시킨다. 실제로 전 플레이북 재실행 시 `changed=0`으로 수렴하는 것을 검증했다.

**왜 HAProxy인가**
로드밸런싱 전용 도구로 헬스체크·분산 알고리즘·통계 대시보드가 정교하다. `mode http`(L7)를 사용한 실질적 이유는 HTTP 헬스체크다. L4는 포트 개방 여부만 확인할 수 있어, 프로세스는 살아 있으나 애플리케이션이 오류를 반환하는 상태를 감지하지 못한다.

**왜 Keepalived VIP인가**
HAProxy로 웹을 이중화해도 LB가 단일 구성이면 LB 자체가 새로운 단일 장애점이 된다. 두 LB가 가상 IP를 공유하고 VRRP로 페일오버해, 사용자는 단일 주소만 바라보게 했다.

**왜 pull 방식 수집인가 (Prometheus)**
각 노드가 중앙으로 전송하는 push 방식은 "전송할 것이 없는 상태"와 "노드가 다운된 상태"를 구분하기 어렵다. pull 방식은 수집 실패 자체가 노드 다운의 신호가 된다.

**왜 VLAN 대신 접근 제어인가**
정상 동작 중인 7노드의 네트워크를 재설계하는 위험 대비, 동일한 목적(계층 간 통신 최소화)을 firewalld 출발지 제한으로 달성할 수 있다고 판단했다.

---

## 장애 시나리오 및 검증

각 계층 구축 후 의도적으로 장애를 유발해 복구를 검증했다.

| 시나리오 | 수행 | 결과 |
|---|---|---|
| 웹 서버 장애 | web1 nginx 정지 | HAProxy가 자동 제외, 무중단 |
| LB 장애 | MASTER keepalived 정지 | VIP 페일오버, 다운타임 약 1초 |
| 디스크 소진 | 루트 파티션 90% 점유 | PENDING → FIRING → 자동 해제 전 과정 확인 |
| 데이터 삭제 | primary에서 DROP TABLE | **replica에도 전파됨** → 백업에서 선택 복구 |
| 접근 제어 | web 직접 접근 시도 | 차단, LB 경유 서비스는 정상 |

**복제는 백업이 아니다** — primary에서 테이블을 삭제하자 replica에서도 동일하게 사라졌다. 복제는 실수까지 전파하므로 가용성 대책일 뿐이며, 복구 가능성은 별도의 백업이 담당한다.

상세 기록: [`docs/troubleshooting.md`](docs/troubleshooting.md) (17건)

---

## 보안 조치

- **SELinux enforcing 유지** — 비활성화 없이 포트 라벨링(`semanage`)과 컨텍스트 복원(`restorecon`)으로 정책 준수
- **출발지 기반 접근 제어** — 포트 개방 여부가 아니라 허용 출발지를 명시
- **최소 권한 실행** — exporter·Prometheus·Alertmanager·Loki를 각각 전용 시스템 계정(`nologin`)으로 실행
- **자격증명 암호화** — 복제 계정 비밀번호를 Ansible Vault로 암호화, 복호화 키는 저장소 외부에 보관
- **불필요한 서비스 제거** — 미사용 cockpit, 클론 과정에서 딸려온 nginx 제거

---

## 한계 및 개선 과제

- **백업이 단일 호스트에만 존재** — replica 로컬 디스크에만 보관되어, 해당 노드 손상 시 백업도 소실된다. 오프사이트 복사가 필요하다.
- **특정 시점 복구(PITR) 미지원** — 논리 백업은 백업 시각으로만 복구된다. WAL 아카이빙으로 개선 가능하다.
- **알림 수신 채널 미구성** — Alertmanager까지의 파이프라인은 검증했으나 실제 전송은 미구성이다. SMTP 또는 웹훅 연결이 필요하다.
- **관측 계층이 단일 장애점** — mon 노드 다운 시 메트릭·로그·알림이 모두 중단된다.
- **애플리케이션 메트릭 부재** — 시스템 지표만 수집한다. HAProxy·PostgreSQL exporter 추가 시 요청 처리량, 복제 지연까지 관측할 수 있다.

---

## 문서

| 문서 | 내용 |
|---|---|
| [progress.md](docs/progress.md) | 주차별 진행 보고서 — 목표, 설계 판단, 검증 결과 |
| [troubleshooting.md](docs/troubleshooting.md) | 장애 대응 기록 17건 — 증상, 진단, 원인, 해결, 교훈 |
| [study-notes.md](docs/study-notes.md) | 구조와 개념 정리 |
