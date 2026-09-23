# 트러블슈팅 로그

구축 과정에서 실제로 겪은 장애와 해결 과정을 기록한다. 각 항목은 증상 → 진단 → 원인 → 해결 → 교훈 순으로 정리한다.

---

## 1. PostgreSQL 복제 접속이 pg_hba.conf 규칙을 무시 (난이도 최상)

### 증상
db2에서 db1로 복제 접속 시도 시 다음 오류가 반복 발생:

```
치명적오류: 호스트 "192.168.56.42", 사용자 "replicator",
SSL 중지 연결이 복제용 연결로 pg_hba.conf 파일에 설정되어 있지 않습니다
```

pg_hba.conf에 해당 규칙이 분명히 존재하는데도 매칭되지 않았다.

### 진단 과정
가설을 하나씩 세우고 배제해 나갔다.

1. **규칙 누락 여부** — `tail`/`grep`으로 확인. 규칙은 파일에 존재했다.
2. **규칙 순서 문제** — pg_hba.conf는 위에서부터 첫 매칭 규칙이 적용된다. 규칙을 파일 최상단(1번 줄)으로 이동시켰으나 동일 실패.
3. **인증 방식 불일치** — PostgreSQL 13의 기본 비밀번호 암호화는 scram-sha-256이므로 `md5` 지정이 원인일 가능성을 의심. `trust`(인증 생략)로 변경해 인증 계층을 통째로 배제했으나 동일 실패.
4. **주소 범위** — `192.168.56.42/32`를 `0.0.0.0/0`(전체 허용)으로 확대. 여전히 실패. 이 시점에서 정상적인 pg_hba 로직으로는 설명 불가능한 상태임을 확인.
5. **파싱 여부 확인** — `pg_hba_file_rules` 시스템 뷰로 PostgreSQL이 파일을 어떻게 해석하는지 직접 조회:

```sql
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules ORDER BY line_number;
```

결과: `95 | host | {replication} | {replicator} | 0.0.0.0 | trust` — 규칙이 **정상적으로 파싱되어 있었다.** 위쪽 규칙 중 이 접속을 가로챌 수 있는 것도 없었다.

6. **설정 반영 여부** — `SHOW hba_file`, `SHOW data_directory`로 경로 확인(일치), `pg_reload_conf()` 및 `systemctl restart` 수행, `pkill`로 잔여 프로세스 정리 후 재기동. 모두 정상이나 동일 실패.
7. **네트워크·클라이언트 배제** — db2의 `/etc/hosts` 정상, `ip route get 192.168.56.41` 결과 소스 IP가 `.42`로 정확, `/dev/tcp` 테스트로 5432 포트 연결 확인(PORT_OPEN).
8. **결정적 검증** — **db1이 자기 자신(192.168.56.41)에게 복제 접속**을 시도. 동일하게 거부되었다. 이로써 네트워크 경로, 클라이언트, 소스 IP 문제가 모두 배제되고 문제가 db1의 pg_hba.conf 자체에 있음이 확정되었다.

### 원인
디버깅 과정에서 `sed`로 파일을 여러 차례 반복 편집(삽입·삭제·치환)했고, 그 과정에서 파일에 눈에 보이지 않는 손상이 누적되어 PostgreSQL이 규칙을 실제 인증 판단에 적용하지 못하는 상태가 되었다. `pg_hba_file_rules` 뷰에는 정상으로 표시되었으나 런타임 매칭에는 반영되지 않았다.

### 해결
파일을 부분 수정하지 않고 heredoc으로 **전체를 새로 작성**한 뒤 소유권·권한을 정상화하고 재기동했다.

```bash
sudo tee /var/lib/pgsql/data/pg_hba.conf > /dev/null << 'HBA'
local   all             all                                     peer
host    all             all             127.0.0.1/32            ident
host    all             all             ::1/128                 ident
local   replication     all                                     peer
host    replication     replicator      192.168.56.0/24         trust
HBA
sudo chown postgres:postgres /var/lib/pgsql/data/pg_hba.conf
sudo chmod 600 /var/lib/pgsql/data/pg_hba.conf
sudo systemctl restart postgresql
```

