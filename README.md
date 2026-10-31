# Magpie Slicer mods

Magpie Slicer용 모드를 만드는 데 필요한 것을 모은 저장소입니다.

- **[모드 개발자 가이드 (한국어)](docs/index.md)**: 툴체인, CMake 템플릿, hello 모드, 슬롯 레퍼런스, 매니페스트, 지켜야 할 규칙, 서명과 인증, 테스트, 체크리스트, 문제 해결
- **`sdk/`**: 모드가 쓰는 C 헤더(`include/`)와 CMake 템플릿(`MagpieMod.cmake`). 슬라이서 소스 없이 이것만으로 모드를 빌드할 수 있습니다.
- **`mods/<id>/`**: 앱에 번들되는 Magpie 공식 모드 10개의 소스. 실제 동작하는 예제로 보세요. (프린터 연결용 Anycubic 모드는 이 저장소에 없습니다.)
- **`index.json`**: 앱의 "모드 받기"가 읽는 모드 목록. `mods/<id>.json` 데이터 모드(과 `<id>.meta.json`)를 넣고 `python build_index.py --base-url https://raw.githubusercontent.com/Gen7920335/magpie-mods/main` 으로 다시 만듭니다.

| 모드 | 슬롯 | 내용 |
|---|---|---|
| `cura_support` | `support.areas` | CuraEngine 방식 서포트 |
| `mixed_support` | `support.compose` | 일반 + 트리 혼합 서포트 |
| `resin_support` | `support.contacts`, `object.z_offset` | 레진식 기둥 서포트 (PrusaSlicer SLA 코드) |
| `support_extras` | `support.style` | 서포트 인터페이스 패턴·벽 수 |
| `multi_nozzle_walls` | `wall.plan` | 다중 노즐 벽 (작은 노즐로 바깥 벽) |
| `half_layer` | `layer.bands` | 절반 높이 외벽과 서포트 |
| `layer_time_blend` | `gcode.time_plan` | 레이어 시간 균일화 대기 |
| `low_temp_interface` | `gcode.events` | 저온 서포트 접점 |
| `low_temp_calibration` | `gcode.events` | 저온 접점 캘리브레이션 (LESIC 링, 격자 보정) |
| `snapmaker_u1` | `gcode.events` | Snapmaker U1 시작 G-code 보호 |

## 빌드에 대해

- 제3자 모드는 `sdk/MagpieMod.cmake`의 `magpie_add_mod()`로 앱 소스 없이 빌드합니다. 가이드 8장의 hello 모드를 보세요.
- 이 저장소의 공식 모드는 Magpie Slicer 앱 소스 트리(`mods/` 폴더)에서 `magpie_official_mod()`로 빌드되도록 되어 있습니다. `resin_support`는 앱의 libslic3r를 정적 링크합니다.
- 서명 키, `SIGNATURE` 파일, 빌드 결과물은 이 저장소에 없습니다.

## 인증과 공식 편입

- 직접 만든 모드는 서명 없이도 배포할 수 있습니다. 사용자에게는 "인증 안 됨" 배지와 처음 한 번 경고가 뜹니다.
- 인증 배지를 받거나 공식 모드로 넣고 싶다면 **소스 상태로** 보내 주세요(컴파일한 DLL은 받지 않습니다). 자세한 절차는 가이드 0.3절과 0.4절에 있습니다.

## 라이선스

- `docs/`(가이드)와 `sdk/`(헤더, CMake 템플릿): **MIT** (`LICENSE`). SDK로 만드는 모드는 원하는 라이선스로 배포할 수 있습니다.
- `mods/`(공식 모드 소스): **GNU AGPL-3.0** (`mods/LICENSE`). 이 모드들은 OrcaSlicer 기반인 Magpie Slicer 코어에서 옮겨 왔고, 일부는 PrusaSlicer·CuraEngine 코드를 담고 있어 그 라이선스를 따릅니다. 이 소스를 고쳐 배포하면 그 소스도 AGPL-3.0으로 공개해야 합니다.
