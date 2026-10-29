# Shadow API 모니터링 — ELK 구축 진행 상황 (인수인계)

> 다른 컴퓨터의 Claude에게: 이 파일을 읽고 이어서 진행해 줘. (마지막 갱신 2026-10-29)

## 목표
- Shadow/Zombie API 탐지 + 신규 대시보드 → Prometheus가 아닌 ELK(Filebeat → Elasticsearch → Kibana) 로그 기반으로 결정
- 탐지 원리: 관측 엔드포인트 − 명세(OpenAPI) = Shadow, 명세상 deprecated ∩ 관측 = Zombie

## 문서·레포
- 팀 레포: https://github.com/sesac-nobreak-team-one/team_one — 브랜치 `api-monitoring`, PR #12 (main 대상, 머지 전), 경로 `docs/monitoring/`
- 가이드: `docs/monitoring/docs/02_ELK_구축_가이드라인.md` — A. FortiGate / B. pfSense / C. 현버전 우회(Wazuh archives 경유)
- 개인 레포 `zeldaim/shadow_api_monitoring`: 이전 버전, 손대지 않기로 함
- 사용자 요청: 커밋·PR에 Claude 공동작성 표기 넣지 않기, 본인(YHC) 이름으로 커밋

## 인프라 실측 (팀 문서 기준)
- FortiGate: WAN 192.168.0.141 / LAN port2 192.168.10.1 / DMZ port3 192.168.20.1 — 평가판 라이선스 정책 수 한도로 정책 추가 불가, pfSense로 교체 진행 중
- WEB 192.168.20.10 (DMZ, Ubuntu, 2GB): Docker Apache+PHP 쇼핑몰
  - 액세스 로그 호스트 경로 `/home/ubuntu/sesac_shop_web_php/logs/apache/shop_access.log` (combined)
  - SSH: 192.168.0.141:2222
- Wazuh 192.168.10.30 (LAN): 4.14.9 올인원, archives(logall_json) 활성화
  - `/var/ossec/logs/archives/archives.json`에 WEB 액세스 로그 이미 수집 중
  - 기존 Wazuh용 Filebeat 있음 → 건드리지 않기
- DMZ·LAN 인터넷 불가(폐쇄망). LAN/DMZ는 인프라 PC의 VMware Host-only(VMnet1/VMnet2)
- 노트북 → `ssh -p 8822 dongyoung@192.168.0.141` 포트포워딩 있음. Wazuh 서버로 가는지 확인 필요 (`hostname`, `ip a`)

## 현재 진행 상황
- ELK VM은 **사용자 노트북**에 생성 (Wazuh는 다른 PC) → Wazuh LAN에 직접 붙일 수 없음
- 연결 방식: **SSH 역방향 터널** (방화벽 정책 추가 불필요)
  - ELK VM에서 `ssh -p 8822 -N -R 19200:localhost:9200 dongyoung@192.168.0.141`
  - Wazuh의 `/opt/filebeat-elk`가 `https://localhost:19200`으로 전송 (9200은 Wazuh indexer가 사용 중)
- ELK VM: Ubuntu 26.04.1, amd64 (Elastic 공식 지원 OS 목록에는 아직 없음 — 인지하고 진행)
- 네트워크: 노트북 VMware NAT, IP 192.168.100.137 (DHCP라 바뀔 수 있음)
- OpenSSH 서버 설치 완료
- 버전: Elastic **9.5.5** (2026-10-06 GA) — ES·Kibana·Filebeat 동일 버전

## 다음 단계 (여기서부터 이어서)
1. VM에서 다운로드·검증 (+ `.sha512`, `sha512sum -c`)
   - `elasticsearch-9.5.5-amd64.deb`
   - `kibana-9.5.5-amd64.deb`
   - `filebeat-9.5.5-linux-x86_64.tar.gz`
2. `sudo dpkg -i elasticsearch-9.5.5-amd64.deb` → elastic 초기 비밀번호 메모 (레포에 올리지 않기)
3. `/etc/elasticsearch/elasticsearch.yml`
   ```yaml
   cluster.name: api-monitoring
   node.name: elk-01
   network.host: ["_local_", "_site_"]
   ```
4. heap: VM RAM 절반 → `/etc/elasticsearch/jvm.options.d/heap.options`
5. `sudo systemctl enable --now elasticsearch` → `sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic https://localhost:9200`
6. Kibana 설치 (`server.host: "0.0.0.0"`, enrollment token) → 노트북 브라우저 `http://<VM IP>:5601`
7. Dev Tools: API 키 → api-inventory → enrich 정책 → api-access 파이프라인 → 시뮬레이션 (가이드 4~5장)
8. ELK VM에서 `ssh-keygen -t ed25519` + `ssh-copy-id -p 8822 dongyoung@192.168.0.141` → 역방향 터널 상시화
9. filebeat tar.gz를 Wazuh로 옮겨 `/opt/filebeat-elk` 설치, hosts를 `https://localhost:19200`으로 (가이드 6-C)
10. Data View `filebeat-*` + 대시보드 (가이드 7장)

## 미확인
- 8822가 Wazuh 서버로 가는지
- Apache 로그 클라이언트 IP가 실제 IP인지 Docker 브리지 IP인지
- 쇼핑몰 API OpenAPI 명세 유무 (없으면 api-inventory 수동 작성)