재작성 직후 오류가 즉시 변경되었고(`"replication" 데이터베이스 없음` → 접속 문자열 문법 문제로 전환), 문법 수정 후 `IDENTIFY_SYSTEM`이 정상 응답하며 복제 경로가 확보되었다.

검증 이후 보안 설정을 좁혀 최종적으로는 replica 단일 IP(/32) + scram-sha-256 인증으로 재구성했다.

### 교훈
- 설정이 명백히 올바른데 동작하지 않으면, 설정 내용이 아니라 **파일 자체를 의심**해야 한다. 반복적인 스크립트 편집은 육안으로 확인되지 않는 손상을 만들 수 있다.
- 구성 파일은 부분 수정을 누적하기보다 **템플릿에서 전체를 생성**하는 방식이 안전하다. 이후 Ansible template 모듈로 관리하도록 전환한 이유이기도 하다.
- 문제 범위를 좁힐 때는 **변수를 하나씩 배제**한다. 인증 방식을 trust로 바꿔 인증 계층을 배제하고, 주소를 0.0.0.0/0으로 넓혀 범위를 배제하고, 최종적으로 서버가 자기 자신에게 접속하게 해 네트워크와 클라이언트를 배제했다.
- `pg_hba_file_rules` 뷰는 PostgreSQL의 파싱 결과를 직접 확인할 수 있는 유용한 진단 도구다.

---

## 2. 중첩 SSH 세션에서 primary 데이터 디렉토리 삭제 (사고 및 복구)

### 증상
db2를 replica로 구성하는 도중 db1(primary)의 데이터 디렉토리가 삭제되어 PostgreSQL이 다음 오류와 함께 동작 불능 상태가 되었다.

```
pg_basebackup: error: 치명적오류: "global/pg_filenode.map" 파일을 열 수 없음
```

### 원인
db2 작업 중 db1에 파일 정리를 위해 SSH로 접속했고(`ssh sysadmin@192.168.56.41`), 이어서 실행하려던 `exit`가 의도대로 처리되지 않은 상태에서 **db2용으로 준비한 명령 묶음을 그대로 붙여넣었다.** 결과적으로 `find /var/lib/pgsql/data -mindepth 1 -delete`가 db1에서 실행되어 primary의 데이터가 삭제되었다.

여러 줄 명령을 한 번에 붙여넣는 방식과, 원격 세션의 현재 위치를 확인하지 않은 것이 복합적으로 작용했다.

### 해결
실습 단계라 보존해야 할 데이터가 없었으므로 db1을 재초기화했다.

1. 잔여 프로세스 정리: `systemctl stop postgresql` → `pkill -u postgres`
2. 클러스터 재생성: `postgresql-setup --initdb`
3. 복제 설정 재적용: postgresql.conf 복제 항목 추가, pg_hba.conf 재작성(heredoc)
4. 복제 계정 재생성: `CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD ...`
5. 방화벽 확인 후 기동

LVM 마운트(`/var/lib/pgsql`)는 유지되었으므로 스토리지 구성은 재작업이 불필요했다.

### 교훈
- 원격 세션에서 파괴적 명령을 실행하기 전 **`hostname`으로 현재 위치를 확인**한다.
- 여러 줄 명령을 한 번에 붙여넣지 않는다. 특히 `exit`가 포함된 묶음은 세션 전환 실패 시 나머지 명령이 의도하지 않은 서버에서 실행된다.
- 이 사고를 계기로 Ansible 롤의 replica 구성에 **멱등성 가드**를 추가했다. `standby.signal` 존재 여부를 확인해 이미 구성된 replica에서는 정지·삭제·basebackup task를 모두 건너뛰도록 하여, 동일한 종류의 데이터 손실을 코드 수준에서 차단했다.

---

## 3. Keepalived split-brain — VIP가 양쪽 노드에 동시 부착

