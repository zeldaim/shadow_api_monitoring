# SBOM (소프트웨어 부품 목록)

> Shadow API 탐지와는 별개 트랙이다. SBOM은 "어떤 라이브러리를 쓰는가"를 알려 줄 뿐, "어떤 API가 열려 있는가"는 알려 주지 않는다.

## SBOM이란

- 소프트웨어에 들어간 부품(라이브러리·패키지)과 버전의 목록. 식품의 원재료 표시와 비슷하다.
- 형식: CycloneDX 또는 SPDX (JSON)
- 용도
  - 취약점 대응: Log4Shell 같은 CVE가 나왔을 때 "우리 중 어디가 영향받나"를 바로 확인
  - 라이선스 관리, 공급망 보안·규제 대응
- 도구: `syft`(SBOM 생성), `grype`(SBOM으로 취약점 검사), Dependency-Track(SBOM 모아 보기, 선택)

## 팀 레포 대상

| 재료 | 위치 | SBOM에 잡히나 |
|---|---|---|
| PHP 라이브러리 | `web/src/composer.lock` | ✅ (`firebase/php-jwt v6.10.0`) |
| 웹 이미지 | `web/src/Dockerfile` (`php:7.4-apache`) | ✅ (빌드 후 OS 패키지까지) |
| ML 의존성 | `ml/requirements.txt` | ❌ 버전 범위(`>=`)만 있음 → 고정 파일(`pip freeze`) 필요 |

## 로컬에서 해 보기

```bash
# 설치 (Linux/WSL)
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

syft dir:web/src -o cyclonedx-json > sbom.cdx.json   # 부품 목록 생성
grype sbom:sbom.cdx.json                             # 취약점 검사
```

Docker만 있을 때 (Windows):
```powershell
docker run --rm -v "D:/team_one:/repo:ro" anchore/syft:v1.18.1 dir:/repo -o cyclonedx-json -q > sbom-source.cdx.json
docker build -t shop-web:sbomtest web/src
docker run --rm -v //var/run/docker.sock:/var/run/docker.sock anchore/syft:v1.18.1 docker:shop-web:sbomtest -o cyclonedx-json -q > sbom-image.cdx.json
```

## CI (GitHub Actions)

워크플로 파일: [`sbom.yml`](sbom.yml) → 팀 레포의 `.github/workflows/sbom.yml`에 넣는다.
(이 폴더에 있는 사본은 참고용이다. 이 레포에는 `web/src`가 없어서 여기서는 돌지 않는다.)

```
push / PR ─▶ GitHub Actions (GitHub 서버에서 실행, 실습 인프라와 무관)
              ├─ ① 소스 SBOM (syft)
              ├─ ② 이미지 빌드 → 이미지 SBOM (syft)
              ├─ ③ 취약점 검사 (grype, High 이상 기준)
              └─ ④ SBOM 파일 보관 (Artifacts, 90일)
```

| 언제 도나 | |
|---|---|
| main 대상 PR 생성·커밋 추가 | ✅ |
| main 머지 | ✅ |
| Actions 화면 **Run workflow** | ✅ 수동 |
| 다른 브랜치에 푸시만 (PR 없음) | ❌ |

- 결과: PR 화면의 **sbom** 체크 → 단계별 기록에서 취약점 표, 실행 화면 **Artifacts**에서 SBOM 파일
- 지금은 **점검만 하고 막지 않는다** (`fail-build: false`). 현황 파악 후 `true`로 바꾸면 High 이상 취약점이 있는 PR 머지를 막는다.
- 이미지 빌드가 실패해도(지원 종료 베이스 이미지의 apt 문제 등) 소스 SBOM은 남도록 이미지 단계만 건너뛴다.

## 로컬 확인 결과 (2026-10-30)

| 확인 | 결과 |
|---|---|
| actionlint (워크플로 문법) | 오류 없음 |
| 웹 이미지 빌드 | 성공 |
| 소스 SBOM | `firebase/php-jwt v6.10.0` + 워크플로가 쓰는 Actions 4개 |
| 이미지 SBOM | 부품 191개 (deb 187) |
| 이미지 취약점 | Critical 56 / High 233 / Medium 208 / Low 35 / Negligible 706 |

대부분 베이스 이미지 `php:7.4-apache`(지원 종료)에서 나온다. 줄이려면 PHP 8.x 베이스로 올려야 하고, 웹 코드 호환성 확인이 필요한 별도 작업이다.
숫자는 취약점 DB 갱신에 따라 달라진다.

## 팀 레포에 올리는 법

팀 규칙상 분류는 `infra` (`ci` 분류는 없음).

```powershell
cd D:\team_one
git switch -c infra/sbom-workflow origin/main
# .github/workflows/sbom.yml 넣기
git add .github/workflows/sbom.yml
git commit -m "infra: SBOM 생성 및 취약점 검사 워크플로 추가"
git push -u origin infra/sbom-workflow
```
푸시만으로는 PR이 생기지 않는다. GitHub 웹의 **Compare & pull request** 또는 `gh pr create --base main --web`으로 PR을 만든다.

## 토폴로지에 그릴 때

- 방화벽 바깥(WAN 위)에 **"외부 (GitHub)"** 영역을 따로 그린다. GitHub 서버에서 돌기 때문에 DMZ·LAN 안에 넣으면 틀린 그림이다.
- 팀원 PC → 팀 레포: 실선 "push"
- 팀원 PC → WEB 서버: **점선** "수동 반입" (폐쇄망이라 자동 배포가 아님)
- 이름은 "CI/CD"보다 **"CI (SBOM 점검)"** — 배포는 하지 않는다.
