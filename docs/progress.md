# 진행 보고서 — 온프레미스 3-Tier 인프라 구축

프로젝트 진행 내역을 주차별로 기록한다. 각 단계의 목표, 설계 판단, 검증 결과를 남겨 재현과 회고가 가능하도록 한다.

---

## 0주차 — 토대 구축

### 목표
전 노드의 원본이 될 골든 이미지를 만들고, Ansible 컨트롤 노드에서 모든 서버를 코드로 제어할 수 있는 관리 평면을 세운다.

### 수행 내역
- **OS 선정**: Rocky Linux 9.8 (RHEL 9 계열). 대안이던 Ubuntu는 apt/AppArmor 계열로 RHCSA 범위(dnf/SELinux/firewalld)와 겹치지 않아 제외. CentOS Stream은 RHEL 상류라 안정성 측면에서 제외.
- **네트워크 설계**: VM마다 어댑터 2개 구성. NAT(인터넷 출구, 패키지 설치용) + 호스트전용(192.168.56.0/24, VM 상호 통신 및 호스트 접근). 브리지 방식은 공유기 DHCP에 IP가 종속되고 VIP·페일오버 실험이 물리 네트워크로 새어나가 통제가 어려워 제외.
- **파티셔닝**: 설치 시 수동 LVM 구성. /boot 1GiB(표준 파티션), swap 2GiB(LVM), /(root) 17GiB(LVM). 이후 DB 데이터 볼륨 분리 실습의 기반.
- **SELinux**: enforcing 유지. 비활성화하지 않고 문제 발생 시 정책으로 해결하는 원칙 수립.
- **클론 전 초기화**: `/etc/machine-id` 비우기(truncate), SSH 호스트 키 삭제. 미실행 시 클론 간 machine-id·호스트 키 중복으로 journald·SSH 지문 충돌 발생.
- **Ansible 부트스트랩**: ansible-core 설치, ed25519 키 생성, `ssh-copy-id`로 전 노드 키 배포, ansible.cfg·inventory 구성.

### 검증
`ansible all -m ping` 전 노드 SUCCESS, `ansible all -m command -a "uptime"` 동시 실행 확인.

### 노드 구성
| 호스트 | IP | 역할 |
|---|---|---|
| ansible | 192.168.56.10 | 컨트롤 노드 |
| VIP | 192.168.56.20 | Keepalived 가상 IP |
| lb1 / lb2 | .21 / .22 | HAProxy + Keepalived |
| web1 / web2 | .31 / .32 | Nginx |
| db1 / db2 | .41 / .42 | PostgreSQL primary / replica |

---

## 1주차 — 웹 계층

### 목표
웹 서버 2대에 Nginx를 배포하되, 수동 설치 없이 전 과정을 Ansible 롤로 코드화한다.

### 설계 판단
- **왜 롤(role)인가**: ad-hoc 명령이나 셸 스크립트는 멱등성이 없어 재실행 시 상태가 깨진다. 롤은 설치·설정·방화벽·서비스를 표준 구조로 모듈화해 재사용과 재현을 보장한다.
- **hostname을 응답에 노출**: Jinja2 템플릿(`index.html.j2`)에서 `{{ ansible_hostname }}`을 사용해 각 서버가 자신의 이름을 응답 본문에 출력하도록 했다. 2주차에서 로드밸런싱 분산 여부를 육안으로 검증하기 위한 사전 작업.

### 구성 (roles/nginx)
- `tasks/main.yml`: dnf 설치(state: present) → 템플릿 배포(notify) → firewalld http 허용 → 서비스 기동(started + enabled)
- `templates/index.html.j2`: 서버별 hostname·IP 출력
- `handlers/main.yml`: 설정 변경 시에만 nginx 재시작

### 검증
`ansible-playbook web.yml` 실행 시 web1/web2 모두 failed=0. 브라우저에서 각 노드 접속 시 자신의 hostname 출력 확인. 재실행 시 changed=0으로 멱등성 확인.

### 장애 드릴 — 단일 장애점 체감
web1의 nginx를 정지시킨 결과, web2가 정상임에도 web1로 접속하던 요청은 전부 실패했다. 서버가 2대여도 각각 독립된 주소로 서비스되면 이중화가 아니며, 단일 진입점과 헬스체크 기반 분산이 필요하다는 것을 확인했다. 2주차 로드밸런서 도입의 근거.

---

## 2주차 — 로드밸런서 계층 및 이중화

### 목표
단일 진입점(VIP)을 만들고, 웹 이중화와 로드밸런서 자체의 이중화를 모두 구성한다.

