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

## 4주차 — 자동 백업 및 복구 리허설

### 목표
DB 논리 백업을 자동화하고, 실제로 데이터를 삭제한 뒤 백업에서 복구하는 리허설로 백업의 유효성을 검증한다.

### 설계 판단
- **백업은 replica(db2)에서 수행**: pg_dump는 DB 전체를 읽어 부하가 크다. primary에서 수행하면 서비스 성능에 영향을 주므로, 읽기가 가능한 replica(hot standby)에서 수행해 부하를 분리했다.
- **cron 대신 systemd timer**: 실습 VM은 자주 꺼져 있다. cron은 꺼져 있던 시간의 작업을 건너뛰지만, timer는 `Persistent=true`로 놓친 실행을 부팅 후 수행한다. 실행 결과가 journald에 남아 `journalctl -u`로 추적할 수 있다.
- **custom 형식(`pg_dump -Fc`)**: 압축되며, `pg_restore -t`로 특정 테이블만 선택 복구할 수 있다. 전체를 덮어쓰면 백업 이후의 정상 변경까지 되돌아가므로, 피해 범위만 복구하는 것이 원칙이다.
- **전역 객체 별도 백업**: pg_dump는 DB 내부 객체만 포함한다. 계정·권한 같은 클러스터 전역 객체는 `pg_dumpall --globals-only`로 따로 백업했다.
- **실패 시 즉시 중단**: 스크립트에 `set -euo pipefail`을 적용했다. 이것이 없으면 덤프가 실패해도 스크립트가 정상 종료되어, 손상된 백업을 정상으로 오인하게 된다.
- **보안**: 백업 경로를 데이터 디렉토리 밖(`/var/backups/pgsql`)에 두고 디렉토리 0700, `umask 077`로 파일을 postgres 전용으로 제한했다. 백업 파일에는 DB 전체 데이터가 들어 있다.
- **보관 정책**: `find -mtime +7 -delete`로 7일 경과분을 자동 삭제해 디스크 고갈을 방지했다.

### 구성 (roles/pg-backup)
- `templates/pg_backup.sh.j2`: 비템플릿 DB 전체 개별 덤프 + 전역 객체 덤프 + 보관 기간 정리
- `templates/pg-backup.service.j2`: `Type=oneshot`, `User=postgres`(peer 인증으로 비밀번호 없이 로컬 접속)
- `templates/pg-backup.timer.j2`: `OnCalendar=*-*-* 03:00:00`, `Persistent=true`
- `tasks/main.yml`: 백업 디렉토리 생성 → 스크립트 배포 → 유닛 배포 → `daemon_reload` 후 timer 활성화
- `backup.yml`: `hosts: db2`

### 검증
- `systemctl list-timers`로 다음 실행 시각(03:00) 등록 확인
- `systemctl start pg-backup.service`로 예약 실행과 동일한 경로를 수동 트리거
- `journalctl -u pg-backup.service`에서 `backup done` 로그 확인
- `/var/backups/pgsql`에 DB 덤프와 전역 객체 파일 생성 확인

### 복구 리허설 — "복제는 백업이 아니다"
1. `pg_restore -l`로 백업 파일에 대상 테이블(repltest)과 데이터가 포함되어 있음을 **복구 전에 먼저 확인**
2. primary(db1)에서 `DROP TABLE repltest` 실행 (운영자 실수 시뮬레이션)
3. replica(db2)에서도 테이블이 사라진 것을 확인. **복제는 DROP까지 그대로 전파**하므로 사람의 실수에 대한 대비책이 되지 못한다.
4. 백업 파일을 db2 → 컨트롤 노드 → db1로 전송 (`fetch` / `copy`). 전 구간 checksum이 동일해 전송 중 무결성 확인
5. primary에서 `pg_restore -d postgres -t repltest`로 해당 테이블만 복구
6. db1·db2 모두에서 데이터 복귀 확인. primary에 복구한 결과가 복제를 통해 replica에도 전파됨
7. 임시 복구 파일 즉시 삭제

