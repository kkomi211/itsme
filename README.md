# Synology Surveillance Station Web API 분석 보고서

본 문서는 Synology Surveillance Station Web API(v2.1)의 핵심 구조와 연동 절차를 개발자 관점에서 분석하여 정리한 가이드입니다.

---

## 1. 시스템 구조 및 기본 원칙
Synology API는 **WebAPI** 프레임워크를 기반으로 하며, 모든 요청은 HTTP/HTTPS GET 또는 POST를 통해 이루어집니다.

- **Base URL**: `http://{Synology_IP}:{Port}/webapi/`
- **데이터 포맷**: JSON (UTF-8)
- **핵심 파라미터**:
  - `api`: 호출할 API의 고유 이름
  - `method`: 실행할 동작
  - `version`: API 버전
  - `_sid`: 로그인 후 발급받는 세션 ID

---

## 2. API 연동 워크플로우 (Flow)



### 단계 1: API 정보 조회 (`SYNO.API.Info`)
서버가 지원하는 API의 버전과 실제 접근 경로(CGI)를 확인합니다.
- **Path**: `query.cgi`
- **Method**: `query`
- **주요 응답**: 각 API명에 대응하는 `path`, `minVersion`, `maxVersion`

### 단계 2: 인증 및 로그인 (`SYNO.API.Auth`)
API 제어를 위한 세션 권한을 획득합니다.
- **Method**: `login`
- **파라미터**: `account`, `passwd`, `session=SurveillanceStation`, `format=sid`
- **결과**: 응답 데이터의 `sid`를 이후 모든 요청 파라미터에 `_sid={sid}` 형태로 포함해야 합니다.

---

## 3. 주요 모듈별 상세 분석

### 3.1 카메라 관리 (Camera)
카메라 장치의 상태 조회 및 제어를 담당합니다.
- **API**: `SYNO.SurveillanceStation.Camera`
- **주요 Method**:
  - `List`: 등록된 모든 카메라의 ID, 이름, IP, 상태 정보 획득
  - `GetSnapshot`: 특정 카메라의 실시간 정지 영상(JPEG) 획득
  - `Enable/Disable`: 카메라 소프트웨어 전원 제어

### 3.2 라이브 뷰 및 스트리밍 (Live View)
실시간 영상을 외부 플레이어(VLC 등)에서 재생하기 위한 경로를 생성합니다.
- **API**: `SYNO.SurveillanceStation.Streaming`
- **주요 Method**:
  - `GetLiveViewPath`: RTSP, HTTP, RTSP over HTTP 등 프로토콜별 스트리밍 주소 반환

### 3.3 녹화물 관리 (Recording)
저장된 데이터를 검색하고 재생/다운로드합니다.
- **API**: `SYNO.SurveillanceStation.Recording`
- **주요 Method**:
  - `List`: 시간/이벤트별 녹화 파일 목록 필터링
  - `Download`: .mp4 또는 .ts 파일 다운로드
  - `Export`: 여러 카메라의 녹화물을 하나의 아카이브로 내보내기

### 3.4 PTZ 제어 (Pan/Tilt/Zoom)
물리적인 카메라 움직임을 제어합니다.
- **API**: `SYNO.SurveillanceStation.PTZ`
- **주요 Method**:
  - `Move`: 상하좌우 이동 및 속도 조절
  - `Zoom`: 확대/축소
  - `GoPreset`: 설정된 프리셋 위치로 즉시 이동

---

## 4. 공통 에러 코드 가이드

| 코드 | 의미 | 원인 및 해결방안 |
| :--- | :--- | :--- |
| **101** | Invalid Parameter | 필수 인자 누락 또는 데이터 타입 오류 |
| **105** | Insufficient Privilege | 해당 계정에 카메라 보기/관리 권한 없음 |
| **106** | Session Timeout | 세션 만료. `SYNO.API.Auth`를 통한 재로그인 필요 |
| **401** | Parameter Error | Surveillance Station 내부 파라미터 값 오류 |
| **402** | Camera Disabled | 비활성화된 카메라에 명령을 내릴 때 발생 |

---

## 5. 구현 시 주의사항 (Best Practices)
1. **버전 하드코딩 금지**: 반드시 `SYNO.API.Info`를 먼저 호출하여 서버가 지원하는 `maxVersion`을 동적으로 할당하세요.
2. **세션 유지**: 매번 로그인하기보다 `sid`를 유지하다가 `106` 에러가 발생했을 때만 재로그인하는 방식이 서버 부하가 적습니다.
3. **HTTPS 권장**: 패스워드와 영상 데이터 보호를 위해 5001번(기본) 포트의 HTTPS 사용을 강력히 권장합니다.
