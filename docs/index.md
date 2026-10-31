<img alt="Magpie Slicer" src="assets/magpie-slicer.png" width="96">

# Magpie Slicer 모드 개발자 가이드

제3자 개발자가 Magpie Slicer(이하 "Magpie")용 모드를 만들고, 빌드하고, 붙이고, 배포하는 데 필요한 내용을 한 곳에 모았습니다.

| 항목 | 내용 |
|---|---|
| 대상 버전 | Magpie Slicer 다음 릴리스, 모드 API **1.1** |
| 이 저장소 | `docs/` 이 가이드, `sdk/` 헤더와 CMake 템플릿(앱 소스의 `mods/sdk/`), `mods/<id>/` 번들 공식 모드 10개의 소스 (예제로 보세요). Anycubic 모드 소스는 들어 있지 않습니다 |
| 번들 공식 모드 | 10개. Anycubic 프린터 모드는 앱에 들어가지 않고 별도 패키지로 배포됩니다 (12장) |
| 문서 성격 | 코드를 읽고 쓴 문서입니다. 코드에서 직접 확인하지 못하고 추론한 내용은 **(추론)** 으로 표시했습니다. 본문의 `src/...`, `docs/HLSD/...` 경로는 Magpie Slicer 앱 소스 기준입니다 |

---

## 0. 신뢰 등급: 인증(파란색) 모드와 인증 안 된(빨간색) 모드

### 0.1 등급

| 등급 | 무엇으로 정해지나 | 모드 관리자 표시 | 로드 | 권한 |
|---|---|---|---|---|
| 공식 | 앱에 번들된 모드, 또는 `magpie-official-2026` 키 서명 | 파란색 체크 배지, "기본 포함" 또는 "공식" | 바로 로드 | 모든 기능 |
| **인증 (verified)** | Gen이 모드를 직접 검토한 뒤 Magpie 인증 키 `magpie-verified-2026`로 서명 | **파란색 체크 배지, "인증됨"** | 바로 로드 | 모든 기능 |
| **인증 안 됨** | 서명 없음, 모르는 키로 서명, 또는 서명 후 파일이 바뀜 | **빨간 느낌표 배지, "인증 안 됨"** | 네이티브 코드는 처음 한 번 경고 대화상자에서 사용자가 허락하면 로드. JSON 전용 모드는 바로 로드 | 모든 기능 |