### 설계 판단
- **왜 HAProxy인가**: Nginx도 로드밸런싱이 가능하나 본질은 웹 서버다. HAProxy는 로드밸런싱 전용 도구로 헬스체크·분산 알고리즘·통계 대시보드가 정교하다. 특히 stats 대시보드로 백엔드 UP/DOWN을 시각적으로 확인할 수 있어 헬스체크 검증에 유리하다.
- **두 계층의 서로 다른 이중화**: HAProxy 헬스체크는 "백엔드 웹 서버 장애"를 해결하고, Keepalived VIP 페일오버는 "로드밸런서 자체의 장애"를 해결한다. 서로 다른 문제를 푸는 별개의 메커니즘이다.
- **balance roundrobin 선택**: leastconn(연결 수 기준), source(세션 고정) 등의 대안이 있으나, 분산 동작을 명확히 관찰하기 위해 roundrobin을 채택.
- **host_vars 분리**: 동일한 keepalived 롤로 lb1은 MASTER(priority 110), lb2는 BACKUP(priority 100)이 되도록 host_vars에 역할별 변수를 두고, VIP·인터페이스 등 공통 값은 group_vars에 두었다.

### 구성
- `roles/haproxy`: 설치 → 설정 배포(validate로 사전 문법 검증) → SELinux 8404 포트 라벨링 → firewalld 80·8404 → 기동
- `roles/keepalived`: 설치 → VRRP 방화벽 허용 → 설정 배포 → 기동
- `templates/haproxy.cfg.j2`: frontend(:80) → backend(roundrobin, httpchk, check) + stats(:8404)
- `templates/keepalived.conf.j2`: vrrp_instance(state/priority 변수화, advert_int 1)

### 검증
- `curl` 반복 호출 시 web1/web2 교대 응답 확인 (브라우저에서는 keep-alive로 인해 한 서버에 고정되어 보이는 현상 관찰)
- HAProxy stats에서 백엔드 2대 UP 확인
- VIP(.20)로 접속 시 정상 서비스, `ip a`로 lb1에만 VIP 부착·lb2에는 미부착 확인

### 장애 드릴
1. **웹 서버 장애**: web1 nginx 정지 → stats에서 web1 DOWN 표시 → curl 응답이 web2로만 수렴, 서비스 무중단.
2. **로드밸런서 페일오버**: lb1의 keepalived 정지 → VIP가 lb2로 이동 확인 → lb1 복구 시 priority 우위로 VIP 회수(preempt). 1초 간격 curl 루프로 측정한 실제 다운타임은 약 1초(advert_int 1초 설정 기준, 이론상 최대 3초).

---

## 3주차 — DB 계층 및 복제

### 목표
DB 서버를 primary/replica로 분리하고, 데이터를 전용 LVM 볼륨에 격리한 뒤 스트리밍 복제를 구성한다.

### 설계 판단
- **왜 PostgreSQL인가**: 기존 MySQL 경험과 대비되는 스택을 다루기 위함. MySQL은 binlog(논리적 변경 기록) 기반 복제, PostgreSQL은 WAL(물리적 블록 변경 기록) 스트리밍 기반 복제로 동작 방식이 다르다. 계정 권한 모델도 MySQL이 `GRANT REPLICATION SLAVE`인 반면 PostgreSQL은 role 속성(`WITH REPLICATION`)과 접근 제어(pg_hba.conf)가 분리되어 있다.
- **데이터 볼륨 분리**: DB 데이터가 루트 파티션을 채우면 OS 전체가 마비된다. 가상 디스크(5GB)를 추가해 PV → VG(pgvg) → LV(pglv) → XFS → `/var/lib/pgsql` 마운트 순으로 전용 볼륨을 구성했다. LVM을 사용한 이유는 향후 디스크 추가로 무중단 확장이 가능하기 때문.
- **버전**: Rocky 9 기본 저장소의 PostgreSQL 13. 별도 저장소(PGDG)로 최신 버전을 설치할 수 있으나, 스트리밍 복제의 개념과 절차가 버전 간 거의 동일해 기본 저장소 버전을 채택.

### 구성 (roles/postgres — 설치 및 스토리지)
LVM 볼륨 생성(lvg/lvol) → XFS 포맷 → 마운트(fstab 등록) → PostgreSQL 설치 → `restorecon`으로 SELinux 컨텍스트 복원 → `PG_VERSION` 존재 여부로 initdb 중복 실행 방지 → 기동