**결론**: 복제는 가용성(서버 장애 대비), 백업은 복구 가능성(데이터 손상·실수 대비)을 담당하며 서로 대체할 수 없다.

### 한계 및 개선 과제
- **백업이 db2 로컬 디스크에만 존재**: db2 자체가 손상되면 백업도 함께 소실된다. 별도 호스트로의 복사(오프사이트 백업, 3-2-1 원칙)가 필요하다.
- **특정 시점 복구(PITR) 미지원**: 논리 백업은 백업 시각의 상태로만 복구된다. 03:00 백업 이후 발생한 변경은 복구되지 않는다. WAL 아카이빙을 구성하면 임의 시점으로 복구할 수 있다.
- **페일오버 시 백업 대상 고정**: 백업 대상이 `hosts: db2`로 고정되어 있어, replica가 primary로 승격되는 상황에서는 대상 조정이 필요하다.

---

## 5주차 — 관측 계층 (메트릭·알림·로그)

### 목표
전 노드의 상태를 한 곳에서 확인하고, 임계치를 넘으면 알림이 발생하며, 로그를 중앙에서 조회할 수 있는 관측 체계를 구축한다.

### 설계 판단
- **pull 방식 수집(Prometheus)**: 각 노드가 중앙으로 밀어 보내는 push 방식은 "보내지 않는 것"과 "노드가 죽은 것"을 구분하기 어렵다. Prometheus가 가지러 가는 pull 방식은 수집 실패 자체가 노드 다운의 신호가 되며(`up == 0`), 수집 대상 목록을 한 곳에서 관리할 수 있다.
- **로그는 push 방식(Promtail → Loki)**: 로그는 연속적으로 발생하는 스트림이므로, 에이전트가 발생 시점에 전송하는 편이 자연스럽다. 관측 대상의 성격에 따라 수집 방향이 달라진다.
- **ELK 대신 Loki**: Elasticsearch는 로그 본문 전체를 색인해 메모리 요구가 크다. Loki는 라벨만 색인하고 본문은 압축 저장해 자원 사용이 적으며, Grafana와 동일 벤더로 통합이 자연스럽다. VM 기반 실습 환경에 적합하다.
- **수집 대상 자동 생성**: Prometheus 설정 템플릿에서 Jinja2 반복문으로 인벤토리의 전체 호스트를 순회해 타겟을 생성한다. IP를 하드코딩하면 노드 추가 시 인벤토리와 감시 설정 두 곳을 수정해야 하고, 누락 시 해당 노드가 조용히 감시에서 빠진다. 인벤토리에 추가하고 플레이북을 실행하면 자동으로 감시 대상이 된다.
- **instance 라벨에 호스트명 사용**: 대시보드와 알림에 IP 대신 `web1`, `db2` 같은 이름이 표시되도록 라벨을 부여했다.
- **Grafana 데이터소스 프로비저닝**: 데이터소스를 UI에서 수동 등록하면 재현이 불가능하다. 프로비저닝 디렉토리에 설정 파일을 배포해 Grafana 기동 시 자동 등록되도록 했다.
- **전용 시스템 계정**: exporter·Prometheus·Alertmanager·Loki를 각각 전용 계정(`nologin`)으로 실행했다. 메트릭 수집에 root 권한은 불필요하다. 단, Promtail은 systemd journal 접근이 필요해 root로 실행했으며, 운영 환경에서는 `systemd-journal` 그룹 소속 전용 계정으로 축소하는 것이 바람직하다.

### 구성
| 역할 | 컴포넌트 | 배포 대상 | 포트 |
|---|---|---|---|
| 메트릭 노출 | node_exporter | 전 노드 | 9100 |
| 메트릭 수집·규칙 평가 | Prometheus | mon | 9090 |
| 알림 라우팅 | Alertmanager | mon | 9093 |
| 로그 전송 | Promtail | 전 노드 | - |
| 로그 저장 | Loki | mon | 3100 |
| 시각화 | Grafana | mon | 3000 |