### 증상
Keepalived 배포 후 확인 결과 VIP(192.168.56.20)가 lb1과 lb2 **양쪽 모두**에 부착되어 있었다. 정상이라면 MASTER인 lb1에만 존재해야 한다.

### 진단
lb1의 keepalived 로그를 확인:

```
(VI_1) Receive advertisement timeout
(VI_1) Entering MASTER STATE
```

"상대 노드의 advertisement를 수신하지 못해 MASTER로 전환"한다는 의미다. lb2에서도 동일하게 동작해 양쪽이 모두 자신을 MASTER로 판단한 상태였다. 즉 두 노드 간 VRRP 통신이 차단되어 서로의 생존 신호를 받지 못하고 있었다.

### 원인
firewalld가 VRRP 트래픽을 차단하고 있었다. VRRP는 TCP/UDP 포트가 아니라 **IP 프로토콜 번호 112**를 사용하므로, 일반적인 포트 개방(`--add-port`, `--add-service`)으로는 허용되지 않는다.

### 해결
firewalld rich rule로 vrrp 프로토콜을 허용하고, 이를 Ansible 롤에 포함시켰다.

```yaml
- name: Allow VRRP protocol in firewalld
  ansible.posix.firewalld:
    rich_rule: rule protocol value="vrrp" accept
    permanent: true
    immediate: true
    state: enabled
```

적용 후 두 노드가 서로를 인식했고, priority가 낮은 lb2가 BACKUP으로 전환되며 VIP는 lb1에만 남았다.

### 교훈
- 방화벽 설정 시 프로토콜의 특성을 확인해야 한다. 포트 기반이 아닌 프로토콜(VRRP, OSPF 등)은 rich rule로 별도 허용이 필요하다.
- split-brain은 HA 구성의 대표적 장애 유형이다. 양 노드가 서로를 인식하지 못하면 각자 자신을 MASTER로 판단해 동일한 자원(VIP)을 동시에 점유하게 된다.
- 진단의 출발점은 로그다. "advertisement timeout" 한 줄이 통신 차단이라는 원인을 직접 가리켰다.

---

## 4. 브라우저에서 로드밸런싱이 동작하지 않는 것처럼 보이는 현상

### 증상
HAProxy 구성 후 브라우저에서 VIP로 접속해 새로고침을 반복했으나, 응답 서버가 약 10초 단위로만 바뀌어 roundrobin 분산이 제대로 동작하지 않는 것처럼 보였다.

### 원인
두 가지가 복합적으로 작용했다.

1. **HTTP keep-alive**: 브라우저는 성능을 위해 수립된 TCP 연결을 재사용한다. 동일 연결로 요청이 계속되면 HAProxy 입장에서는 이미 특정 백엔드로 연결된 세션이므로 새 분산 결정이 발생하지 않는다.
2. **브라우저 캐시**: 새로고침 시 캐시된 응답이 사용되어 서버로 요청이 도달하지 않는 경우가 있다.

### 해결
매 요청마다 새로운 연결을 생성하는 `curl`로 검증했다.

```bash
for i in $(seq 1 10); do curl -s http://192.168.56.20 | grep "Served by"; done
```

결과는 web1/web2가 정확히 교대로 응답하여 roundrobin이 정상 동작함을 확인했다.

### 교훈
- 로드밸런서 동작 검증은 브라우저가 아닌 연결을 재사용하지 않는 도구로 수행해야 한다.
- 장애로 보이는 현상이 실제로는 클라이언트 측 동작(연결 재사용, 캐시)인 경우가 있다. 검증 도구의 특성을 이해해야 잘못된 결론을 피할 수 있다.

---

## 5. Ansible 권한 승격 실패 — sudo 비밀번호 요구

### 증상
플레이북 실행 시 전 노드에서 실패:

```
sudo: 암호가 필요합니다 (MODULE FAILURE)
```

### 원인
플레이북에 `become: true`를 지정했으나, 대상 노드의 sysadmin 계정이 sudo 실행 시 비밀번호를 요구하는 상태였다. 자동화 실행 중에는 대화형 입력이 불가능하므로 권한 승격 단계에서 중단된다.