새 볼륨을 마운트하면 SELinux 라벨이 없어 PostgreSQL이 접근하지 못하므로 `restorecon`이 필수다. 또한 `initdb`는 이미 초기화된 디렉토리에 재실행하면 실패하므로 `stat` + `when` 조건으로 멱등성을 확보했다.

### 구성 (roles/postgres-replication — 복제)
- `tasks/main.yml`: `pg_role` 변수로 `include_tasks` 분기 (primary.yml / replica.yml)
- `primary.yml`: psycopg2 설치 → conf.d/replication.conf 배포 → include_dir 등록 → pg_hba.conf 배포 → firewalld → 복제 role 생성
- `replica.yml`: `standby.signal` 존재 확인 → (미구성 시에만) 서비스 정지 → 데이터 디렉토리 정리 → pg_basebackup(-R) → 기동
- `templates/postgresql.conf.j2`: listen_addresses, wal_level=replica, max_wal_senders, **wal_keep_size**, **hot_standby=on**
- `templates/pg_hba.conf.j2`: 복제 접근을 replica IP(/32)와 scram-sha-256 인증으로 제한

**설정 모듈화**: postgresql.conf 본문을 직접 수정하지 않고 `conf.d/replication.conf`로 분리한 뒤 `include_dir`로 포함시켰다. 원본을 건드리지 않아 안전하고, 복제 설정만 독립적으로 관리·제거할 수 있다.

**wal_keep_size / hot_standby 추가**: replica가 일시적으로 끊겼다 재접속할 때 primary가 해당 구간 WAL을 이미 삭제했다면 복제가 깨져 basebackup을 다시 해야 한다. wal_keep_size로 WAL을 일정량 보관해 짧은 단절은 자동 복구되도록 했다. hot_standby는 replica에서 읽기 쿼리를 허용해 읽기 부하 분산이 가능하게 한다.

**파괴적 작업 방어**: pg_basebackup은 데이터 디렉토리를 비우고 다시 받는 파괴적 작업이다. 이미 구성된 replica에 재실행되면 정상 복제 중인 데이터를 삭제하게 되므로, `standby.signal` 파일 존재 여부로 관련 task 전체를 건너뛰도록 가드를 걸었다.

### 검증
- `pg_basebackup` 완료 (25MB 전송, 1 tablespace)
- db2에서 `pg_is_in_recovery()` → `t` (standby 모드 확인)
- **복제 동작 검증**: db1에서 `CREATE TABLE repltest` + `INSERT` 수행 후, db2에서 동일 데이터 조회 성공. replica에는 아무 작업도 하지 않았음에도 primary의 변경이 반영되는 것을 확인.
- 롤 검증: `--syntax-check` 통과, `--check`(dry-run)에서 db1은 primary 분기·db2는 replica 분기로 정확히 나뉘고, db2의 파괴적 task가 전부 skip되는 것을 확인.

### 보안 처리
- 복제 계정 비밀번호를 `group_vars/db.yml`에 두고 **ansible-vault로 암호화**. 평문 자격증명이 저장소에 올라가지 않도록 처리.
- pg_hba.conf의 복제 접근 규칙을 대역 전체·trust에서 **replica 단일 IP(/32) + scram-sha-256**으로 축소. 복제 접속은 DB 전체를 복제할 수 있는 권한이므로 최소 권한 원칙을 적용.

### 인프라 이전
실습 중 VM 저장 위치를 HDD에서 SSD로 이전했다. DB 계층은 잦은 디스크 쓰기(데이터 파일, WAL)가 발생해 I/O가 병목이 될 수 있어, VirtualBox의 머신 이동 기능으로 전 노드를 옮겼다. 파일 직접 복사 대신 이동 기능을 사용한 이유는 VirtualBox 내부 경로 등록이 자동 갱신되어 설정이 깨지지 않기 때문.

---

## 진행 현황

- [x] 0주차 — 골든 이미지, Ansible 관리 평면
- [x] 1주차 — 웹 계층 (Nginx)
- [x] 2주차 — 로드밸런서 계층 (HAProxy + Keepalived VIP)
- [x] 3주차 — DB 계층 (PostgreSQL 복제 + LVM 볼륨 분리)
- [ ] 4주차 — 자동 백업 (pg_dump 스케줄링 + 복구 리허설)
- [ ] 5주차 — 관측 계층 (Prometheus + Grafana + Loki)
- [ ] 6주차 — 네트워크 심화 (VLAN, 방화벽 정책)
- [ ] 7주차 — 통합 문서화 및 아키텍처 다이어그램