- 바이너리 배포 컴포넌트는 `stat` 기반 조건으로 이미 설치된 경우 다운로드를 건너뛰도록 구성
- Prometheus 설정·규칙은 `promtool check config` / `check rules`로 배포 전 검증
- Alertmanager 설정은 `amtool check-config`로 검증
- 데이터 보관: Prometheus 15일, Loki 7일(compactor로 실제 삭제 수행)

### 알림 규칙
| 알림 | 조건 | 지속 |
|---|---|---|
| NodeDown | `up{job="node"} == 0` | 1m |
| DiskSpaceLow | 여유 공간 20% 미만 | 2m |
| HighMemoryUsage | 사용률 85% 초과 | 3m |

`for` 절로 지속 시간 조건을 둔 이유는 순간적인 값 변동으로 알림이 발생하는 것을 막기 위함이다. 빈번한 오탐은 알림 피로를 유발해 실제 장애 알림까지 무시하게 만든다.

### 검증 — 디스크 소진 장애 드릴
1. web2에서 `fallocate -l 13G`로 루트 파티션을 90% 사용 상태로 만듦 (여유 10.9%)
2. Prometheus에서 `DiskSpaceLow`가 **PENDING**으로 전환 (조건 충족, `for: 2m` 미경과)
3. 2분 경과 후 **FIRING**으로 전환
4. Alertmanager API에서 해당 알림 수신 확인. `summary`에 `{{ $labels.instance }}`가 치환되어 "web2 disk below 20% free"로 표시됨
5. 파일 삭제 후 알림이 자동으로 해제(resolved)되는 것까지 확인

규칙 등록에 그치지 않고 **발화부터 자동 해제까지 알림 생애주기 전체**를 검증했다.

### 검증 — 로그 수집
- Loki API로 `host` 라벨 값 조회 시 전체 노드 확인
- Grafana Explore에서 `{host="web1"}`, `{unit="haproxy.service"}` 등 노드·서비스 단위 조회 확인
- 다중 노드 통합 검색(`{job="systemd-journal"} |= "error"`)으로 기존에 노드별 SSH 접속이 필요했던 작업을 단일 쿼리로 대체

### 한계 및 개선 과제
- **알림 수신 채널 미구성**: Alertmanager의 receiver가 비어 있어 실제 전송은 이루어지지 않는다. 규칙 평가 → 라우팅까지의 파이프라인은 검증했으며, 운영에서는 SMTP 또는 Slack 웹훅을 연결하면 된다.
- **관측 계층 자체가 단일 장애점**: mon 노드가 다운되면 메트릭·로그·알림이 모두 중단된다. 관측 시스템의 이중화 또는 외부 모니터링이 별도로 필요하다.
- **애플리케이션 메트릭 부재**: 현재는 시스템 지표만 수집한다. HAProxy exporter, PostgreSQL exporter를 추가하면 요청 처리량·복제 지연 같은 서비스 지표까지 관측할 수 있다.

---

## 진행 현황

- [x] 0주차 — 골든 이미지, Ansible 관리 평면
- [x] 1주차 — 웹 계층 (Nginx)
- [x] 2주차 — 로드밸런서 계층 (HAProxy + Keepalived VIP)
- [x] 3주차 — DB 계층 (PostgreSQL 복제 + LVM 볼륨 분리)
- [x] 4주차 — 자동 백업 (systemd timer + 복구 리허설)
- [x] 5주차 — 관측 계층 (Prometheus + Grafana + Alertmanager + Loki)
- [ ] 6주차 — 네트워크 심화 (VLAN, 방화벽 정책)
- [ ] 7주차 — 통합 문서화 및 아키텍처 다이어그램