### 해결
두 가지 방식을 검토했다.

- `ansible-playbook -K` (`--ask-become-pass`): 실행 시 비밀번호를 한 번 입력받는다. 안전하지만 완전 자동화는 아니다.
- sudoers NOPASSWD 설정: 대상 노드에서 비밀번호 없이 sudo를 허용한다.

```bash
echo "sysadmin ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/sysadmin
```

본 프로젝트는 무인 재현이 목표이므로 후자를 채택했다. 격리된 실습 환경이라는 전제가 있으며, 운영 환경이라면 대상 명령을 한정하거나 Vault 기반 비밀번호 전달을 검토해야 한다.

### 교훈
- 자동화 도구의 권한 승격은 대화형 입력을 전제하지 않는다. 설계 시점에 승격 방식을 결정해야 한다.
- 편의성과 보안은 trade-off 관계이며, 환경의 성격에 따라 선택이 달라진다.

---

## 6. Ansible 롤 탐색 경로 불일치

### 증상

```
ERROR! the role 'nginx' was not found in /home/sysadmin/onprem-3tier/playbooks/roles: ...
```

### 원인
playbook을 `playbooks/` 하위에 배치했는데, Ansible은 **playbook 파일이 위치한 디렉토리를 기준으로** roles를 탐색한다. 실제 롤은 프로젝트 최상단 `roles/`에 있어 경로가 어긋났다.

### 해결
playbook을 프로젝트 최상단으로 이동해 `roles/`, `inventory/`, `ansible.cfg`와 같은 층에 두었다. 이는 Ansible 프로젝트의 일반적인 디렉토리 구조이기도 하다.

### 교훈
- Ansible의 상대 경로는 playbook 위치와 ansible.cfg 위치를 기준으로 해석된다. 프로젝트 구조를 표준 레이아웃에 맞추면 이런 문제를 피할 수 있다.
- 관련하여, `ansible` 명령은 ansible.cfg가 있는 디렉토리에서 실행해야 inventory 설정이 적용된다. 다른 위치에서 실행하면 "hosts list is empty" 경고와 함께 대상을 찾지 못한다.

---

## 7. 컬렉션 및 모듈 의존성 누락

### 증상
- `couldn't resolve module/action 'ansible.posix.firewalld'`
- `couldn't resolve module/action 'ansible.builtin.seport'`
- `Failed to import the required Python library (psycopg2)`

### 원인
ansible-core는 최소 코어만 포함하며, 다수의 모듈은 별도 컬렉션으로 분리되어 있다. 또한 일부 모듈은 대상 노드에 Python 라이브러리를 요구한다.

- `firewalld` → `ansible.posix` 컬렉션
- `lvg`, `lvol`, `filesystem` → `community.general` 컬렉션
- `postgresql_user` → `community.postgresql` 컬렉션 + 대상 노드의 `python3-psycopg2`
- `seport`는 `ansible.builtin`이 아닌 `community.general` 소속

### 해결
필요한 컬렉션을 설치했다.

```bash
ansible-galaxy collection install ansible.posix
ansible-galaxy collection install community.general
ansible-galaxy collection install community.postgresql
```

`seport`의 경우 컬렉션 추가 대신 내장 `command` 모듈로 `semanage`를 직접 호출하되, 멱등성을 수동으로 구현했다.

```yaml
- name: Allow haproxy stats port in SELinux
  ansible.builtin.command: semanage port -a -t http_port_t -p tcp 8404
  register: seport_result
  changed_when: "'already defined' not in seport_result.stderr"
  failed_when:
    - seport_result.rc != 0
    - "'already defined' not in seport_result.stderr"
```

psycopg2는 롤 내에서 `python3-psycopg2`를 먼저 설치하는 task를 앞단에 배치해 해결했다.