- 배지 색: 인증·공식은 **Magpie 파란색(#398FF4) 체크 배지**, 인증 안 됨은 **빨간 느낌표 배지**입니다(`ModManagerDialog.cpp`).

- **권한은 세 등급이 같습니다.** G-code 설정 규칙(`machine_start_gcode` 등 `*_gcode`, `post_process`)과 `gcode.events`의 시작 G-code 필터도 모든 모드에 열립니다 (Gen 결정 2026-10-22, `ModSet.cpp`).
- 등급의 차이는 **표시와 경고뿐**입니다. 인증은 "Gen이 이 버전의 이 파일들을 직접 보고 서명했다"는 뜻이고, 코드가 안전하다는 기술적 보장은 아닙니다.
- 서명은 패키지 폴더의 모든 파일을 덮습니다 (10장). 데이터 폴더에 설치된 모드는 서명 후 파일이 하나라도 바뀌면 "인증 안 됨"이 됩니다. 앱 설치 폴더의 번들 모드는 서명이 있는데 맞지 않으면 아예 로드되지 않습니다(설치본 변조 방지).
- 앱에 들어 있는 공개키 (참고용, Ed25519, 16진수 64자):
  - `magpie-official-2026`: `2b7bb01a93e079a6e2c4d6d7a41fd71dc7fb2eb24e57900a9e96a0767426f1f0`
  - `magpie-verified-2026`: `3a129f7dfe5475ae2e1092d1e3487d2dbe9f8a7be988274e4323464fdffe27c8`

  개발자가 이 값을 쓸 일은 없습니다. 받은 `SIGNATURE`의 `"key"`가 이 이름인지, 그리고 서명이 이 공개키로 검증되는지 직접 확인하고 싶을 때만 씁니다.

### 0.2 인증 안 된 모드의 경고 (한 번)

인증되지 않은 네이티브 모드는 앱 시작 때, 라이브러리를 로드하기 **전에** 아래 대화상자를 띄웁니다.

- **언제 다시 묻나:** 모드 id + 버전 + `bin/<플랫폼>/` 폴더의 모든 파일 해시마다 한 번입니다. 허락은 `%APPDATA%\MagpieSlicer\mods\mods_state.json`의 `accepted_unverified`에 기억되고, 모드를 업데이트하거나 그 폴더의 파일이 바뀌거나 추가되면 다시 묻습니다. 로드되는 것은 허락받은 바로 그 파일들의 캐시 복사본입니다.
- **"로드하지 않음"에 포커스가 가 있고, 창을 닫아도 "로드하지 않음"입니다.** 이를 고르면 그 실행 동안 모드를 건너뛰고(그 모드에 의존하는 모드도 함께), 다음 시작 때 다시 묻습니다.
- **명령줄(CLI)** 에서는 물을 수 없으므로, GUI에서 한 번 허락한 모드만 로드됩니다. 허락받지 않은 모드는 `unverified native code not accepted yet (...); start the application and accept its warning to load it`로 남습니다. `--mod-test`도 마찬가지이니, 새 빌드를 시험하기 전에 앱을 한 번 열어 허락하세요.
- 한국어 문구 (`{...}`는 모드 정보로 채워짐):

> **인증되지 않은 모드**
>
> {모드 이름} {버전}({제작자})은 Magpie가 검토하거나 인증하지 않은 모드입니다.
> 이 모드는 프로그램 안에서 코드로 실행되며, 슬라이싱 결과와 프린터가 실행할 G-code를 바꿀 수 있습니다. 악성 코드나 바이러스가 들어 있어도 Magpie가 막아 주지 못합니다. 이 경우 PC의 파일과 정보가 위험해지거나, 잘못된 G-code로 프린터가 손상될 수 있습니다.
> 믿을 수 있는 곳에서 받은 모드만 사용하세요. 이 모드를 사용해서 생기는 문제의 책임은 사용자에게 있습니다.
>
> [위험을 이해하고 로드]  **[로드하지 않음]**

- 영어 원문 (앱 번역 카탈로그에 함께 들어감):

> **Unverified mod**
>
> {name} {version} ({author}) has not been reviewed or verified by Magpie.
> It runs as code inside the application and can change the slicing result and the G-code your printer runs. Magpie cannot stop malicious code or viruses in it; your files and data could be at risk, and wrong G-code could damage your printer.
> Only use mods from sources you trust. You are responsible for any problem this mod causes.
>
> [I understand the risk, load it]  **[Don't load]**

- `{제작자}`를 채우려면 `mod.json`에 새 선택 필드 `"author"`를 넣으세요 (4.1절). 없으면 "제작자 정보 없음"으로 표시됩니다.
- JSON 전용 모드는 코드가 없으므로 이 경고를 띄우지 않습니다 (태그는 "인증 안 됨").

### 0.3 인증받는 방법 (Gen 검토)

연락은 이 저장소의 Issues로 해 주세요.

1. 이 가이드대로 빌드하고 `--mod-test <id>`를 통과시킵니다 (9장). 꺼진 상태가 바이트 동일한지(6.3절)도 확인합니다.
2. **서명하지 않은** 패키지 폴더(`mod.json`, `bin/<플랫폼>/`, `i18n/`, `tests/`)와 **소스 코드 전체**, 빌드 방법을 Gen에게 보냅니다. 소스 없이 바이너리만으로는 검토할 수 없습니다.
3. Gen이 소스를 읽고, 같은 소스로 빌드되는지와 동작을 확인한 뒤 인증 키로 서명합니다:
   `--mod-sign <패키지 폴더> --mod-sign-key <인증 개인키> --mod-sign-key-id magpie-verified-2026`
4. 서명된 폴더(`SIGNATURE`가 생김)를 그대로 배포합니다. 이후 파일을 하나라도 바꾸면 다시 받아야 하고, 새 버전마다 다시 검토·서명합니다.

### 0.4 공식 모드로 편입하기 (소스로 제출)

자기 모드를 Magpie **공식 모드**(앱에 기본으로 들어가는 모드)로 넣고 싶다면:

- **DLL로 컴파일하지 말고 소스 상태로 Gen에게 주세요.** 컴파일된 `bin/<플랫폼>/*.dll`이나 서명된 패키지는 편입 제출로 받지 않습니다.
- 보낼 것:
  - `mods/<id>/` 형태의 소스 폴더: `CMakeLists.txt`, 소스 파일, `mod.json`, `i18n/`, `tests/`. 공식 모드와 같은 배치이고, `mods/half_layer`를 참고하세요.
  - 직접 쓰지 않은 코드가 있으면 그 출처와 라이선스.
  - 테스트 케이스와 기대 결과, 켠 상태와 끈 상태를 비교한 설명.
- 이후 과정: Gen이 소스를 검토하고, Magpie 저장소에서 공식 모드 빌드(`magpie_official_mod`)로 직접 컴파일합니다. 그 빌드가 ctest, `--mod-test`, 골든 비교(끈 상태 바이트 동일)를 통과하면 공식 키로 서명해 앱에 함께 넣습니다. 편입된 뒤에는 그 소스가 공식 저장소에서 관리되고, 버전도 앱과 함께 올라갑니다.
- 이유: 공식 모드는 모든 사용자에게 기본으로 실행되므로, Magpie가 읽은 소스와 앱에 들어가는 바이너리가 같아야 합니다. 같은 이유로 공식 모드는 SDK 템플릿과 같은 컴파일러·플래그로 빌드되어야 합니다.

인증(파란색 태그) 요청(0.3절)도 소스를 함께 보내야 하는 점은 같습니다. 차이는 인증 모드는 개발자가 따로 배포하고, 공식 모드는 앱에 들어간다는 것입니다.

---

## 1. 모드의 두 종류

| | JSON 모드 | 네이티브 모드 |
|---|---|---|
| 형태 | `<id>.json` 한 파일, 또는 `<id>/mod.json` 패키지 | `<id>/mod.json` + `bin/<플랫폼>/<바이너리>` 공유 라이브러리 |
| 할 수 있는 일 | 파라미터를 설정 탭에 추가하고, 그 값으로 **기존 설정**을 바꿈 (규칙) | 위에 더해, 코어가 제공하는 **슬롯**에 코드를 꽂아 서포트 생성, 벽 계획, 레이어 밴드, G-code 이벤트 등을 구현 |
| 코드 실행 | 없음 | 사용자 권한으로 같은 프로세스에서 실행 |
| 서명 | 불필요 (빨간 태그로 표시만 됨) | 선택. Gen 인증 서명이 있으면 파란색 태그, 없으면 빨간 태그와 처음 한 번 경고 (0장) |
| 적용 시점 | Get Mods로 받은 JSON 모드는 즉시 | 다음 앱 시작 때 (실행 중에는 모드 집합이 바뀌지 않음) |

어떤 모드든 **꺼져 있거나 기본값이면 Orca와 바이트 단위로 같은 G-code**가 나와야 합니다 (6.3절).

### 1.1 할 수 있는 것과 할 수 없는 것

모드는 **코어가 열어 둔 지점에서만** 동작합니다. 새 슬롯을 만들거나, 슬롯이 없는 단계(슬라이싱, 인필, 솔기, 이동 경로, 미리보기)를 바꾸는 방법은 없습니다. 아래 표의 "불가"는 현재 API로는 방법이 없다는 뜻이고, 필요하면 코어에 슬롯을 추가해야 합니다.

#### JSON 모드 (규칙)

| 할 수 있는 것 | 예 |
|---|---|
| G-code 설정 쓰기: `machine_start_gcode`, `layer_change_gcode` 등 `*_gcode`, `*_gcodes`, `post_process` | 높이별 일시정지(M600/M601), 레이어 변경 매크로, 후처리 스크립트. 모든 등급의 모드가 가능 (0장) |
| 프린트, 프린터, 필라멘트 설정 값을 파라미터에서 계산해 덮어쓰기 | 첫 레이어 속도 낮추기, 노즐 지름에 비례한 선폭, 재료별 온도 보정, 체크 하나로 여러 설정 묶어 켜기 |
| 조건부 적용 (`when`), 리스트 설정의 한 원소 (`index`) | "엔지니어링 재료면 챔버 온도 X", 2번 필라멘트 온도만 변경 |
| 설정 탭 Mods 페이지에 파라미터 추가 (프로세스, 프린터, 필라멘트, 객체, 영역 범위) | 드롭다운, 숫자, 체크박스, 퍼센트 |

| 할 수 없는 것 | 이유 / 근거 |
|---|---|
| 점·좌표 타입 설정 쓰기 (`printable_area`, 베드 모양 등), 열거형 리스트(`coEnums`) 쓰기 | 규칙이 쓸 수 있는 타입은 float, percent, float_or_percent, int, bool, string, enum과 그 리스트(enum 리스트 제외)뿐 (`ModSet.cpp`) |
| 사용자가 프리셋에서 직접 바꾼 값 덮어쓰기 | 규칙은 사용자 편집값 아래에 적용 |
| 기하, 레이어, 객체 모양에 따라 다른 값 | 규칙은 설정과 파라미터만 읽음. 객체·영역 범위 값도 규칙은 프로세스 값만 읽음 |
| 새 Orca 설정 키 추가 | 파라미터는 `mod_settings` 등에 JSON으로 저장되고 Orca 설정이 되지 않음 |

#### 네이티브 모드 (슬롯별)

| 슬롯 | 할 수 있는 것 (예) | 할 수 없는 것 / 한계 |
|---|---|---|
| `support.areas` | 객체 레이어마다 서포트 몸체·지붕·바닥 **영역**을 직접 계산 (Cura 방식 서포트, 계단식·원뿔형 서포트, 타워). CuraEngine식 분수 지붕 높이 (`fractional_*`) | 툴패스·패턴·래프트·인터페이스 간격은 Orca 일반 서포트가 만듦. 트리 서포트 형태 불가. 객체 하나에 서포트 구현 하나 |
| `support.compose` | 코어 생성기(Orca 일반, `support.areas` 모드, Orca 트리)를 마스크로 나눠 돌리고 일반 + 트리 두 세트를 합침 (Mixed 서포트) | 레이어를 직접 만들거나 수정 불가. 세트는 최대 2개(첫째 일반, 둘째 트리) |
| `support.contacts` | 서포트가 닿는 지점과 몸체를 직접 정함, 메시(삼각형) 접근 가능 (레진식 기둥 서포트) | 인터페이스·간격·래프트·툴패스는 Orca 일반 서포트. `support_type` 무시 |
| `support.style` | 모든 생성기의 인터페이스 패턴(직선, 격자, 삼각, 동심 4종), 일반·트리 벽 수 0-10, 인터페이스 레이어별 패턴·각도, (1.1) 그 레이어의 노즐 온도 | 다른 패턴 추가 불가. 객체 하나에 스타일 구현 하나. 온도는 펌웨어 냉간 압출 한계와 필라멘트 최대 온도로 검사됨 |
| `object.z_offset` | 서포트가 있는 객체를 띄워 바닥 밑에도 서포트 (레진식) | 서포트 없는 객체는 못 띄움. 객체 하나에 구현 하나 |
| `wall.plan` | 다중 노즐 프린터에서 바깥 벽 몇 줄을 가는 "디테일" 노즐로, 레이어 전체 벽을 다른 툴헤드로, 벽 태그별 필라멘트 선택 (다중 노즐 벽) | 벽 경로 자체(Classic/Arachne) 생성 불가, 벽 수·폭·겹침만 계획. 단일 노즐 프린터에서는 의미 없음 |
| `layer.bands` | 바깥 벽과 서포트를 모델 레이어 안에서 두 개의 얇은 띠로 인쇄, 나중 띠의 시작 시점 선택 (H/2) | **현재 실행기는 0.5 + 0.5 두 띠, 바깥 벽 2줄만** 인쇄. 다른 계획은 검증에서 거부. 프린트 전체가 같은 구현 |
| `gcode.events` | 레이어 시작·끝, 경로마다, 서포트 끝 등에서: 주석, 노즐 온도 변경·대기, 일시 대기(dwell), **보조** 팬, 코어 규칙의 이동, 자유 이동(노즐 브러시, 보정 표시), 코어 압출 코드로 추가 압출(느린 꼬리 포함), 경로별·점별 속도 변경, 보조 타워 배치, 서포트를 객체 뒤에 인쇄하도록 순서 요청, 프린트 검증 오류 (저온 서포트 접점, 온도 강하 타워, LESIC 보정, 오버행 속도 조절, 레이어마다 노즐 닦기) | **이벤트 지점에 임의 G-code 텍스트 불가** (`text`는 주석만). 레이어 중간의 M600, 매크로, 카메라 트리거는 이 슬롯으로 넣을 수 없음 (레이어 변경 G-code 같은 설정은 JSON 규칙으로 바꿀 수 있음). 주 부품 냉각 팬 직접 제어 불가(보조 팬만, 경로의 냉각은 코어 규칙). 기존 경로 삭제·이동 불가 |
| `gcode.time_plan` | 대기 지점마다 대기 시간, 파킹 여부, 타워 블록 생략 지점, 경고 종류 (레이어 시간 블렌딩) | 숫자만: G-code나 기하에 접근 불가. 대기 지점 위치는 코어가 정함 |
| `printer.agent` | Orca에 없는 프린터 연결: 상태, 재료 유닛, 파일 업로드, 출력 시작·일시정지·중지, 온도, 팬, 조명, 카메라 웹 패널 (애니큐빅 LAN). 클라우드 계정 로그인 페이지도 가능 (Anycubic Cloud, 12.4절) | 원시 G-code 전송 불가(중립 명령만). 매니페스트에 적고 사용자가 동의한 호스트·포트만. GUI 앱에서만 |

#### 모든 모드에 공통인 한계

| 할 수 없는 것 | 비고 |
|---|---|
| 새 슬롯, 새 서비스 종류를 코어에 추가 | 모드끼리의 서비스(`export_service`)는 가능 |
| 인필, 솔기, 이동 경로, 다리(bridge), 다림질, 프라임 타워 알고리즘 교체 | 해당 슬롯 없음 |
| 슬라이싱 결과·미리보기 렌더링 변경, 설정 탭의 자유로운 GUI 위젯 | GUI는 선언형만: 파라미터 행, 드롭다운, 서포트 메뉴 항목, 페인트 채널 2개, 객체 파라미터 순서, 프린터 연결 패널(필드, 버튼, 알림). 자유로운 화면은 `printer.agent` 모드의 웹 패널(모드 폴더 안 HTML) 하나뿐 |
| 보정 메뉴에 새 보정 추가 | 앱에 그 id의 대화상자가 있어야 하며 지금은 `lesic`뿐 |
| 실행 중 켜기·끄기·업데이트 | 다음 시작에 적용 |
| 공식 모드 교체 (같은 id) | 거부됨 |
| 코어 내부 C++ 객체 접근 | C ABI의 숫자, 폴리곤, 문자열, 핸들만 |
| 네트워크 (슬라이싱 슬롯에서) | `core.net`은 GUI 전용이고 선언한 끝점만 |
| 경고 없이 네이티브 모드 로드 | Gen 인증 서명이 없으면 처음 한 번 경고 (0장) |

---

## 2. 툴체인과 SDK

### 2.1 준비물

| 플랫폼 | 컴파일러 | 비고 |
|---|---|---|
| Windows x64 (주 대상) | Visual Studio 2022 (MSVC v143), CMake 3.16 이상 | 공식 모드는 MSVC 14.44로 빌드됨 |
| Windows arm64 | MSVC arm64 | 플랫폼 폴더 `win-arm64` |
| Linux x64/arm64 | GCC 또는 Clang | `linux-x64` / `linux-arm64`, 파일명 `lib<바이너리>.so` |
| macOS x64/arm64 | Apple Clang | `macos-x64` / `macos-arm64`, 파일명 `lib<바이너리>.dylib` |

- 언어: SDK 헤더는 **C**입니다. 모드 본체는 C++17로 써도 됩니다 (`magpie_mod.hpp`가 C++ 도우미). 경계를 넘는 것은 C 타입뿐입니다.
- 슬라이서 소스나 Orca 헤더는 **필요 없고, 쓰면 안 됩니다**. SDK 템플릿이 include 경로를 SDK로 제한합니다.
- 외부 라이브러리(Clipper2, Eigen 등)가 필요하면 **정적으로** 링크합니다. 코어의 Clipper와 같은 결과가 필요하면 `core.geometry` 서비스를 쓰세요 (5.10절).

### 2.2 SDK 구성 (`mods/sdk/`)

| 파일 | 내용 |
|---|---|
| `include/magpie_mod_api.h` | 기본 ABI: 상태 코드, 점/폴리곤 타입, 호스트 테이블 `MagpieHostApi`, 등록기 `MagpieRegistrar`, 세 개의 export 함수 |
| `include/magpie_mod.hpp` | C++ 도우미 (헤더 전용): `MAGPIE_MOD_DEFINE`, `magpie::guarded`, `param_*`, `offset`, 폴리곤 변환 |
| `include/magpie_geometry.h` | 서비스 `core.geometry` v1 (코어의 ClipperUtils) |
| `include/magpie_support.h` | 슬롯 `support.areas` v1 |
| `include/magpie_slot_support_compose.h` | 슬롯 `support.compose` v1 |
| `include/magpie_slot_support_contacts.h` | 슬롯 `support.contacts` v1 |
| `include/magpie_slot_support_style.h` | 슬롯 `support.style` v1 |
| `include/magpie_slot_object_z_offset.h` | 슬롯 `object.z_offset` v1 |
| `include/magpie_slot_wall_plan.h` | 슬롯 `wall.plan` v1 |
| `include/magpie_slot_layer_bands.h` | 슬롯 `layer.bands` v1 |
| `include/magpie_slot_gcode_events.h` | 슬롯 `gcode.events` v1 |
| `include/magpie_slot_time_plan.h` | 슬롯 `gcode.time_plan` v1 |
| `MagpieMod.cmake` | 빌드 템플릿 `magpie_add_mod()` |
| `include/magpie_slot_printer_agent.h`, `include/magpie_net.h` | 슬롯 `printer.agent` v1, 서비스 `core.net` v1 |

SDK는 별도 배포본이 아직 없습니다. 저장소의 `mods/sdk/` 폴더를 통째로 복사해 쓰면 됩니다 (슬라이서 소스에 의존하지 않습니다).

> 참고: 가속 슬롯은 없습니다. 예전 개발 버전의 `accel.batch` 슬롯(CUDA 가속)은 삭제되었습니다.

### 2.3 CMake 템플릿이 강제하는 것 (`MagpieMod.cmake`)

`magpie_add_mod(<target> SOURCES ... [BINARY <이름>] [OUTPUT_DIR <패키지 폴더>] [DEFINITIONS ...])`

- `MODULE` 라이브러리, 접두사 없음, C++17.
- **include 경로를 SDK의 `include/` 하나로 교체**합니다 (추가가 아니라 교체). 슬라이서 트리 안에서 빌드해도 코어 헤더가 보이지 않습니다.
- **부동소수점 모드 고정**: MSVC는 `/fp:precise`, GCC/Clang/clang-cl은 `-ffp-contract=off -fno-fast-math`와 `MAGPIE_MOD_FP_CONTRACT_OFF=1`. 라이브러리는 이 모드를 `magpie_mod_query`로 보고하고, 코어는 `precise`가 아니면 거부합니다 (FMA 축약이나 fast-math는 마지막 자릿수를 바꿔 G-code가 달라짐).
- 세 export 외의 심볼은 숨김.
- `OUTPUT_DIR`을 주면 결과물이 `<OUTPUT_DIR>/bin/<플랫폼>/`에 떨어집니다 (설치된 모드와 같은 배치).

---

## 3. 패키지 구조

```
<mods 폴더>/<id>/                       폴더 이름 = mod.json의 id (다르면 로드 실패)
    mod.json                           매니페스트 (4장)
    bin/win-x64/<binary>.dll           플랫폼별 라이브러리 (linux-x64/lib<binary>.so, macos-arm64/lib<binary>.dylib ...)
    i18n/<lang>.json                   선택: 코드가 내보내는 문구의 번역 (7장)
    tests/                             선택: --mod-test 케이스 (9장)
    web/...                            선택: printer.agent 웹 패널 페이지
    SIGNATURE                          Ed25519 서명 (10장). 폴더의 다른 모든 파일을 덮음
```

모드를 찾는 곳 (`load_mods`, `ModPackage.cpp`):

| 위치 | Windows 예 | 취급 |
|---|---|---|
| `<resources>/mods` | `<앱 설치 폴더>\resources\mods` | 번들: 앱과 함께 설치되는 공식 모드. 제3자 모드를 넣는 곳이 아닙니다 |
| `<data dir>/mods` | `%APPDATA%\MagpieSlicer\mods` | 설치: 신뢰된 키 서명이 있으면 바로, 없으면 네이티브 코드는 한 번 경고 후 로드 (0장) |

- 각 폴더에서 `*.json`과 `<id>/mod.json`을 읽습니다. 바이너리가 있는 모드는 반드시 패키지 폴더여야 합니다.
- 설치 모드의 `bin/<플랫폼>/` 폴더는 `<data dir>/cache/mods/<id>-<version>-<해시16>/`로 복사되고, 복사본이 서명된 파일(인증 안 된 모드는 사용자가 허락한 파일)과 같은지 확인한 뒤 **복사본이 로드**됩니다. 원본은 실행 중에도 교체할 수 있고, 확인한 뒤 바꿔치기된 파일은 로드되지 않습니다.
- 번들 모드와 같은 id의 설치 모드는 거부됩니다 (`a bundled mod has this id and cannot be replaced`). 공식 모드 id를 피하세요: `cura_support`, `half_layer`, `layer_time_blend`, `low_temp_calibration`, `low_temp_interface`, `mixed_support`, `multi_nozzle_walls`, `resin_support`, `snapmaker_u1`, `support_extras`, 그리고 별도 배포되는 `anycubic`.
- Windows 로드는 `LoadLibraryExW(LOAD_LIBRARY_SEARCH_DLL_LOAD_DIR | LOAD_LIBRARY_SEARCH_SYSTEM32)`입니다. **의존 DLL은 모드의 `bin/<플랫폼>/` 폴더와 System32에서만** 찾습니다. 앱 폴더의 DLL은 검색 경로가 아닙니다 (3.1절 참고).

### 3.1 MSVC 런타임

공식 모드는 `/MD`(동적 런타임)로 빌드되어 `MSVCP140.dll`, `VCRUNTIME140.dll`에 의존합니다 (`half_layer.dll`의 dumpbin 결과). 이 DLL들은 앱 프로세스에 이미 로드되어 있어 이름으로 해결되는 것으로 보입니다 **(추론, 확인하지 않음)**. 제3자 모드는 확실하게 하려면 둘 중 하나를 권합니다.

- 정적 런타임: `set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")` (`/MT`). 메모리 소유권 규칙(5.3절)을 지키면 힙이 달라도 문제없습니다.
- 또는 필요한 런타임 DLL을 `bin/win-x64/`에 함께 넣고 서명에 포함.

---

## 4. 매니페스트 (`mod.json`)

주석(`//`)을 허용하는 JSON입니다. 파서는 `ModSet::parse_mod` (`src/libslic3r/ModHost/ModSet.cpp`).

### 4.1 최상위 필드

| 필드 | 필수 | 형식 | 의미 |
|---|---|---|---|
| `id` | ✅ | `a-z`, `0-9`, `_`, 숫자로 시작하지 않음 | 모드 식별자. 패키지 폴더 이름과 같아야 함 |
| `version` | ✅ | 문자열 (`"1.2.0"`) | `magpie_mod_query`의 버전과 정확히 같아야 함. 비교는 점으로 나눈 숫자 비교 |
| `name`, `name_<lang>` | | 문자열 | 표시 이름 (`name_ko`) |
| `author` | | 문자열 | 제작자. 인증 안 된 모드의 경고문과 모드 목록에 쓰임 (0.2절) |
| `params` | | 객체 | 파라미터 (4.2절). **키 순서가 설정 탭의 행 순서** |
| `rules` | | 배열 | 설정 쓰기 규칙 (4.3절) |
| `api` | 바이너리 있으면 ✅ | `{"major": 1, "min_minor": 0}` | 빌드한 API. `major`는 코어와 같아야 하고 `min_minor`는 코어의 minor 이하 |
| `binary` | | `[A-Za-z0-9_][A-Za-z0-9_.-]*` | 확장자·접두사 없는 라이브러리 이름 |
| `slots` | | 배열 | 등록할 슬롯 구현 (4.4절). 있으면 `binary` 필수 |
| `depends` | | `[{"id", "min"}]` | 먼저 로드할 모드 (`"core"`는 모드 API 버전, 예: `{"id": "core", "min": "1.1"}`) |
| `conflicts` | | id 배열 | 같이 로드할 수 없는 모드 |
| `thread_safe` | | bool, 기본 `false` | `true`면 여러 스레드에서 동시에 호출됨 (5.5절) |
| `legacy_keys` | | 객체 | 옛 설정 키를 파라미터로 옮김 (4.6절) |
| `gui` | | 객체 | 선언형 GUI (4.5절) |
| `network` | | 객체 | `core.net`이 허용할 호스트·포트 (12장) |

모르는 최상위 필드는 무시됩니다.

### 4.2 파라미터 (`params`)

```json
"params": {
    "enable":  { "type": "bool",  "default": false, "label": "Use my feature", "label_ko": "내 기능 사용",
                 "invalidates": ["posSupportMaterial"] },
    "gap":     { "type": "float", "default": 0.2, "unit": "mm", "scope": "object",
                 "invalidates": ["posSupportMaterial"], "label": "Gap", "label_ko": "간격",
                 "visible": "my_mod.enable" },
    "mode":    { "type": "string", "default": "off",
                 "options": [ "off", { "value": "fast", "label": "Fast", "label_ko": "빠르게" } ],
                 "invalidates": ["psGCodeExport"], "label": "Mode" }
}
```

| 키 | 의미 |
|---|---|
| `type` ✅ | `float`, `int`, `bool`, `string`, `float_or_percent` (`12.5` 또는 `"50%"`) |
| `default` ✅ | 타입에 맞는 기본값. **기능이 꺼진 중립값으로** (6.3절) |
| `scope` | `process`(기본, 프로세스 프리셋 `mod_settings`), `printer`(`printer_mod_settings`), `filament`(`filament_mod_settings`, 필라멘트마다, 코드에서는 `"<name>[i]"`), `object`(`object_mod_settings`), `region`(`region_mod_settings`: 객체, 레이어 범위, 파트, 모디파이어별, 가장 구체적인 값이 이김) |
| `label`, `label_<lang>`, `unit` | 표시용. (`page`는 파싱되지만 지금 Mods 페이지 배치에 쓰이지 않음, 4.9절의 `ui`가 대신할 예정) |
| `invalidates` | 값이 바뀔 때 다시 할 슬라이싱 단계 (4.7절) |
| `options` | `string` 전용 드롭다운. 값 문자열 또는 `{"value","label","label_<lang>"}`. 기본값이 목록에 있어야 하고, 저장된 값이 목록 밖이면 타입 오류로 보고 |
| `choices_from` | `string` 전용 (`options`와 함께 못 씀). 값이 `"<슬롯 이름>"`이면 드롭다운이 `orca` + 로드된 모드가 그 슬롯에 등록한 구현 이름들이 됨 (예: `mixed_support`의 `"choices_from": "support.areas"`). **값 검증은 모드가 직접** |
| `visible` | 파라미터만 읽는 식. 거짓이면 행을 숨김. **표시 전용**: 숨겨도 값과 규칙은 그대로이므로, 동작 제어는 규칙의 `when`이나 슬롯의 `when`에서 |

파라미터는 설정 탭의 **Mods 페이지**에 행으로 나옵니다. `object`/`region` 파라미터는 프로세스 Mods 페이지에서 기본값을 갖고, 객체 우클릭 "Mod settings..."에서 객체·영역별로 바꿀 수 있습니다. 규칙은 프로세스 값을 읽고, 네이티브 코드는 호출 컨텍스트(객체/영역)의 값을 읽습니다.

프리셋이나 CLI에서는 JSON 문자열로 저장됩니다: `"mod_settings": "{\"my_mod.enable\":true}"`.

### 4.3 규칙 (`rules`, JSON 모드의 핵심)

```json
"rules": [
    { "target": "initial_layer_speed", "value": "my_mod.speed", "when": "my_mod.enable" },
    { "target": "nozzle_temperature", "index": 0, "value": "my_mod.temp", "when": "my_mod.enable", "priority": 1 }
]
```

- `target`: 프린트, 프린터, 필라멘트의 아무 설정 키. `index`는 리스트 설정의 한 원소.
- `value`, `when`: 식 (4.8절). 높은 `priority`가 나중에 적용되어 이깁니다. 같은 설정에 같은 priority 규칙 두 개는 함께 로드되지 않습니다. 다른 모드끼리 다른 값을 쓰면 높은 priority가 이기고 경고합니다.
- 규칙은 프린터/필라멘트/프로세스 프리셋 위, **사용자가 프리셋에서 직접 바꾼 값 아래**에 적용됩니다.
- 평가할 수 없는 규칙은 슬라이스를 멈춥니다 (반쯤 적용된 상태로 슬라이스하지 않음).
- **G-code 설정** (`*_gcode`, `*_gcodes`, `post_process`): 모든 모드가 쓸 수 있습니다 (0장, `ModSet.cpp`). 사용자의 프린터가 그 텍스트를 그대로 실행하므로 기본값에서는 아무것도 바꾸지 말고 `when`으로 켜세요.

### 4.4 슬롯 선언 (`slots`)

```json
"slots": [ { "slot": "support.areas", "version": 1, "impl": "my_support",
             "when": "my_mod.enable", "priority": 0,
             "settings": { "name": "<식>" },
             "menu": { "param": "enable", "entries": [
                 { "label": "Mine (auto)", "label_ko": "내 것 (자동)", "support_type": "normal(auto)" } ] } } ]
```

- `slot`, `version`, `impl`은 필수이고, 라이브러리가 `register_slot`으로 **정확히 같은 세 값**을 등록해야 합니다. 선언하지 않은 것을 등록하거나 선언한 것을 등록하지 않으면 로드 실패 (`NativeMods.cpp`).
- 같은 `(slot, impl)`을 다른 모드가 이미 등록했으면 실패합니다. impl 이름에 모드 고유 접두사를 쓰세요.
- **코어가 하나를 고르는 슬롯** (`support.areas`, `support.compose`, `support.contacts`, `wall.plan`, `gcode.time_plan`):
  - `when`이 참인 선언 중 가장 높은 `priority`가 선택됩니다. **`when`이 없는 선언은 절대 선택되지 않습니다** (`ModSet.cpp`). 반드시 켜는 파라미터를 두고 `when`에 쓰세요.
  - 같은 priority에서 두 모드가 참이면 슬라이스가 멈춥니다. 공식 서포트 모드는 `mixed`=2, `resin`=1, `cura`=0을 씁니다.
  - `when`은 그 작업 단위의 값으로 평가됩니다: 서포트는 객체, 벽은 영역, 시간 계획은 프린트.
  - `settings`: 선택된 구현에 대해 코어가 읽는 값 (예: `wall.plan`의 `detail_wall_speed`, `gcode.time_plan`의 `wait_at`, `tower_size`, `park_x`, `park_y`, `heat_rate`, `cool_rate`).
- **모든 구현을 묻는 슬롯** (`support.style`, `layer.bands`, `gcode.events`, `object.z_offset`, `printer.agent`): `when`/`priority`로 고르지 않습니다. 각 구현이 호출될 때 "해당 없음"을 스스로 답합니다 (상태 null, 0 반환 등).
- `menu` (`support.*` 슬롯 전용): 서포트 유형 드롭다운에 항목 추가. `param`은 이 모드의 **process 범위 bool 파라미터**. 항목을 고르면 `support_type`을 그 값(Orca의 `SupportType` 값: `normal(auto)`, `normal(manual)`, `tree(auto)` ...)으로 쓰고 이 모드의 `param`을 켜고 다른 메뉴 모드는 끕니다. 메뉴 파라미터는 Mods 페이지에 나오지 않습니다.
- `when`이나 `settings`가 읽는 파라미터는 **반드시 `invalidates`를 선언**해야 합니다. 그 값에서 쓰이는 설정이 없어 자동 무효화가 일어나지 않기 때문입니다.

### 4.5 선언형 GUI (`gui`)

모드 코드는 GUI에서 실행되지 않습니다 (데이터로 답하는 질문 제외). GUI는 선언을 읽어 그립니다.

| 키 | 내용 |
|---|---|
| `paint_channels` | 최대 2개 `{"id", "color": "#RRGGBB", "label", "label_<lang>"}`. 서포트 페인트 도구의 채널. `paint_when` 식이 참인 동안 첫 모드의 채널이 나오고, `support.compose`에 `paint_channel` 0/1로 전달 |
| `paint_when` | 위의 조건 식 |
| `calibrations` | `{"id", "label", "description", ..._<lang>}`. 보정 메뉴 항목. **앱에 그 id의 대화상자가 있을 때만** 나옵니다 (지금은 `lesic`만). 제3자는 사실상 쓸 수 없음 |
| `aux_towers`, `legends` | `{"id", "label", "unit", ...}` 파싱됨. 현재 사용처는 제한적 |
| `object_params` | 객체 "Mod settings..."의 행 순서. `object`/`region` 파라미터 이름만 |
| `printer` | 프린터 연결 패널 (12장) |

### 4.6 `legacy_keys`

옛 프로젝트·프리셋의 설정 키를 이 모드의 파라미터로 옮깁니다. 꺼진 모드의 매니페스트에서도 읽힙니다.
- `"old_key": "param"`
- `"old_key": {"param": "p", "saved_default": "0"}`: 옛 파일이 늘 저장하던 의미 없는 값은 무시
- `"old_key": {"point": "pos"}`: `"XxY"` 위치를 `pos_x`/`pos_y`로

새 모드라면 보통 필요 없습니다.

### 4.7 무효화 (`invalidates`)

값이 바뀌었을 때 다시 할 단계 이름입니다 (`mod_step_names()`, `ModSet.cpp`):

`posSlice`, `posPerimeters`, `posEstimateCurledExtrusions`, `posPrepareInfill`, `posInfill`, `posIroning`, `posContouring`, `posSupportMaterial`, `posSimplifyPath`, `posSimplifySupportPath`, `posDetectOverhangsForLift`, `posSimplifyWall`, `posSimplifyInfill`, `psWipeTower`, `psToolOrdering`, `psSkirtBrim`, `psGCodeExport`, `none`

- 목록이 없으면 슬라이싱부터 전부 다시 합니다. 느리지만 틀리지는 않습니다.
- `["none"]`: 이 값은 규칙이 쓰는 설정을 통해서만 출력에 영향을 줌 (그 설정이 스스로 무효화). `object`/`region` 파라미터는 `none`을 쓸 수 없습니다 (네이티브 코드가 읽으므로).
- 슬롯별 기준: 서포트 계열 `posSupportMaterial`, 벽 `posPerimeters`, `layer.bands`는 벽이면 `posPerimeters`, 서포트면 `posSupportMaterial`, G-code 이벤트·시간 계획은 `psGCodeExport` (공식 `low_temp_interface`는 타워가 있어 `psWipeTower`).
- **빠뜨리면 값이 바뀌어도 이전 결과가 남습니다(stale)**. 확신이 없으면 목록을 생략하세요.

### 4.8 식(expression)

설정은 키로(`layer_height`), 파라미터는 `<id>.<name>`으로 읽습니다. 산술, 비교, `and`/`or`/`not`, `if(c, a, b)`, `min`/`max`, 큰따옴표 문자열을 지원합니다. `visible`은 파라미터만 읽을 수 있습니다. 프린터 패널(`gui.printer`)의 `visible`은 예외로, 그 패널의 필드 id와 `printer_state`를 읽습니다 (12.3절).

### 4.9 (진행 중) 파라미터를 Orca 설정 옆에 놓기: `"ui"`

> 새로 들어간 기능입니다. 이 필드를 모르는 이전 앱은 무시하므로 넣어도 해가 없습니다.

파라미터 정의에 `"ui"`를 넣으면, Mods 페이지 대신 설정 탭의 Orca 설정 옆에 행이 나옵니다. 그 브랜치의 `ModParamUi`(`ModSet.hpp`) 기준입니다.

| 키 | 의미 |
|---|---|
| `after` | 이 Orca 설정 키(예: `"outer_wall_speed"`)의 줄 바로 다음에 한 줄로 |
| `page`, `group` | 또는 페이지 `page`(예: `"Quality"`)의 그룹 `group`에. 그런 그룹이 있으면 그 안에, 없으면 새 그룹을 만듦 |
| `group_after` | 새 그룹을 이 그룹(예: `"Precision"`) 바로 뒤에. 없으면 페이지 끝 |
| `widget` | `""`(타입대로), `"toolhead"`(int: 0 = 자동, n = n번 툴헤드, 프린터 노즐 이름의 드롭다운), `"layer_ranges"`(`first:last:toolhead` 항목들의 문자열을 표로) |

- 이름은 Orca 소스에 쓰인 영어 그대로입니다(번역된 화면 이름이 아님).
- 탭에 그 자리가 없으면 행은 Mods 페이지로 돌아갑니다.
- 예: `"ui": { "page": "Quality", "group": "Multi-nozzle walls", "group_after": "Precision", "widget": "toolhead" }` (그 브랜치의 `mods/multi_nozzle_walls/mod.json`).

---

## 5. 네이티브 ABI

### 5.1 세 개의 export

```c
const char *magpie_mod_query(void);                                      /* 부작용 없음, init 전에 호출됨 */
int32_t     magpie_mod_init(const MagpieHostApi *host, MagpieRegistrar *registrar);
void        magpie_mod_shutdown(void);                                   /* 마지막 호출 */
```

C++에서는 `MAGPIE_MOD_DEFINE("my_mod", "1.0.0")` 한 줄이 세 함수를 만들고, 여러분은 `magpie::mod_init`과 `magpie::mod_shutdown`만 정의합니다.

`magpie_mod_query`는 정적 문자열 `"<id> <version> <major> <minor> fp=<mode>"`를 돌려줍니다 (`MAGPIE_MOD_QUERY_STRING`). 코어가 거부하는 경우 (`native_query_problem`, `NativeMods.cpp`):

| 조건 | 메시지 |
|---|---|
| id가 mod.json과 다름 | `the library is mod 'x', not 'y'` |
| 버전이 다름 | `the library is version A but mod.json says B` |
| major가 코어와 다르거나 mod.json `api.major`와 다름 | `it was built for mod API ...` |
| mod.json `min_minor` > 코어 minor | `it needs mod API 1.N; this application has 1.M. Update the application` |
| fp가 `precise`가 아님 | `it was compiled with floating-point mode "..."` |

### 5.2 로드 순서

1. 매니페스트 파싱 → 의존성 순서로 정렬 → 서로 맞는지 검사 (하나가 나빠도 나머지는 로드)
2. (설치 모드) 서명 확인. 신뢰된 서명이 없으면 허락 기록을 보고, 없으면 경고 대화상자(GUI) 또는 건너뜀(CLI) (0.2절) → 캐시로 복사 → 서명된(또는 허락한) 해시와 비교
3. 라이브러리 로드 → 세 export 확인 → `magpie_mod_query` 검사
4. `magpie_mod_init(host, registrar)`. `MP_OK`가 아니면 등록한 것을 되돌리고 `magpie_mod_shutdown`을 부른 뒤 제외. **그 모드에 의존하는 모드도 제외**
5. 선언한 슬롯이 모두 등록됐는지 확인

init에서 실패할 때는 먼저 `host->log(MP_LOG_ERROR, id, "이유")`를 남기세요.

### 5.3 데이터와 메모리 소유권

- 경계를 넘는 것: 정수, 실수, `mp_point`/`mp_ring`/`mp_expoly(s)`/`mp_rings`, NUL 종료 UTF-8 문자열, 불투명 핸들(`mp_ctx*`, `mp_handle`, `void *state`). **C++ 객체, STL 타입, 예외는 절대 넘기지 않습니다.**
- 모든 테이블과 입출력 구조체는 첫 필드가 `uint32_t size`입니다. 채울 때 `size = sizeof(...)`를 넣으세요.
- **할당한 쪽이 해제합니다.**
  - 호스트가 만든 결과(`host->offset`, `host->boolean`, `core.geometry`, compose의 레이어 목록)는 호스트의 해제 함수로 (`free_expolys`, `free_rings`, `free_layers`).
  - 모드가 만든 결과는 (a) `host->alloc`으로 할당하면 코어가 해제하거나, (b) 슬롯 테이블의 `release`로 모드가 해제합니다. 어느 쪽인지는 슬롯마다 다르며 5.9절 표에 있습니다.
  - `mp_status.message`는 정적 문자열이거나 **그 모드의 다음 호출까지** 유효해야 합니다. `magpie::guarded`는 thread-local 버퍼를 씁니다.
- 입력 포인터(코어가 넘긴 폴리곤, 메시, 문자열)는 **그 호출 동안만** 유효합니다. 보관하려면 복사하세요.
- 등록한 슬롯 테이블과 `user` 포인터는 `magpie_mod_shutdown`까지 살아 있어야 합니다 (보통 `static const` 테이블).
- 좌표는 정수 단위이고 **1단위 = `host->scaling_factor_mm` mm**입니다. 보통 1e-6 mm이지만 아주 큰 베드에서는 1e-5 mm로 바뀝니다. **매 호출마다 읽고 캐시하지 마세요.** 다른 길이는 mm, 시간 s, 온도 °C, 속도 mm/s.
- 외곽선은 반시계, 구멍은 시계 방향. 링의 마지막 점은 첫 점을 반복하지 않습니다.

### 5.4 호스트 테이블 `MagpieHostApi`

| 함수 | 용도 |
|---|---|
| `param_f64/param_i64/param_str(ctx, key, ...)` | 이 모드의 파라미터. 우선순위: 컨텍스트 영역 값 → 객체 값 → 프로세스 프리셋 → 기본값. 필라멘트 범위는 `"name[i]"`(0부터). `param_f64`는 float/int/bool(0/1), `param_i64`는 int/bool, `param_str`은 string과 float_or_percent. 없으면 `MP_ERR_NOT_FOUND`, 타입이 다르면 `MP_ERR_TYPE`, 버퍼 부족은 `MP_ERR_BUFFER`(잘린 채 NUL 종료) |
| `config_f64/config_str(ctx, key, index, ...)` | 슬라이서 설정. `index`는 리스트 원소(-1은 단일). 컨텍스트에 영역/객체가 있으면 그 값. 퍼센트는 `config_f64`에서 숫자, `config_str`에서 `"50%"` |
| `offset`, `boolean`, `free_expolys` | 코어 Clipper. delta는 정수 단위 |
| `alloc`, `free` | 코어가 해제할 메모리 |
| `log(level, mod_id, text)` | 앱 로그 한 줄 |
| `warn(ctx, level, text)` | 사용자에게 보이는 슬라이스 경고(`MP_LOG_WARNING`), 또는 오류(`MP_LOG_ERROR`, 호출이 반환되면 슬라이스 중단). 영어로 쓰고 번역은 7장 |
| `cancelled(ctx)` | 사용자가 취소했으면 0이 아님. 긴 루프에서 확인하고 `MP_ERR_CANCELLED` 반환 |
| `parallel_for(ctx, n, fn, user)` | 코어 스레드 풀(TBB)에서 `fn(user, i)`, 모두 끝나면 반환 |
| `scaling_factor_mm`, `app_version`, `major`, `minor`, `size` | 정보 |

나중 minor에서 추가된 항목을 쓰기 전에는 `MAGPIE_HOST_HAS(host, member)`로 테이블 크기를 확인하세요.

### 5.5 스레드

- `"thread_safe": false`(기본): 코어가 **모드 단위 뮤텍스로 모든 호출을 직렬화**합니다 (`NativeMod::guarded_call`).
- `"thread_safe": true`: 여러 스레드에서 동시에 호출됩니다. 공유 상태는 모드가 잠급니다. 공식 모드는 모두 `true`입니다.
- `parallel_for`의 `fn`은 **TBB 작업 스레드에서 동시에** 실행됩니다. `thread_safe`와 상관없이 `fn`은 스레드 안전해야 합니다.
- 직렬화 뮤텍스는 재진입 불가(`std::mutex`)입니다. `thread_safe: false`인 모드가 호출 도중 호스트를 통해 자기 자신을 다시 부르게 되면(예: `support.compose`에서 `run_normal`로 자기 `support.areas` 구현 실행, 자기가 export한 서비스를 거쳐 되돌아오는 경우) **교착될 수 있습니다 (추론, 코드 구조에서)**. 그런 구조라면 `thread_safe: true`로 하고 스스로 잠그세요.
- `printer.agent`는 `thread_safe: true`가 아니면 등록되지 않습니다.
- 전역 상태를 피하고, 상태는 `open`이 돌려주는 `state`에 두세요. 같은 모드의 다른 객체·영역 호출이 동시에 올 수 있습니다.

### 5.6 오류, 예외, 크래시

- **예외는 경계를 넘으면 안 됩니다.** 모든 슬롯 함수 본문을 `return magpie::guarded([&]{ ... });`로 감싸면 C++ 예외가 `{MP_ERR_FAILED, 메시지}`로 바뀝니다. `MAGPIE_MOD_DEFINE`은 init/shutdown을 이미 감쌉니다.
- `MP_OK`가 아닌 상태는 슬라이스 오류가 되어 `"<mod id>: <메시지>"`로 사용자에게 보입니다 (메시지는 7장의 카탈로그로 번역). `MP_ERR_CANCELLED`는 취소로 처리됩니다.
- **크래시 가드 (Windows만)**: 접근 위반, 0으로 나누기, 스택 오버플로 같은 하드웨어 예외는 SEH로 잡혀 슬라이스 오류 `mod 'x' 1.0 crashed: <예외>. It is disabled until the application restarts`가 되고, **그 모드는 앱을 다시 시작할 때까지 꺼집니다** (`NativeCall.cpp`, `NativeMods.cpp`). `parallel_for` 작업 스레드의 크래시도 같은 경로로 옵니다. Linux/macOS에는 가드가 없어 크래시가 앱을 끝냅니다.
- 크래시 후 프로세스 상태는 신뢰할 수 없습니다. 가드는 안전망일 뿐이고 정상 경로가 아닙니다.

### 5.7 결정성(determinism)

- **같은 입력이면 같은 출력**이어야 합니다. `--mod-test`가 새 프로세스 두 번으로 확인합니다.
- 금지: 시간, 난수(고정 시드 없이), 해시 맵의 순회 순서, 포인터 값 정렬, 스레드 완료 순서에 의존하는 결과, 초기화되지 않은 메모리.
- `parallel_for` 결과는 인덱스 `i`의 칸에 쓰고, 합칠 때는 인덱스 순서로.
- 부동소수점은 템플릿이 고정합니다. 컴파일 플래그를 덮어쓰지 마세요 (`/fp:fast`, `-ffast-math`, `-mfma` 등).
- 코어가 같은 질문을 여러 번 할 수 있습니다 (`object.z_offset`, `layer.bands.plan`은 설정 적용 때마다). 답은 파라미터와 설정에만 의존해야 합니다.

### 5.8 버전 호환

- API major가 같은 동안 이전 빌드의 모드가 새 코어에서 그대로 로드됩니다. major가 바뀌면 다시 빌드해야 합니다.
- minor는 항목이 **뒤에 추가될 때** 오릅니다. 새 쪽은 상대가 준 `size` 너머를 무시합니다.
  - 모드가 새 필드를 쓰려면: 구조체의 `size`가 그 필드를 덮는지 확인하고 씁니다 (예: API 1.1의 `mp_support_style_object_v1.sublayer_temperature`, `mp_support_output_v1.fractional_*`, `core.geometry.boolean_ex`).
  - 새 필드가 필요하면 `"api": {"major": 1, "min_minor": 1}`로 선언해 낡은 앱에서 로드되지 않게 합니다.
- 슬롯 테이블은 **모든 함수 포인터를 채우세요.** 코어는 테이블 크기가 그 버전의 구조체보다 작으면 그 구현을 조용히 건너뛰고(로그만), `wall.plan`은 null 항목도 거부합니다. 할 일이 없는 함수는 "해당 없음"을 반환하는 빈 함수로.
- 모드 버전을 올릴 때는 `mod.json`의 `version`과 `MAGPIE_MOD_DEFINE`의 버전을 함께 바꾸세요. G-code 헤더 `; mods:` 줄에 들어가 출력이 바뀌므로 테스트의 `contains`도 같이.

### 5.9 슬롯 요약

| 슬롯 | 헤더 | 고르는 방식 | 단위 | 상태/소유권 |
|---|---|---|---|---|
| `support.areas` v1 | `magpie_support.h` | 하나 선택 (객체) | 객체 | 출력 배열은 모드 소유, 코어가 `release` 호출 |
| `support.compose` v1 | `magpie_slot_support_compose.h` | 하나 선택 (객체) | 객체 | 호스트 레이어 목록은 `free_layers`, 세트 핸들은 `release`/`merge` |
| `support.contacts` v1 | `magpie_slot_support_contacts.h` | 하나 선택 (객체) | 객체 | `mod_data`에 몸체 보관, `release`로 해제. `body` 결과는 다음 `body`/`release`까지 모드 소유 |
| `support.style` v1 | `magpie_slot_support_style.h` | 모두에게 물음, 한 객체에 하나만 | 객체 | `open`의 state, `close` |
| `object.z_offset` v1 | `magpie_slot_object_z_offset.h` | 모두에게 물음, 한 객체에 하나만 | 객체 | 없음 |
| `wall.plan` v1 | `magpie_slot_wall_plan.h` | 하나 선택 (영역) | 영역 | `open`의 state, `close`. 아무 스레드에서 호출 |
| `layer.bands` v1 | `magpie_slot_layer_bands.h` | 모두에게 물음, 한 객체에 하나, 프린트 전체가 같은 구현 | 객체 | 없음 |
| `gcode.events` v1 | `magpie_slot_gcode_events.h` | 모두에게 물음, 로드 순서대로 | 내보내기 1회 | `open`의 state, `close`. 문자열 결과는 `host->alloc` |
| `gcode.time_plan` v1 | `magpie_slot_time_plan.h` | 하나 선택 (프린트) | 프린트 | 코어가 배열 할당 |
| `printer.agent` v1 | `magpie_slot_printer_agent.h` | 모두 등록 | 연결 | `create`/`destroy`, 문자열은 모드 할당 + `release` |

### 5.10 서비스

- `registrar.export_service(registrar, "<이름>", 버전, 테이블, 크기, user)`: 다른 모드에 기능 제공. 이름은 `"<mod id>.<무엇>"` 관례 (`cura.support_areas`, `mixed_support.plan`).
- `registrar.import_service(registrar, "<이름>", 최소버전, &size, &user)`: 다른 모드의 서비스 사용. 제공 모드를 `depends`에 적어 먼저 초기화되게 하세요. 없으면 null.
- `core.*`는 코어 자신의 서비스이며 `depends`가 필요 없습니다. 코어 서비스를 import하면 `*user`에 그 모드를 가리키는 값이 들어옵니다(`NativeMods.cpp`). `core.net`은 이 값을 모든 호출의 첫 인자로 받아 호출한 모드를 확인하므로, 버리지 말고 보관하세요. `core.geometry`는 이 값을 쓰지 않습니다.
  - `core.geometry` v1: 코어 ClipperUtils의 `offset`, `boolean`(+`MP_OP_UNION_POSITIVE`), `simplify`, `union_ex`, `area`, `min_dist2`, `free_rings`, 그리고 나중에 추가된 `boolean_ex` (테이블 크기 확인). **코어 안에서 계산하던 것과 같은 폴리곤**이 필요하면 이것을 쓰세요.
  - `core.net` v1: 네트워크, 비밀 저장소, 스레드 (12장).

---

## 6. 규칙 (반드시 지킬 것)

### 6.1 ABI
- 데이터 전용 ABI: C 타입만, C++ 객체·STL·예외 금지.
- 모든 진입점은 상태를 반환하고 예외를 흘리지 않습니다 (`magpie::guarded`).
- 할당한 쪽이 해제. 호출 동안만 유효한 입력을 보관하지 않기.
- 코어 헤더나 슬라이서 소스를 include하지 않기 (템플릿이 막음).

### 6.2 결정성과 스레드
- 5.5절, 5.7절.

### 6.3 꺼짐 = Orca와 바이트 동일
- 기본값은 아무것도 바꾸지 않아야 합니다. 설치만 하고 켜지 않은 상태에서 G-code가 한 바이트도 달라지면 안 됩니다.
- 꺼져 있으면: 선택 슬롯은 `when`이 거짓, 묻는 슬롯은 상태 null / 0 / "띠 없음"을 반환.
- `gcode.events`에서 `MP_GCODE_USED` 플래그나 `writer->mark_used()`는 실제로 출력을 바꿀 때만. 이 플래그가 G-code 헤더에 `; mods: <id> <버전>` 줄을 추가하므로, 꺼진 상태에서 세우면 바이트 동일이 깨집니다.

### 6.4 성능
- 핫 경로: `gcode.events`의 `event`(경로마다 `PATH`), `path_speeds`, `wall.plan`의 `plan`(섬마다). 파라미터와 설정은 `open`에서 한 번 읽어 state에 두세요 (헤더도 그렇게 권함).
- 큰 작업은 `host->parallel_for`로 (자체 스레드 풀을 만들지 마세요).
- 긴 루프에서 `host->cancelled(ctx)`를 확인.
- `host->param_str`, `config_str`의 버퍼는 넉넉히 잡고 `MP_ERR_BUFFER`면 늘려 다시 (`magpie::param_str` 참고).

### 6.5 안전
- 네트워크, 파일 시스템, 프로세스 실행은 슬라이싱 슬롯에서 하지 마세요. 네트워크는 `core.net`만.
- 프린터에 위험한 출력: 베드 밖 이동, 온도 범위 밖, 비정상 압출은 `--mod-test`가 검사합니다 (9장). 그 검사를 통과해도 실제 프린터 안전은 모드 책임입니다.

### 6.6 권한

(0장) 공식, 인증, 인증 안 됨 세 등급의 권한이 같습니다. 다른 것은 표시와 경고뿐입니다.

| 기능 | 공식 | 인증 (파란색) | 인증 안 됨 (빨간색) |
|---|---|---|---|
| 네이티브 코드 로드 | ✅ | ✅ | ✅ 처음 한 번 경고 후 |
| G-code 설정(`*_gcode`, `post_process`)에 규칙 | ✅ | ✅ | ✅ |
| `gcode.events`의 시작 G-code 필터 (`MP_GCODE_WANTS_START_GCODE`) | ✅ | ✅ | ✅ |
| `gcode.events`의 나머지 (텍스트, 온도, 이동, 압출) | ✅ | ✅ | ✅ |

---

## 7. 다국어 (i18n)

- **매니페스트 문구**: `name_<lang>`, `label_<lang>`, `description_<lang>`, 옵션의 `label_<lang>`, 그리고 프린터 패널의 `help_<lang>`, `text_<lang>`, `confirm_<lang>`, `network.reason_<lang>`. 한국어는 `_ko`.
- **웹 패널 페이지(HTML)** 의 문구는 코어가 번역하지 않습니다. 페이지가 직접 언어를 고르세요(Anycubic 모드의 로그인 페이지는 `lang` 값을 받아 씁니다).
- **코드가 내보내는 문구** (슬라이스 경고·오류, `mp_status.message`, compose의 `status`/`warn`, `validate` 문제 문장): **영어로** 쓰고 패키지의 `i18n/<lang>.json`에 번역을 둡니다.

```json
{
    "My feature needs at least two walls.": "내 기능은 벽이 2개 이상 필요합니다.",
    "Layer %1% is too thin for %2%.": "레이어 %1%은(는) %2%에 비해 너무 얇습니다."
}
```

- 형식: 영어 원문 → 번역의 JSON 객체. 객체가 아니면 **모드 로드 실패**.
- 찾는 순서 (`ModHost::mod_text`): 모드 자신의 카탈로그(앱 언어, `ko_KR` → `ko`) → 앱 번역 → 원문.
- 원문과 정확히 같은 문자열이어야 찾습니다. 문장을 조각내 붙이지 말고, 숫자는 `%1%`~`%4%` 자리표시자로 (`mp_gcode_problem_v1`).
- 카탈로그도 서명 대상입니다. 서명 후 수정하면 서명이 깨집니다.

---

## 8. Hello 모드 따라 하기

### 8.1 가장 작은 JSON 모드 (서명 불필요)

`%APPDATA%\MagpieSlicer\mods\first_layer_slowdown.json`:

```json
{
    "id": "first_layer_slowdown",
    "name": "First layer slowdown", "name_ko": "첫 레이어 감속",
    "version": "1.0.0",
    "params": {
        "enable": { "type": "bool",  "default": false, "label": "Slow the first layer", "label_ko": "첫 레이어 느리게",
                    "invalidates": ["none"] },
        "speed":  { "type": "float", "default": 20, "unit": "mm/s", "label": "First layer speed", "label_ko": "첫 레이어 속도",
                    "invalidates": ["none"], "visible": "first_layer_slowdown.enable" }
    },
    "rules": [
        { "target": "initial_layer_speed", "value": "first_layer_slowdown.speed", "when": "first_layer_slowdown.enable" }
    ]
}
```

앱을 다시 시작하면 프로세스 설정 탭의 Mods 페이지에 두 행이 생깁니다.

### 8.2 가장 작은 네이티브 모드: 레이어마다 주석 한 줄

`gcode.events` 슬롯으로, 켜면 각 레이어 시작에 `; <메시지>` 주석을 씁니다. 꺼져 있으면 상태 null이라 출력이 바뀌지 않습니다.

폴더 구조:

```
hello_layers/
    CMakeLists.txt
    mod.json
    hello_layers.cpp
    i18n/ko.json
    tests/
        cube.json
        cube.stl            (아무 작은 모델)
        presets/machine.json
        presets/process.json
```

**`mod.json`**

```json
{
    "id": "hello_layers",
    "name": "Hello layers", "name_ko": "레이어 인사",
    "version": "0.1.0",
    "api": { "major": 1, "min_minor": 0 },
    "binary": "hello_layers",
    "slots": [ { "slot": "gcode.events", "version": 1, "impl": "hello_layers" } ],
    "thread_safe": true,
    "params": {
        "enable":  { "type": "bool",   "default": false, "invalidates": ["psGCodeExport"],
                     "label": "Write a comment at each layer", "label_ko": "레이어마다 주석 쓰기" },
        "message": { "type": "string", "default": "hello", "invalidates": ["psGCodeExport"],
                     "visible": "hello_layers.enable", "label": "Comment text", "label_ko": "주석 내용" }
    }
}
```

**`hello_layers.cpp`**

```cpp
// hello_layers: a comment line at the start of each layer while "enable" is on. Off returns no state, so the
// G-code stays byte-identical.
#include <magpie_mod.hpp>
#include <magpie_slot_gcode_events.h>

#include <memory>
#include <stdexcept>
#include <string>

#define HELLO_ID "hello_layers"
MAGPIE_MOD_DEFINE(HELLO_ID, "0.1.0")

namespace {

struct State
{
    const mp_gcode_writer_v1 *writer = nullptr; // valid from open until close
    std::string               line;             // "; <message>\n", read once at open
};

mp_status open(void *, mp_ctx *ctx, const mp_gcode_export_v1 *, const mp_gcode_writer_v1 *writer, void **state, uint32_t *flags)
{
    return magpie::guarded([&] {
        *state = nullptr;
        *flags = 0;
        if (magpie::param_i64(ctx, "enable").value_or(0) == 0)
            return; // off: not called again for this export
        std::string message = magpie::param_str(ctx, "message").value_or("hello");
        for (char &c : message)
            if (c == '\n' || c == '\r')
                c = ' '; // writer->text takes comment lines only
        auto s    = std::make_unique<State>();
        s->writer = writer;
        s->line   = "; " + message + "\n";
        *flags    = MP_GCODE_USED; // named in the "; mods:" header line
        *state    = s.release();
    });
}

mp_status event(void *, void *state, mp_ctx *, mp_gcode_event_v1 *ev)
{
    return magpie::guarded([&] {
        const State &s = *static_cast<const State *>(state);
        if (ev->kind == MP_GCODE_EV_LAYER_BEGIN && s.writer->text(s.writer->w, s.line.c_str()) != MP_OK)
            throw std::runtime_error("The layer comment was refused.");
    });
}

mp_status support_last(void *, void *, mp_ctx *, int32_t *out) { *out = 0; return {MP_OK, nullptr}; }
mp_status tower_on_layer(void *, void *, mp_ctx *, double, int32_t *out) { *out = 0; return {MP_OK, nullptr}; }
mp_status path_speeds(void *, void *, mp_ctx *, const mp_gcode_speed_query_v1 *, double *, int32_t *changed)
{
    *changed = 0;
    return {MP_OK, nullptr};
}
mp_status start_gcode(void *, void *, mp_ctx *, int32_t, const char *, mp_gcode_text_v1 *) { return {MP_OK, nullptr}; }
void      close(void *, void *state) { delete static_cast<State *>(state); }
mp_status validate(void *, mp_ctx *, const mp_gcode_validate_v1 *, mp_gcode_problem_v1 *) { return {MP_OK, nullptr}; }
mp_status tower_preview(void *, mp_ctx *, int32_t, mp_tower_preview_v1 *out) { out->show = 0; return {MP_OK, nullptr}; }
mp_status tower_requested(void *, mp_ctx *, int32_t *out) { *out = 0; return {MP_OK, nullptr}; }

// Every entry filled: the core skips a table shorter than mp_gcode_events_v1.
const mp_gcode_events_v1 table = {sizeof(mp_gcode_events_v1), open, event, support_last, tower_on_layer, path_speeds,
                                  start_gcode, close, validate, tower_preview, tower_requested};

} // namespace

int32_t magpie::mod_init(const MagpieHostApi &, MagpieRegistrar &registrar)
{
    return registrar.register_slot(&registrar, MP_GCODE_EVENTS_SLOT, MP_GCODE_EVENTS_VERSION, HELLO_ID, &table,
                                   sizeof(table), nullptr);
}

void magpie::mod_shutdown() {}
```

**`i18n/ko.json`**

```json
{ "The layer comment was refused.": "레이어 주석이 거부되었습니다." }
```

**`CMakeLists.txt`** (슬라이서 소스 밖에서 단독 빌드)

```cmake
cmake_minimum_required(VERSION 3.16)
project(hello_layers CXX)

# SDK 폴더 경로 (저장소의 mods/sdk를 복사한 곳)
set(MAGPIE_SDK_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../sdk" CACHE PATH "Magpie mod SDK")
include(${MAGPIE_SDK_DIR}/MagpieMod.cmake)

# Windows: 정적 런타임 권장 (3.1절)
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

set(PACKAGE_DIR "${CMAKE_BINARY_DIR}/package/hello_layers")
magpie_add_mod(hello_layers SOURCES hello_layers.cpp OUTPUT_DIR "${PACKAGE_DIR}")

# 패키지 폴더 완성: mod.json, i18n, tests (공식 모드의 magpie_official_mod와 같은 배치)
configure_file(mod.json "${PACKAGE_DIR}/mod.json" COPYONLY)
file(GLOB _cat RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/i18n" "${CMAKE_CURRENT_SOURCE_DIR}/i18n/*.json")
foreach (_f IN LISTS _cat)
    configure_file("i18n/${_f}" "${PACKAGE_DIR}/i18n/${_f}" COPYONLY)
endforeach ()
file(GLOB_RECURSE _tests LIST_DIRECTORIES false RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/tests" "${CMAKE_CURRENT_SOURCE_DIR}/tests/*")
foreach (_f IN LISTS _tests)
    configure_file("tests/${_f}" "${PACKAGE_DIR}/tests/${_f}" COPYONLY)
endforeach ()
```

**빌드 (Windows, VS 2022 개발자 명령 프롬프트 또는 아무 셸)**

```bash
cmake -S hello_layers -B build-hello -G "Visual Studio 17 2022" -A x64
```

```bash
cmake --build build-hello --config Release
```

결과: `build-hello/package/hello_layers/` 아래에 `mod.json`, `bin/win-x64/hello_layers.dll`, `i18n/ko.json`, `tests/`.

**붙이기**
- `build-hello/package/hello_layers/` 폴더를 `%APPDATA%\MagpieSlicer\mods\hello_layers\`로 복사합니다. 앱을 시작하면 "인증되지 않은 모드" 경고가 한 번 뜨고, "위험을 이해하고 로드"를 고르면 빨간 "인증 안 됨" 태그로 로드됩니다. Gen의 인증 서명을 받은 폴더는 경고 없이 파란색 태그로 로드됩니다.

앱을 다시 시작하고, 프로세스 설정 Mods 페이지에서 켠 뒤 슬라이스하면 각 레이어 변경 뒤에 `; hello` 줄과 헤더에 `; mods: hello_layers 0.1.0`이 나옵니다. 로그(`%APPDATA%\MagpieSlicer\log`)에 `Mod not loaded: ...`가 있으면 14장을 보세요.

---

## 9. 테스트

### 9.1 `--mod-test <id>`

```bash
magpie-slicer.exe --mod-test hello_layers
```

`--datadir <폴더>`를 같이 주면 그 데이터 폴더의 모드로 테스트합니다 (자식 프로세스에도 전달됨).

검사 내용 (`ModHost/ModTest.hpp`, `src/OrcaSlicer.cpp` `run_mod_tool_actions`):
1. 모드가 **실제로 로드됐는지** (매니페스트, 네이티브면 query와 슬롯 등록). 즉 **로더를 통과해야** 테스트할 수 있습니다. 인증 안 된 네이티브 모드는 GUI에서 한 번 허락해 둬야 명령줄에서도 로드됩니다 (0.2절). 결과 줄에 `(official)`, `(verified)`, `(unverified)` 중 하나가 붙습니다.
2. 패키지 `tests/`의 각 케이스를 **별도 프로세스로 두 번** 슬라이스해 G-code가 같은지 (결정성; 생성 시각, 객체 id 줄은 정규화).
3. G-code 검사 `check_gcode()`: XY가 베드 밖(1 mm 여유), Z < 0 또는 최대 높이 초과, 온도 0~350 °C 밖, 비유한 수, 한 번의 E 이동 50 mm 초과.
4. 케이스의 `contains` 문구가 모두 있고 `not_contains`가 하나도 없는지.

출력 끝 줄은 `PASSED <id>` 또는 `FAILED <id>`, 종료 코드 0이면 통과. 작업 파일은 `%TEMP%\magpie-mod-test-<id>\<케이스>\<1|2>\`에 남고 실패 시 `output.txt`를 보세요.

### 9.2 케이스 파일

`tests/<이름>.json` (경로는 `tests/` 기준):

```json
{
    "description": "Cube with the comment on: every layer gets it.",
    "args": [ "--load-settings", "presets/machine.json;presets/process.json", "cube.stl" ],
    "contains": [ "; hello", "; mods: hello_layers 0.1.0" ],
    "not_contains": [ "; mods: hello_layers 0.0" ]
}
```

또는 `tests/<이름>.3mf`를 그대로 두면 그 프로젝트를 슬라이스합니다.

프리셋 예 (`mods/half_layer/tests/presets/`를 본뜸):

```json
{ "type": "machine", "name": "mod test printer", "from": "User", "instantiation": "true", "version": "2.4.0.0",
  "layer_change_gcode": "G92 E0" }
```

```json
{ "type": "process", "name": "hello test", "from": "User", "instantiation": "true", "version": "2.4.0.0",
  "mod_settings": "{\"hello_layers.enable\":true,\"hello_layers.message\":\"hello\"}" }
```

`contains`는 **사람이 정한 기대값**이어야 합니다. 모드가 계산한 값을 그대로 기대값으로 옮기지 마세요.

### 9.3 꼭 넣을 케이스
- **켠 케이스**: 기능이 실제로 출력을 바꾸는지 (`contains`).
- **끈 케이스** (또는 기본값 프리셋): `not_contains`에 `"; mods: <id>"`. 꺼짐 = 바이트 동일의 최소 확인.
- 경계값: 파라미터 최소·최대, 다중 객체, 객체별 값이 다른 경우 (`object`/`region` 범위면).

### 9.4 골든 비교 (권장)

Magpie 본 저장소는 `scripts/magpie/golden/`의 golden 스크립트로 여러 모델·프리셋 묶음을 "모드 없음(vanilla)"과 비교합니다. 제3자도 같은 원리로:
1. 모드를 **설치하지 않은** 앱으로 모델 묶음을 슬라이스 (기준).
2. 모드를 **설치만 하고 끈** 상태로 같은 묶음 → 기준과 정규화 후 **완전히 같아야** 합니다.
3. 켠 상태 → 차이가 기대한 곳에만 있는지 사람이 확인.

정규화는 `normalize_gcode_for_comparison`과 같게: 생성 프로그램·시각 줄과 객체 id만.

### 9.5 단위 테스트
공식 모드는 `mods/<id>/unit/`에 Catch2 테스트를 두고 모드 소스만 컴파일합니다 (코어 없이). 슬롯 함수의 순수 계산 부분을 분리해 두면 앱 없이 테스트할 수 있습니다 (`layer_time_blend/planner.cpp` 참고).

---

## 10. 서명과 배포

### 10.1 서명 형식
- `SIGNATURE` 파일: `{"format": 1, "key": "<키 이름>", "signature": "<128자 hex>"}`
- 서명 대상 메시지: `"magpie-mod-signature-v1\n"` 다음에, 패키지 폴더의 **`SIGNATURE`를 제외한 모든 파일**을 상대 경로(`/` 구분) 순으로 `경로\tSHA-256 hex\n`.
- 알고리즘: Ed25519 (OpenSSL).
- 키 이름이 앱의 신뢰 목록에 없으면 `it is signed with "<키>", which this application does not trust`.
- **서명 후 폴더에 파일을 더하거나 바꾸면** 검증 실패 (`the files differ from what "<키>" signed`). `tests/`, `i18n/`도 포함됩니다. 빌드 산출물(`.pdb`, `.ilk`, `.exp`, `.lib`)을 `bin/` 폴더에 남기지 마세요. 설치 모드가 인증 등급으로 로드되려면 `bin/<플랫폼>/`의 모든 파일이 서명에 포함돼야 합니다.

### 10.2 도구

```bash
magpie-slicer.exe --mod-keygen my-dev-key.pem
```
새 Ed25519 개인키를 쓰고 공개키 hex를 출력합니다 (이미 있으면 덮어쓰지 않음).

```bash
magpie-slicer.exe --mod-sign build-hello/package/hello_layers --mod-sign-key my-dev-key.pem --mod-sign-key-id my-dev-key
```
`SIGNATURE`를 쓰고 "this application trusts it" 또는 "not trusted here: ..."를 출력합니다. **`--mod-sign-key-id`를 빼면 기본값이 `magpie-official-2026`** 이라 공식 키 이름으로 서명되고, 공식 개인키가 아니면 검증에 실패합니다. 반드시 자기 키 이름을 주세요.

앱은 `my-dev-key`를 신뢰하지 않으므로 "not trusted here"가 나옵니다. 자기 키 서명은 배포 중 파일이 바뀌지 않았는지 확인하는 용도일 뿐이고, 모드 관리자에서는 계속 "인증 안 됨"으로 표시됩니다 (0장). 파란색 인증 태그는 Gen의 인증 키 서명으로만 붙습니다.

### 10.3 배포 경로
- **인증 서명**: 0.3절대로 서명 전 패키지와 소스를 Gen에게 보내면, Gen이 검토한 뒤 인증 키 `magpie-verified-2026`으로 서명합니다. 공식 키 `magpie-official-2026`은 Magpie 자체 모드에만 씁니다. 두 개인키는 모두 저장소 밖에 있습니다.
- **공식 모드로 편입**: 0.4절대로 소스 상태로 Gen에게 제출합니다. 컴파일된 DLL은 받지 않습니다.
- **서명 없이 배포**: 폴더를 그대로 나눠 줄 수 있습니다(사용자가 `mods` 폴더에 직접 복사). 받는 사람에게는 빨간 태그와 한 번의 경고가 뜹니다. Get Mods(인덱스)로는 서명 없는 네이티브 패키지를 아직 받지 않습니다 (0.5절).
- **모드 인덱스**: `{"version": 1, "mods": [...]}`, 항목은 `id`, `name`, `version`, `description`, `author`, `tags`, `trust`(`official`/`verified`/`community`), `url`(http/https/file), `sha256`. `url`이 `.zip`이면 패키지:
  - zip 루트에 `mod.json` (그 안에 `bin/`, `SIGNATURE` ...).
  - 설치 시 해시, id 일치, 폴더 밖으로 나가는 항목 없음, **바이너리가 있으면 신뢰된 서명** 확인 후 `<data dir>/mods/<id>.pending`에 풀고 다음 시작에 설치.
  - 인덱스 저장소 배치는 `scripts/magpie/mod-index` (`mods/<id>.json`, `mods/<id>.meta.json`, `build_index.py`, GitHub Actions). 작성자는 PR로 두 파일을 추가.
  - CLI: `--mod-search "단어"`, `--mod-install id1,id2`, `--mod-index <url>`. GUI: Get Mods..., 앱 설정 `mod_index_url`.
- 설치·제거·켜기·끄기는 Installed Mods에서 하고 **다음 시작에 적용**됩니다 (`mods_state.json`의 `disabled`).

---

## 11. 슬롯 상세 레퍼런스

공통: 모든 함수는 `mp_status`를 반환하고 `user`는 `register_slot`에 준 포인터입니다. `ctx`는 그 호출의 프린트/객체/영역 설정을 담고 `host->param_*`, `host->config_*`, `host->cancelled`, `host->warn`에 넘깁니다. 좌표는 정수 단위, 객체 좌표계.

### 11.1 `support.areas` v1 (`magpie_support.h`)
- 역할: 객체의 레이어마다 서포트 몸체·지붕(roof)·바닥 영역을 계산. 툴패스, 래프트, 인터페이스는 코어(Orca 일반 서포트)가 만듭니다.
- `compute(user, ctx, in, out)`: `in`은 레이어 수, 섬(`slices`), 인포서/블로커, 선택적 `mask`(다른 생성기가 나머지를 맡을 때 이 안의 오버행만), 베드 영역, 선폭·노즐 지름, `adhesion_margin`. `out->layer_count`를 입력과 같게(서포트 없음이면 0), `body/roof/bottom`을 `[layer_count]`로, `first_layer_dense`, `mod_data`. 추가 필드 `fractional_body/roof/drop`은 `out->size`가 덮을 때만.
- `overhang_demand(...)`: 지지하려는 영역만 `out->body`에 (다른 생성기가 객체를 나눌 때 사용).
- `release(user, out)`: `compute`/`overhang_demand`가 넣은 것을 해제.
- 서포트 유형, Z/XY 거리 등은 `host->config_*`로 직접 읽습니다. 선택된 구현 이름은 `config_str("support_generator")`.

### 11.2 `support.compose` v1
- 역할: 코어의 생성기들(Orca 일반, `support.areas` 모드, Orca 트리)을 마스크로 돌려 세트를 만들고, 최대 두 세트(일반 + 트리)를 병합. 레이어를 직접 만지지 않습니다.
- `compose(user, ctx, host, c)` 하나. `host` 테이블: `has_generator`, `demand`, `buildplate_shadow`, `paint_channel`, `free_layers`, `run_normal`, `run_tree`(style `organic`/`tree_slim`/`tree_strong`/`tree_hybrid`), `release`, `merge`(마지막에 한 번), `discard_tree_preview`, `status`, `warn`.
- 코어 생성기 안의 예외는 보관했다가 `compose`가 반환한 뒤 다시 던집니다. 모드 안으로는 던지지 않습니다.
- 오류 상태를 반환하면 코어가 모든 세트를 버립니다.

### 11.3 `support.contacts` v1
- 역할: 서포트가 객체에 닿는 곳(contacts)과 서포트 몸체를 정하고, 나머지(인터페이스, 간격, 래프트, 툴패스)는 Orca 일반 서포트에 맡김. `support_type`은 적용되지 않음.
- `compute`: 입력에 섬, 슬라이스 높이, 레이어 높이, 인포서/블로커, **메시**(mm, 프린트 높이), `print_z_min`, `raft_interface_top_z`, 서포트 노즐·선폭. 출력 `contacts[layer_count]`, `keep_narrow_islands`, `mod_data`(몸체).
- `body(user, ctx, of, n, z, out)`: 몸체를 높이 `z[0..n)`에서 자른 섬. 결과는 다음 `body`/`release`까지 모드 소유.
- `release`.

### 11.4 `support.style` v1
- 역할: 모든 서포트 생성기에 대해 인터페이스 패턴, 일반/트리 벽 수, 인터페이스 레이어별 패턴·각도(서브레이어), 그리고 API 1.1부터 서브레이어 노즐 온도.
- `open(user, ctx, out, state)`: 바꿀 게 없으면 `*state = NULL`. 객체당 두 구현이 스타일을 주면 슬라이스 중단.
- `interface_layer(user, state, number, total, out)`: `number`는 접촉 레이어가 1.
- `close`.
- `sublayer_temperature`는 `out->size`를 확인하고 씁니다. 코어가 펌웨어 냉간 압출 한계와 필라멘트 최대 온도를 검사합니다.

### 11.5 `object.z_offset` v1
- `z_offset(user, ctx, lift_mm)`: 서포트가 있는 객체를 얼마나 띄울지(mm, 0 이상). 서포트가 있는 객체만 띄웁니다. 답은 파라미터와 설정에만 의존.

### 11.6 `wall.plan` v1
- 역할: 다중 노즐 프린터에서 영역의 벽을 크기가 다른 노즐에 나눔 (바깥 몇 줄은 "디테일" 노즐).
- `open(user, ctx, printer, state)`: 영역당 한 번. 여기서 파라미터와 설정을 모두 읽어 state에. `printer`는 노즐 지름, 필라멘트 종류·색·수용성·툴헤드 (리스트 길이가 다를 수 있고, 끝을 넘는 인덱스는 첫 항목).
- `detail_tool(state, layer_id, out)`: 그 레이어의 디테일 노즐 (없으면 0 → Orca 벽).
- `plan(state, in, out)`: 섬 하나의 계획. 코어가 "Orca 벽"으로 미리 채워 줌.
- `layer_toolhead`, `wall_filament`, `tool_uses`: 레이어 전체 툴헤드, 태그별 필라멘트, 선폭 검사를 위한 사용 목록.
- 필라멘트·툴헤드는 1부터, 0은 없음. **아무 스레드에서** 호출됩니다. 테이블의 null 항목은 거부.
- 선언의 `settings`에 `detail_wall_speed`.

### 11.7 `layer.bands` v1
- 역할: 모델 레이어의 일부 경로를 더 얇은 물리적 띠(band)로 인쇄 (H/2).
- `plan(user, ctx, out)`: 객체마다, 설정 적용 때마다 호출. 아무것도 띠로 만들지 않으면 `wall_depths = 0, support = 0`.
- `schedule(user, timing, boundaries, n, out)`: 나중 띠 블록을 레이어 인쇄 순서의 어느 경계에 넣을지. `allowed == 0`은 고르면 안 됨. 못 고르면 `UINT32_MAX`.
- **현재 실행기는 두 개의 같은 띠(0.5, 0.5)와 바깥 벽 2줄만** 인쇄하고 다른 계획은 `Print::validate`에서 거부합니다. 객체당 하나의 구현, 프린트 전체가 같은 구현.
- 계획이 바뀌면 파라미터가 선언하지 않아도 벽과 서포트를 다시 합니다.

### 11.8 `gcode.events` v1
- 역할: G-code 내보내기의 정해진 지점에서 쓰기, 순서 요청, 보조 타워, 경로별 속도, 시작 G-code 필터.
- `open(user, ctx, info, writer, state, flags)`: 내보내기당 한 번, 쓰기 전. 할 일이 없으면 `*state = NULL`. 이때 `ctx`에는 객체 설정이 없습니다(프린트 설정만). `info->calibration`은 보정 종류(`"lesic"`, `"temp_tower"` ... 일반 출력은 `""`).
- `flags`: `MP_GCODE_WANTS_PATHS`(경로마다 `PATH` 이벤트), `WANTS_PATH_SPEEDS`, `WANTS_START_GCODE`, `CHANGES_TEMPERATURE`(레이어 안에서 온도를 바꾸면, 시간 추정용), `USED`(헤더 `; mods:`에 이름).
- `event(user, state, ctx, ev)`: `LAYER_BEGIN`, `PATH`, `TOOL_AHEAD`, `SUPPORT_END`, `OBJECTS_END`, `FIRST_LAYER_END`, `LAYER_END`. 여러 구현이 로드 순서대로 받습니다. `PATH`의 `owns_temperature`를 1로 쓰면 태그된 서포트 경로 온도를 모드가 맡음.
- `support_last`, `tower_on_layer`, `path_speeds`, `start_gcode`, `close`.
- 내보내기 밖에서도 쓰이는 함수: `validate`(객체 하나 검증, `out->message`에 영어 + `%1%`), `tower_preview`(3D 미리보기 타워), `tower_requested`(선폭 검사용).
- **writer** (open부터 close까지 유효, 이벤트 처리 중에만 출력 가능):
  - 상태: `filament`, `nozzle_of`, `filaments`, `layer_index`, `nominal_z`, `layer_height`, `last_height`, `first_layer`, `over_raft`, `position_known`, `printable_area`, `top_support_interface_z`, `has_support_layers`, `style_interface_temperature`, `calibration_value`, `mark_used`.
  - 출력: `text`(**모든 줄이 빈 줄이거나 `;` 주석**), `set_temperature`(펌웨어 형식은 코어가 선택, `MP_TEMP_WAIT_COOLING`), `dwell`, `aux_fan`, `travel`(코어 이동 규칙), `raw_travel`/`raw_move`/`set_feedrate`(그대로 쓰는 이동, 기계 좌표), `plate_frame`, `extrude`(코어 압출 코드, 느린 꼬리 선택), `tower_arrival`.
  - 서비스: `flow`, `flow_with_spacing`, `place_tower`.
  - 모드는 **다른 G-code 텍스트를 직접 쓰지 않습니다.** 코어가 끝내지 못한 출력(선폭 검사 거부 등)은 아무것도 쓰지 않고, 같은 호출의 이후 출력도 쓰지 않으며, 모드가 반환한 뒤 코어가 슬라이스를 멈춥니다.
- 좌표: `machine`은 G-code에 쓰이는 값(활성 툴의 익스트루더 오프셋 포함), `plate`는 오프셋 없는 플레이트 좌표.

### 11.9 `gcode.time_plan` v1
- 역할: 내보낸 G-code의 대기 지점마다 얼마나 기다릴지. 숫자만 다룹니다 (G-code, 기하 없음).
- `plan(user, ctx, sites, n, print_time_s, out)`: 코어가 `out`의 모든 배열을 n개로 할당하고 "대기 없음"으로 채워 둠. 모드는 결정한 것만 덮어씀. 경고는 `warnings_cap`까지, 종류만(문구는 코어가 작성·번역).
- 코어가 개수, 인덱스, 유한·음수 아님을 검사한 뒤 씁니다.
- 선언의 `settings`: `wait_at`, `tower_size`, `park_x`, `park_y`, `heat_rate`, `cool_rate`. 고른 이름을 제공하는 모드가 없으면 슬라이스 중단.

---

## 12. 프린터 연결: `printer.agent` 슬롯과 `core.net` 서비스

> 첫 사용 모드는 Anycubic 프린터 모드(LAN·Cloud, 1.2.0)이며, 소스는 이 저장소에 없습니다. 앱에 번들되지 않고 공식 키로 서명된 별도 패키지로 배포되며, 사용자는 그 폴더를 `%APPDATA%\MagpieSlicer\modsnycubic\`에 넣습니다. 슬롯 v1이 처음 쓰이는 중이라 새 필드나 이벤트가 더해질 수 있습니다(기존 것은 바뀌지 않음).

### 12.1 `printer.agent` v1 (`magpie_slot_printer_agent.h`)
- 각 구현이 Orca 프린터 에이전트 `mod:<mod id>`(구현 이름이 모드 id와 다르면 `.<impl>` 추가)로 등록되어, 프린터 프리셋의 **Printer Agent** 드롭다운에 나옵니다. Device 탭, AMS 보기, 전송 대화상자가 그대로 동작. 슬라이싱과 G-code는 건드리지 않습니다.
- **`"thread_safe": true` 필수** (아니면 등록 안 됨, `ModPrinterAgent.cpp`). 한 세션 호출이 GUI 스레드와 작업 스레드에서 옵니다.
- 함수: `create(events, session)`, `destroy`, `probe(conn_json, info_json)`, `connect(conn_json)`(시작만 하고 결과는 `events->connection`), `disconnect`, `send_command(cmd_json, result_json)`(빨리 반환), `upload(local_path, params_json, progress, ...)`(블로킹, 진행 콜백이 0이 아니면 취소), `start_print`, `list_files`, `camera_info`, `release`(모드가 반환한 문자열 해제).
- 경계를 넘는 텍스트는 UTF-8 JSON. `char **`로 반환하는 문자열은 **모드가 할당하고 코어가 `release`로 해제**.
- 상태는 Magpie 중립 상태 JSON `"v": 1` (`state`, `progress`, `temps`, `fans`, `units`, `errors`, `caps` ...). 필드는 추가만 됩니다. 명령은 `{"cmd": "pause"}` 같은 중립 명령. 모르는 명령은 `MP_ERR_NOT_FOUND`, 지금 못 하는 명령은 `MP_ERR_FAILED` + 사용자용 이유. **원시 G-code는 넘어가지 않습니다.** 전체 목록은 `docs/HLSD/mods.md`의 "Printer connections" 절에 있습니다.
- 비밀 입력값은 `conn_json`에 없고, `secret_keys`가 알려 주는 키로 `core.net`의 `secret_get`에서 읽습니다.
- 이벤트(`events->event`): `probe`(장치 목록 이름), `notice`(알림), `panel_message`(웹 패널로), `open_url`(브라우저로 열기, 12.4절), 그 밖은 프린터 패널로.

### 12.2 `core.net` v1 (`magpie_net.h`)
- **GUI 앱에만** 있습니다. CLI에서는 import가 null이며, 그래도 모드는 로드돼야 합니다.
- import 때 받은 `user`(`svc`)를 모든 호출의 첫 인자로 넘깁니다 (호출한 모드를 식별).
- 매니페스트 `"network": {"reason", "reason_<lang>", "endpoints": [{"host", "ports", "purpose"}]}`에 적은 호스트·포트에만 접속할 수 있고, **사용자가 그 목록에 한 번 동의한 뒤에만**. `host`는 `lan`(사설·링크로컬·`.local`), 호스트 이름, `*.<도메인>`. 버전이나 목록이 바뀌면 다시 묻습니다.
- 기능: `http`(코어 libcurl; GET/POST/PUT/DELETE, 리다이렉트 안 따라감, 인터넷 주소는 항상 인증서 검증, 자체 서명은 LAN만), `tls_connect/read/write/close`(코어 OpenSSL), `secret_put/get/delete`(OS 자격 증명 저장소, 모드별), `queue_on_main`, `thread_start/thread_join`(모드의 크래시 가드 아래에서 실행).
- 모드는 TLS 라이브러리를 링크하지 않습니다. URL, 헤더, 본문, 비밀을 로그에 남기지 마세요 (코어도 남기지 않음).
- 종료 전에 코어가 대기 중인 호출을 버리고 연결을 닫고 남은 스레드를 join합니다.

### 12.3 프린터 패널 (`gui.printer`)
`fields`(`id`, `type`: `text`, `secret`, `choice`+`options`, `path`; `label`, `help` 설명, `default`; secret은 기본값 없음), `actions`(`command`+`args`, `row`, `confirm`, `idle_only`), `panels`(모드 폴더 안 HTML `page`; `window.wx.postMessage` ↔ `panel_message`, `window.magpiePanelMessage`), `notices`. 각 항목에 `visible`(폼 값과 `printer_state`를 읽는 식). 모든 문구는 `_<lang>` 번역.

패널은 프린터 설정 탭의 **Printer connection...** 버튼에서 열립니다. 폼 값은 프린터 주소(Host)마다 따로 저장되므로, 주소를 먼저 넣어야 합니다.

### 12.4 클라우드 연결을 위한 코어 동작

> 아래 네 가지가 없는 이전 버전의 앱에서는 Anycubic Cloud 모드의 로그인 패널이 제대로 동작하지 않습니다.

클라우드 계정에 로그인해야 하는 프린터(예: Anycubic Cloud)를 위해 코어가 바뀐 부분입니다.

1. **웹 패널은 첫 연결 전에도 동작합니다** (`ModPrinterAgent::web_panel_command`).
   - 이전에는 모드 세션이 Device 탭에서 프린터에 연결할 때만 생겼습니다. Device 탭은 기본값에서 숨겨져 있어서, 로그인 페이지가 세션이 없다고 나왔습니다.
   - 이제 세션이 없을 때 웹 패널이 `panel_message`를 보내면 코어가 세션을 `create`만 하고 `connect`는 하지 않습니다. 이후 실제 연결은 그 세션을 그대로 씁니다. 그래서 `connect`가 오기 전에도 로그인 같은 일을 처리할 수 있어야 합니다.
   - 그런 세션을 만들기 전에, `network` 목록에 대한 사용자 동의를 먼저 받습니다. 동의하지 않으면 메시지는 전달되지 않습니다.
   - 모든 `panel_message`와 `panel_closed` 명령에 **`"conn"`** 이 붙습니다. 그 패널이 다루는 프린터의 연결 정보로, `connect`의 `conn_json`과 같은 형식입니다(`dev_id`, `host`, `port`, `fields`, `profile`, `secret_keys`). 모드는 이것으로 토큰을 어느 프린터의 비밀 키 아래에 저장할지 압니다. 주소(Host)가 비어 있으면 `conn`은 붙지 않습니다.
2. **새 이벤트 `open_url`**.
   - `events->event`로 `{"type": "open_url", "url": "https://..."}`를 보내면 코어가 사용자의 기본 브라우저로 엽니다(로그인 페이지 등).
   - **`https://`이고, 호스트와 포트가 매니페스트 `network` 목록에 있는 주소만** 엽니다. 다른 주소는 열지 않고 로그에 남깁니다.
   - 같은 페이지를 패널 안에서 열 수도 있습니다. Anycubic 모드는 두 방법을 모두 제공합니다.
3. **선택(choice) 필드는 값을 저장합니다**. 이전에는 화면에 보이는 라벨이 저장되고, 저장된 선택이 다시 표시되지 않았으며, 그 값을 읽는 `visible` 조건이 거짓으로 남았습니다(그래서 Anycubic Cloud 필드와 로그인 버튼이 안 보였습니다). 이제 `options`의 `value`가 저장되고 `visible` 식도 그 값을 읽습니다. 모드는 `conn_json.fields`에서 라벨이 아니라 `value`를 받는다고 보면 됩니다.
4. **프린터 에이전트는 모드 로드 뒤에 등록됩니다**.
   - 이전에는 Orca가 에이전트 목록을 모드보다 먼저 만들어서, `--datadir` 없이 시작하면 Printer Agent 드롭다운에 `mod:<id>`가 없었습니다.
   - 이제 모드를 로드한 직후 등록합니다. 네이티브 모드는 시작할 때 한 번만 로드되므로, 새 프린터 모드를 설치하면 앱을 다시 시작해야 목록에 나옵니다.
   - 같은 수정으로 프린터 설정의 Printer connection 줄이 전체 폭으로 바뀌었습니다(빈 옵션 줄에서 나던 크래시 수정).

사용자 쪽 참고: Orca 프린터(비 Bambu)의 Device 탭은 환경설정 `use_printer_agents`가 켜져 있어야 보입니다(기본 꺼짐). 로그인과 패널 동작은 1번 덕분에 이 설정과 상관없이 Printer connection 패널에서 됩니다.

---

## 13. 체크리스트

**매니페스트**
- [ ] `id`가 `a-z0-9_`, 숫자로 시작하지 않음, 패키지 폴더 이름과 같음, 공식 모드 id와 겹치지 않음
- [ ] `version`이 `MAGPIE_MOD_DEFINE`과 같음
- [ ] `api.major` = 1, 새 필드를 쓰면 `min_minor` = 1
- [ ] `binary`가 실제 파일 이름 (확장자·`lib` 접두사 없이)
- [ ] `slots`의 `(slot, version, impl)`이 코드의 `register_slot`과 정확히 같음
- [ ] 선택 슬롯이면 `when`이 있고 켜는 파라미터를 읽음, `priority`가 다른 모드와 겹치지 않음
- [ ] 모든 파라미터 기본값이 중립(꺼짐), `invalidates`가 맞거나 생략
- [ ] `when`/`settings`가 읽는 파라미터에 `invalidates`
- [ ] 라벨에 `label_ko`, 이름에 `name_ko`
- [ ] G-code 설정 규칙이 있다면 기본값(꺼짐)에서 아무것도 바꾸지 않음
- [ ] `author` 필드에 제작자 이름 (경고문에 표시됨)
- [ ] (프린터 모드) `thread_safe: true`, 접속하는 모든 호스트·포트를 `network`에, `reason`과 `reason_ko`

**코드**
- [ ] 모든 슬롯 함수가 `magpie::guarded`로 감싸짐, 예외가 새지 않음
- [ ] 테이블의 모든 함수 포인터가 채워짐, `size = sizeof(...)`
- [ ] 할당/해제 짝이 맞음 (호스트 결과는 호스트 해제 함수, `char*` 결과는 `host->alloc` 또는 `release`)
- [ ] 입력 포인터를 호출 뒤에 쓰지 않음
- [ ] `scaling_factor_mm`을 매 호출 읽음
- [ ] 전역 가변 상태 없음 / `thread_safe: true`면 잠금
- [ ] `parallel_for`의 `fn`이 스레드 안전하고 결과를 인덱스 순서로 합침
- [ ] 시간·난수·포인터·해시 순서에 의존하지 않음
- [ ] 긴 루프에서 `cancelled` 확인
- [ ] 사용자 문구는 영어 원문 + `i18n/ko.json`
- [ ] 꺼진 상태에서 상태 null / 0 / 띠 없음, `MP_GCODE_USED` 안 세움
- [ ] (프린터 모드) `connect` 전에 온 `panel_message`도 처리, `conn`으로 프린터 구분, `core.net` import가 null이어도 로드 성공

**빌드·패키지**
- [ ] `MagpieMod.cmake`의 `magpie_add_mod`로 빌드 (플래그를 덮어쓰지 않음)
- [ ] `bin/<플랫폼>/`에 라이브러리와 필요한 DLL만 (`.pdb` 등 없음)
- [ ] (Windows) 정적 런타임 또는 런타임 DLL 동봉
- [ ] 서명 후 폴더를 바꾸지 않음

**테스트**
- [ ] `--mod-test <id>`가 `PASSED`
- [ ] 켠 케이스와 끈 케이스(`not_contains: "; mods: <id>"`)
- [ ] 설치만 하고 끈 상태가 모드 없는 앱과 G-code 동일 (골든)

---

## 14. 문제 해결

로그는 `%APPDATA%\MagpieSlicer\log\`에 있고, 로드 실패는 `Mod not loaded: <id>: <이유>` 한 줄로 남습니다.

| 증상 / 메시지 | 원인 | 해결 |
|---|---|---|
| `unverified native code not accepted yet (...); start the application and accept its warning to load it` | 명령줄(`--mod-test`, `--slice`)에서 아직 허락하지 않은 인증 안 된 모드 | GUI를 한 번 열어 경고에서 "위험을 이해하고 로드" |
| `has native code but is not signed with an official or verified key; it is not loaded` | 이전 버전의 앱 | 앱 업데이트 (새 버전은 이 오류 대신 한 번 경고) |
| 시작할 때 "인증되지 않은 모드" 대화상자 | Gen 인증 서명이 없거나 서명 후 파일이 바뀜 | 의도한 것이면 "위험을 이해하고 로드". 파란색 태그를 원하면 0.3절대로 인증 요청 |
| 인증받은 모드가 빨간 태그로 바뀜 | 서명 후 파일이 추가되거나 바뀜, 또는 인증 키를 모르는 낡은 앱 | 받은 폴더 그대로 쓰기, 앱 업데이트 |
| 경고나 로그의 이유가 `it is signed with "...", which this application does not trust` | 앱이 모르는 키로 서명됨 (개발자 자신의 키 등). 설치 모드는 인증 안 됨으로 처리 | 파란색 태그가 필요하면 0.3절 |
| 이유가 `the files differ from what "..." signed` | 서명 후 파일이 바뀌거나 추가됨 (`.pdb`, 테스트 출력 등). 설치 모드는 인증 안 됨, 번들 모드는 로드 안 됨(`its signature does not verify: ...`) | 받은 폴더 그대로 쓰기, 개발자는 폴더 정리 후 다시 서명 받기 |
| `... is not among the checked files (the signature or the accepted unverified files)` | 서명(또는 허락) 확인 뒤 `bin/` 폴더에 파일이 생기거나 바뀜 | 앱을 다시 시작 (인증 안 된 모드면 다시 묻습니다) |
| `the folder is named "x" but the mod's id is "y"` | 폴더 이름 ≠ id | 맞추기 |
| `a mod with native code must be a package folder <id>/mod.json` | 바이너리 모드를 단일 `.json`으로 둠 | 패키지 폴더로 |
| `has no library for this platform (win-x64)` | `bin/win-x64/<binary>.dll` 없음 / 이름 불일치 | `binary`와 파일 이름, 플랫폼 폴더 확인 |
| `the library does not export magpie_mod_query, magpie_mod_init and magpie_mod_shutdown` | export 누락, C++ 이름 장식 | `MAGPIE_MOD_DEFINE` 사용 |
| `it was compiled with floating-point mode "unknown"` | GCC/Clang에서 템플릿 없이 빌드 | `magpie_add_mod` 사용 |
| `... floating-point mode "fast"` / `"contract"` | `/fp:fast`, `-ffast-math`, 상위 프로젝트 플래그 유입 | 템플릿 플래그를 덮지 않기 |
| `the library is version A but mod.json says B` | 버전 불일치 | 두 곳 같이 수정 |
| `it needs mod API 1.N; this application has 1.M` | `min_minor`가 앱보다 높음 | 앱 업데이트 또는 `min_minor` 낮추기 |
| `registered slot ... which its mod.json does not declare` | 코드와 매니페스트의 slot/version/impl 불일치 | 맞추기 |
| `mod.json declares slot ... but magpie_mod_init did not register it` | 등록 누락 또는 `register_slot` 실패를 무시 | 반환값 확인 |
| `slot X "impl" is already filled by mod 'other'` | 같은 impl 이름 | 고유한 impl 이름 |
| `magpie_mod_init failed (code N)` | init이 실패 반환 | 앞의 `host->log` 줄 확인 |
| `needs X, which is missing, older or circular` | `depends` 미충족 | 의존 모드 설치/버전 |
| `a bundled mod has this id and cannot be replaced` | 공식 모드 id 사용 | id 변경 |
| `only official mods may change "..._gcode"` | 이전 버전의 앱(G-code 설정 규칙이 공식 전용) | 앱 업데이트 |
| 로그 `not an official mod, its start G-code filter is not run` | 이전 버전의 앱(시작 G-code 필터가 공식 전용) | 앱 업데이트 |
| 모드는 로드됐는데 선택 슬롯이 안 쓰임 | `when` 없음, `when` 거짓, 또는 더 높은 priority 모드 | `when` 추가, 파라미터 켜기, priority 확인 |
| `mods 'a' and 'b' are both turned on for ... with priority N` | 같은 priority 두 모드가 동시에 참 | 하나 끄거나 priority 조정 |
| 묻는 슬롯이 조용히 호출되지 않음 | 테이블 `size`가 그 버전 구조체보다 작음 | 모든 항목을 채우고 `sizeof` 전체 |
| 값을 바꿔도 결과가 안 바뀜 (stale) | `invalidates` 누락 / 잘못된 `none` | 단계 추가 또는 목록 생략 |
| `mod 'x' ... crashed: ... It is disabled until the application restarts` | 모드 안에서 접근 위반 등 | 앱 재시작, 디버거로 원인 수정 |
| 슬라이스가 멈춘 채 응답 없음 | `thread_safe: false` 모드의 재진입 교착 **(추론)**, 또는 취소 미확인 | 5.5절, `cancelled` 확인 |
| `--mod-test`가 두 G-code 차이로 실패 | 비결정성 | 5.7절 |
| `--mod-test` G-code 검사 실패 (`XY off the bed` 등) | 베드 밖 이동, 온도 이상, 큰 E | 모드 출력 수정 |
| 번역이 안 나옴 | 원문 문자열 불일치, 카탈로그가 객체가 아님 | 원문을 정확히 같게, JSON 객체로 |
| `the library could not be loaded (Windows error 126)` | 의존 DLL을 `bin/` 또는 System32에서 못 찾음 | 정적 런타임, 의존 DLL 동봉 |
| Printer Agent 드롭다운에 `mod:<id>`가 없음 | `thread_safe: true`가 아님, 슬롯 버전·테이블 크기 부족, 또는 이전 버전의 앱 (12.4절 4번) | 매니페스트 확인, 앱 다시 시작 |
| 웹 패널이 "세션이 없다"고 함 | 12.4절 1번이 없는 이전 버전의 앱 | 앱 업데이트, 또는 Device 탭에서 한 번 연결 |
| `open_url`을 보냈는데 브라우저가 안 열림 | https가 아니거나, 호스트·포트가 `network` 목록에 없음 | `network.endpoints`에 추가 |
| `core.net` 호출이 모두 `MP_ERR_FAILED` | 사용자가 네트워크 접근에 동의하지 않음, 목록에 없는 주소, 또는 명령줄 실행(서비스 없음) | 매니페스트 `network` 확인, 프린터 연결 패널에서 동의 |

---

## 15. 참고 예제 (공식 모드)

| 모드 | 슬롯 | 볼 점 |
|---|---|---|
| `mods/half_layer` (101줄) | `layer.bands` | 가장 짧은 실제 모드. 입력 검증, 객체 범위 파라미터, `legacy_keys` |
| `mods/snapmaker_u1` (143줄) | `gcode.events` | 빈 콜백 채우기, `host->alloc` 문자열 반환, `host->warn`, 테스트 케이스 |
| `mods/support_extras` (117줄) | `support.style` | 상태 null로 "해당 없음" |
| `mods/layer_time_blend` | `gcode.time_plan` | 선언 `settings`, `visible`, 순수 계산 분리(`planner.cpp`)와 단위 테스트 |
| `mods/cura_support` | `support.areas` | `menu`, `export_service`, `core.geometry`, `i18n` |
| `mods/mixed_support` | `support.compose` | `choices_from`, compose 호스트 사용 |
| `mods/multi_nozzle_walls` | `wall.plan` | open에서 한 번 읽기, 다중 노즐 |
| `mods/low_temp_interface` | `gcode.events` | writer 출력 전반, 보조 타워, `printer` 범위 파라미터 |
| `mods/low_temp_calibration` | `gcode.events` | `gui.calibrations`, `calibration_value` |
| `mods/resin_support` | `support.contacts`, `object.z_offset` | 예외: 코어 라이브러리를 정적 링크하는 유일한 공식 모드 (제3자는 따라 하지 마세요) |
| `mods/test_mod` | 테스트 슬롯 `test.math` | 로더·크래시 가드·API/FP 거부 테스트 |
