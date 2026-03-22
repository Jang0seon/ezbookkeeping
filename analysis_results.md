# ezBookkeeping 프로젝트 분석 보고서

이 보고서는 `ezBookkeeping` 프로젝트를 포크하여 사용자 환경에 맞게 수정하고 배포하기 위해 필요한 기술 스택, 프로젝트 구조, 설정, 빌드 및 실행 방법을 정리한 문서입니다.

## 1. 기술 스택 (Tech Stack)

이 프로젝트는 백엔드(Go)와 프론트엔드(Vue.js)가 완전히 분리되어 작성된 풀스택 어플리케이션입니다.

### 백엔드 (Backend)
- **언어**: Go (버전 1.25 이상 권장, Docker 이미지에서는 1.25.7 사용)
- **웹 프레임워크**: [Gin](https://github.com/gin-gonic/gin) (`github.com/gin-gonic/gin`)
- **ORM / 데이터베이스 연동**: [XORM](https://xorm.io/) (`xorm.io/xorm`)
- **지원 데이터베이스**: SQLite3 (기본값), MySQL, PostgreSQL
- **주요 기능 라이브러리**:
  - `golang-jwt`: JWT 기반 인증
  - `go-co-op/gocron`: 스케줄링 및 배치 작업
  - `coreos/go-oidc`: OAuth 2.0 / OIDC 인증 연동
  - `minio-go/v7`: MinIO 등의 S3 호환 Object Storage 연동 기능

### 프론트엔드 (Frontend)
- **언어 및 툴**: TypeScript, Node.js (버전 24 권장), [Vite](https://vitejs.dev/) 빌드 도구
- **코어 프레임워크**: [Vue 3](https://vuejs.org/)
- **UI 프레임워크 (반응형 다형성)**:
  - 데스크톱 UI: [Vuetify 3](https://vuetifyjs.com/)
  - 모바일 UI: [Framework7](https://framework7.io/)
- **상태 관리**: Pinia
- **라우팅**: Vue Router

## 2. 프로젝트 디렉터리 구조

프로젝트는 모노레포(Monorepo) 형태로 백엔드와 프론트엔드 코드가 함께 존재합니다.

- **`/cmd`**: Go 백엔드 실행 포인트 (메인 함수)
- **`/pkg`**: Go 백엔드 비즈니스 로직 
  - `api`: HTTP API 핸들러
  - `models`: 데이터베이스 모델 객체
  - `services`: 핵심 비즈니스 로직
  - `storage`: 로컬/MinIO/WebDAV 등 저장소 스토리지 연동
  - `auth`: 내부 인증 및 외부 OAuth 로직
- **`/src`**: 프론트엔드(Vue.js) 소스 코드
  - 모바일(`mobile-main.ts`, `mobile.html`)과 데스크톱(`desktop-main.ts`, `desktop.html`) 환경을 나누어 빌드 엔트리가 존재
- **`/conf`**: 설정 파일 디렉터리 (`ezbookkeeping.ini`)
- **`/docker`**: Docker 빌드 시 사용하는 부가적인 스크립트
- **`build.sh` / `build.bat` / `build.ps1`**: 환경별 자동 빌드 스크립트
- **`Dockerfile`**: 백엔드와 프론트엔드를 Multi-stage 방식으로 빌드하여 단일 컨테이너로 패키징하는 설정 파일

## 3. 핵심 설정 파일 (`conf/ezbookkeeping.ini`)

수정해서 사용하실 때 가장 먼저 살펴봐야 할 파일이 `conf/ezbookkeeping.ini` 입니다. 이 파일을 통해 어플리케이션의 핵심 설정을 변경할 수 있습니다.

- **`[server]`**: HTTP 포트(기본 8080), 서버 도메인(`root_url`), HTTPS 인증서 설정 등
- **`[database]`**: 데이터베이스 종류(`sqlite3`, `mysql`, `postgres`) 및 접속 정보 등
- **`[storage]`**: 사용자가 업로드할 이미지/영수증 저장 경로 (로컬 파일 시스템 `local_filesystem`이나 S3 호환 `minio` 등 설정 지원)
- **`[security]`**: 보안과 관련된 설정 (기본적으로 `secret_key`는 고유하게 생성하거나 변경해야 함), JWT 리프레시 타임 등
- **`[auth]`**: 회원가입 개방 여부(`enable_register`), 이메일 인증, OAuth2(구글, OIDC, Gitea 등) 연동 등

> **팁**: 개인 서버에 띄울 경우 `[database]`에서 SQLite 대신 MySQL/PostgreSQL 셋팅을 권장하며, 보안을 위해 웹 접속에 사용되는 호스트와 HTTPS (Reverse Proxy 등) 세팅을 하는 것이 좋습니다.

## 4. 빌드 및 실행 방법

### 소스 코드 빌드 (수정 후 로컬 테스트 시)

개발 및 테스트를 위해 수동 빌드할 경우 아래 과정이 필요합니다. Go와 Node.js 환경이 시스템에 설치되어 있어야 합니다.

1. **프론트엔드 빌드**:
   `npm install` 후 `npm run build`를 실행하거나 제공된 스크립트 `./build.sh frontend` 실행
2. **백엔드 빌드**:
   `go build -o ezbookkeeping ezbookkeeping.go` 혹은 `./build.sh backend` 실행
3. **실행**:
   빌드된 바이너리인 `./ezbookkeeping server run` 실행

### Docker를 이용한 빌드 및 배포 (운영 환경 추천)

원하는 대로 소스와 설정을 커스터마이징한 뒤 Docker 컨테이너로 자체 호스팅하기에 매우 적합하게 되어 있습니다.

1. 수정한 소스코드 최상위 경로에서 다음 명령어 실행:
   ```bash
   # build.sh의 헬퍼 스크립트를 사용하여 docker 이미지를 쉽게 생성 (태그 지정 가능)
   ./build.sh docker -t my-ezbookkeeping:latest
   ```
2. 빌드된 이미지를 실행:
   ```bash
   docker run -d -p 8080:8080 -v /내/컴퓨터/경로:/ezbookkeeping/data my-ezbookkeeping:latest
   ```
*(실제 사용할 때는 `docker-compose.yml`을 만들어서 데이터 디렉터리 마운트와 리버스 프록시 연동을 하는 것이 관리가 수월합니다.)*

## 5. 커스터마이징을 위한 시작점 가이드

- **UI/디자인 변경**: `/src/styles` 안의 변수들이나 Vue 컴포넌트(`src/views`, `src/components`)를 직접 수정합니다.
- **백엔드 API 변경/데이터 추가**: `/pkg/models`에 테이블 모델을 추가/수정하고, `/pkg/services`에 로직을 구현한 뒤 `/pkg/api`에 새 라우트 핸들러를 연결합니다.
- **다국어 (언어 추가/수정)**: `/src/locales` 디렉터리에 배포 환경 언어팩이 들어있습니다.

---
추가로 특정 기능(예: DB를 MariaDB로 연동, OAuth 로그인 연결, 화면에서 특정 항목 제거)을 어떻게 수정해야 할지 알고 싶으시다면 상세히 알려주세요.