### 교훈
- `command`/`shell` 모듈은 Ansible이 멱등성을 보장하지 않으므로, `changed_when`·`failed_when`으로 직접 정의해야 한다. `semanage port -a`는 이미 등록된 포트에 대해 오류를 반환하므로 이를 실패로 간주하지 않도록 처리했다.
- 모듈이 요구하는 라이브러리 의존성은 롤 내부에서 선행 설치하도록 구성하면 재현성이 높아진다.

---

## 8. pg_basebackup 실패 — 데이터 디렉토리 내 외부 파일

### 증상

```
pg_basebackup: error: could not get COPY data stream:
오류: "./pg_hba.conf.bak" 파일을 열 수 없음: 허가 거부
```

### 원인
pg_hba.conf를 수정하기 전 백업본(`pg_hba.conf.bak`)을 **데이터 디렉토리 내부에** 생성했다. pg_basebackup은 데이터 디렉토리 전체를 전송하므로 이 파일도 복사 대상이 되었으나, root 소유로 생성되어 postgres 사용자가 읽을 수 없어 전송이 중단되었다.

### 해결
백업 파일을 데이터 디렉토리 외부(`/var/lib/pgsql/`)로 이동한 뒤 재실행했다.

### 교훈
- 데이터 디렉토리에는 PostgreSQL이 관리하는 파일만 두어야 한다. 백업·임시 파일은 외부 경로에 생성한다.
- 관련하여 `rm -rf <dir>/*`는 glob이 숨김 파일을 포함하지 않아 디렉토리가 완전히 비워지지 않는다. `find <dir> -mindepth 1 -delete`를 사용해야 한다.

---

## 9. SSH 연결 실패의 계층 구분

### 증상
Ansible 실행 또는 ssh-copy-id 시 두 종류의 오류가 발생했다.

- `No route to host`
- `Permission denied (publickey,...,password)`

### 원인 및 구분
- **No route to host**: 대상 VM이 기동되지 않은 상태. 네트워크 계층에서 연결 자체가 성립하지 않는다.
- **Permission denied**: 연결은 성립했으나 인증에 실패. 계정 비밀번호가 설정되지 않았거나, 사용 중인 SSH 키가 대상에 배포되지 않은 경우다.

후자와 관련해, 컨트롤 노드의 작업 계정을 root에서 sysadmin으로 변경했을 때 인증이 실패한 사례가 있었다. SSH 키는 사용자별 홈 디렉토리(`~/.ssh`)에 저장되므로, root가 생성한 키는 sysadmin이 사용할 수 없다. sysadmin 계정으로 키를 새로 생성하고 재배포하여 해결했다.

### 교훈
- SSH 실패는 **연결 계층**과 **인증 계층**을 먼저 구분해야 한다. 오류 메시지가 그 구분을 직접 제공한다.
- 인증 자격(키)은 사용자 단위로 관리된다. 작업 계정이 바뀌면 자격 배포도 다시 수행해야 한다.

---

## 10. VM 클론 시 식별자 중복

### 예방 조치
골든 이미지를 클론하기 전 다음을 수행했다.

```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /etc/ssh/ssh_host_*
```

### 이유
- **machine-id 중복**: journald 로그 식별, DHCP 임대, 일부 모니터링 도구가 동일 호스트로 오인한다. 삭제가 아닌 0바이트 truncate를 사용하는 이유는 파일 자체가 존재해야 부팅 시 새 값이 생성되기 때문이다.
- **SSH 호스트 키 중복**: 모든 클론이 동일한 호스트 지문을 제시하게 되어 중간자 공격 탐지 기능이 무력화되고, known_hosts 관리가 불가능해진다. 삭제하면 sshd가 다음 기동 시 새로 생성한다.

추가로 VirtualBox 클론 시 MAC 주소 재생성 옵션을 적용해 L2 주소 충돌을 방지했다.

---

## 11. Vault 암호화 파일로 인한 무관한 플레이북 실행 실패

### 증상
백업 플레이북 실행 시 첫 task도 시작하지 못하고 중단:

```
ERROR! Attempting to decrypt but no vault secrets found
```

백업 롤은 암호화된 변수를 전혀 사용하지 않는데도 발생했다.

### 원인
대상 호스트 db2는 `[db]` 그룹에 속한다. Ansible은 실행 시 대상 호스트가 속한 그룹의 `group_vars/`를 **사용 여부와 무관하게 자동으로 로드**한다. 3주차에 복제 비밀번호를 담아 Vault로 암호화한 `group_vars/db.yml`이 로드되면서 복호화 키를 요구한 것이다.

### 해결
- 일회성: `ansible-playbook backup.yml --ask-vault-pass`
- 상시: Vault 비밀번호를 **리포지토리 밖** 파일에 두고 ansible.cfg에서 참조

```bash
echo '<vault-password>' > ~/.vault_pass
chmod 600 ~/.vault_pass
```
```ini
# ansible.cfg
vault_password_file = ~/.vault_pass
```

### 교훈
- group_vars는 선언적으로 자동 로드된다. 그룹 단위 암호화는 그 그룹을 대상으로 하는 모든 실행에 영향을 준다.
- 복호화 키 파일은 반드시 리포지토리 외부에 두고 권한을 600으로 제한한다. 암호화된 파일과 키를 같은 저장소에 두면 암호화의 의미가 사라진다.

---

## 12. ansible.cfg 중복 옵션으로 전체 명령 실행 불가

### 증상
모든 `ansible` 명령이 즉시 실패:

```
ERROR: Error reading config file (ansible.cfg): [line 6]:
option 'vault_password_file' in section 'defaults' already exists
```

### 원인
설정 존재 여부를 `grep`으로 확인해 이미 있음을 확인했음에도 `echo ... >> ansible.cfg`로 같은 줄을 한 번 더 추가했다. INI 형식 파서는 동일 섹션 내 중복 키를 허용하지 않아 설정 파일 전체를 읽지 못했다.

### 해결
줄 단위 삭제 대신 파일 전체를 heredoc으로 재작성했다 (1번 사례의 교훈 적용).

```bash
cat > ansible.cfg << 'EOF'
[defaults]
inventory = inventory/hosts.ini
remote_user = sysadmin
host_key_checking = False
vault_password_file = ~/.vault_pass
EOF
```

### 교훈
- `>>`(append)는 멱등하지 않다. 두 번 실행하면 두 줄이 된다. 설정 파일 변경은 확인 결과를 보고 실행 여부를 판단하거나, Ansible의 `lineinfile`처럼 멱등한 방식을 사용해야 한다.
- 설정 파일이 손상되면 도구 전체가 동작하지 않는다. 다행히 설정 파일을 읽지 못한 단계에서 중단되어, 뒤이어 입력한 파괴적 명령(DROP TABLE)은 실행되지 않았다.
- 참고: SSH 재접속 시 홈 디렉토리에서 시작하므로, 프로젝트 디렉토리로 이동하지 않고 실행하면 ansible.cfg를 찾지 못해 "hosts list is empty" 경고와 함께 대상 호스트를 인식하지 못한다(6번 사례와 동일 원인).

---

## 13. Loki 기동 실패 — Promtail과 gRPC 포트 충돌

### 증상
Loki 배포 후 서비스가 기동되지 않고 재시작을 반복하다 중단:

```
loki.service: Start request repeated too quickly.
Failed to start Loki log aggregation.
```

### 진단
`journalctl -u loki`에서 원인이 직접 드러났다.

```
level=error msg="error running loki"
err="listen tcp :9095: bind: address already in use
error initialising module: server"
```

### 원인
Loki와 Promtail은 모두 Grafana Labs 제품으로 **gRPC 기본 포트가 9095로 동일**하다. mon 노드에는 관측 스택(Loki)과 로그 에이전트(Promtail)가 함께 설치되는데, 먼저 기동된 Promtail이 9095를 점유한 상태에서 Loki가 같은 포트에 바인딩을 시도해 실패했다.

설정에서 `http_listen_port`만 지정하고 `grpc_listen_port`는 기본값에 맡긴 것이 원인이었다.

### 해결
두 컴포넌트의 gRPC 포트를 명시적으로 분리했다.

```yaml
# loki.yml — gRPC 포트를 9096으로 이동
server:
  http_listen_port: 3100
  grpc_listen_port: 9096
```
```yaml
# promtail.yml — gRPC 비활성화(0 = 임의 포트)
server:
  http_listen_port: 9080
  grpc_listen_port: 0
```

Promtail은 로그를 Loki로 전송하기만 하고 외부 요청을 수신하지 않으므로 gRPC 리스너가 불필요하다.

### 교훈
- 동일 호스트에 같은 계열의 컴포넌트를 배치할 때는 기본 포트 충돌을 사전에 확인해야 한다. 문서에 명시된 포트(HTTP)만 보고 부가 포트(gRPC, 메트릭 등)를 놓치기 쉽다.
- 기동 실패는 `systemctl status`의 요약보다 `journalctl -u <unit>`의 원문 로그에서 원인이 명확히 드러난다. `address already in use` 한 줄로 즉시 특정할 수 있었다.
- 기본값에 의존하지 않고 주요 포트를 설정에 명시하면 이런 충돌을 예방할 수 있다.

---

## 14. Grafana 데이터소스 프로비저닝 미반영

### 증상
Loki 설치 후 Grafana의 데이터소스 목록에 Prometheus만 표시되고 Loki가 나타나지 않았다.

### 진단
처음에는 Grafana가 프로비저닝 파일을 읽지 않은 것으로 판단해 서비스를 재시작했으나 변화가 없었다. 이후 **대상 노드의 실제 파일**을 확인했다.

```bash
ansible mon -m shell -a "grep -A3 'name: Loki' /etc/grafana/provisioning/datasources/prometheus.yml" -b
# rc=1 (매칭 없음)
```

파일 자체에 Loki 항목이 없었다. 즉 Grafana의 문제가 아니라 템플릿이 배포되지 않은 상태였다.

### 원인
데이터소스 템플릿에 Loki를 추가한 변경이 대상 노드에 반영되기 전이었다. Grafana 재시작은 이미 존재하는 파일을 다시 읽게 할 뿐, 없는 내용을 만들어내지 못한다.

### 해결
플레이북을 재실행해 템플릿을 배포하자 handler가 Grafana를 재시작했고, 데이터소스 목록에 Loki가 표시되었다.

### 교훈
- "설정이 적용되지 않았다"고 판단하기 전에 **대상 노드의 실제 파일 상태를 먼저 확인**해야 한다. 컨트롤 노드의 템플릿과 대상 노드의 배포 결과는 별개다.
- 서비스 재시작은 만능 조치가 아니다. 이 경우 재시작은 원인과 무관했고, 파일 확인이 문제를 즉시 특정했다.

---

## 15. Loki에 특정 서비스 라벨이 나타나지 않음 (장애 아님)

### 증상
Grafana Explore에서 `unit` 라벨 목록에 `crond.service`, `sshd.service` 등은 보이는데 `haproxy.service`가 없었다.

### 원인
장애가 아니라 정상 동작이다. Promtail 설정의 `max_age: 12h`에 따라 최근 12시간 범위의 journal만 수집하는데, HAProxy는 해당 기간 동안 journal에 기록을 남기지 않았다. 로그가 없으면 라벨도 생성되지 않는다.

### 확인
HAProxy 서비스를 재시작해 journal에 기록을 발생시키자 `haproxy.service` 라벨이 즉시 나타났다.

### 교훈
- **메트릭과 로그의 성질 차이**: 메트릭은 값이 0이어도 시계열이 계속 생성되지만, 로그는 이벤트가 발생해야만 존재한다. 로그 기반 관측에서 "보이지 않음"은 장애가 아니라 이벤트 부재일 수 있다.
- 수집 범위 설정(`max_age`)이 관측 가능한 대상을 제한한다는 점을 인지해야 한다.
