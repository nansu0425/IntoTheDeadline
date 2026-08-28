# 기여 통계

이 저장소는 여러 주차 팀의 작업이 누적된 공동 작업물이고, 본인 코드는 전체의 일부입니다. 무엇이 본인 작업이고 무엇이 아닌지를 정량적으로 밝힙니다.

## 산정 방법

- `git blame -w --line-porcelain` 으로 파일의 각 줄을 마지막으로 수정한 작성자를 집계했습니다. 기준 커밋은 `0b6a0791` (main HEAD) 입니다.
- 본인 식별은 author 이메일 `nansu0425@gmail.com` 기준입니다 (계정명 `nansu0425` / `NanSu` 두 가지로 커밋됨).
- 표의 수치는 **현재 코드에 남아 있는 본인 작성 줄 수 / 파일 전체 줄 수**입니다. 커밋 수를 세는 방식과 달리, 본인이 작성했더라도 이후 팀원이 고쳐 쓴 줄은 본인 지분에서 빠집니다.
- 아래 기술 문서 3편은 이 blame 결과로 본인 작성 구간을 검증한 뒤, 그 구간에 해당하는 내용만 서술했습니다.

## 전체 지분

추적되는 텍스트 파일 전체(빌드 스크립트·문서 포함) 기준:

| 구분 | 남은 줄 | 비율 |
|---|---:|---:|
| 전체 | 4,911 / 149,804 | 3.3% |
| HLSL 셰이더 | 1,746 / 5,416 | 32.2% |
| C++ | 2,455 / 117,767 | 2.1% |
| Lua | 191 / 11,080 | 1.7% |

## 게임잼 주차 — [피격 포스트프로세스](Feature_HitPostProcess.md)

2025-12-05 ~ 12-12, 팀 4명. 본인 커밋 49건 / 팀 322건.

이펙트 3종(Fire·Slime·Electric)의 셰이더·렌더패스·카메라 모디파이어는 전부 본인 단독 작성입니다 — 합계 1,241 / 1,241줄 (100%).

| 파일 | 남은 줄 |
|---|---:|
| `Mundi/Shaders/PostProcess/Electric_PS.hlsl` | 414 / 414 (100%) |
| `Mundi/Shaders/PostProcess/Slime_PS.hlsl` | 241 / 241 (100%) |
| `Mundi/Shaders/PostProcess/Fire_PS.hlsl` | 191 / 191 (100%) |
| `.../Renderer/PostProcessing/{Electric,Slime,Fire}Pass.cpp` + `.h` | 209 / 209 (100%) |
| `.../GameFramework/Camera/CamMod_{Electric,Slime,Fire}.h` + `.cpp` | 186 / 186 (100%) |

기존 시스템에 연결한 통합 지점 (파일 자체는 팀원 작성, 본인은 일부 추가):

| 파일 | 남은 줄 |
|---|---:|
| `.../Engine/GameFramework/PlayerCameraManager.cpp` (`Start*` 함수들) | 55 / 460 (12%) |
| `.../Engine/Scripting/` 디렉터리 (Lua 바인딩 추가분) | 207 / 3,387 (6%) |
| `Mundi/Data/Scripts/Player/w14_Player.lua` (피격 트리거) | 87 / 802 (11%) |
| `Mundi/Data/Scripts/Player/w14_PlayerADS.lua` (Red Dot Sight / ADS) | 94 / 273 (34%) |

## 엔진 개발 WEEK07 주차 — [조명 시스템](Feature_Lighting.md) · [에셋 파이프라인](Feature_AssetPipeline.md)

2025-10-16 ~ 10-23, 팀 4명 (게임잼과 다른 팀 구성). 본인 커밋 133건. 이 주차에 만든 조명·에셋 인프라가 이 게임의 코드베이스에 남아 라이팅과 텍스처 로딩을 담당합니다.

**라이팅 / 셰이딩** — 합계 1,191 / 2,877줄 (41%). 이후 주차에 팀원의 그림자 매핑(PCF·VSM·CSM)·타일 라이트 컬링 작업이 같은 파일에 쌓여 지분이 희석됐습니다.

| 파일 | 남은 줄 |
|---|---:|
| `Mundi/Shaders/Common/LightingCommon.hlsl` | 334 / 932 (36%) |
| `Mundi/Shaders/Materials/UberLit.hlsl` | 313 / 686 (46%) |
| `.../Engine/Components/SpotLightComponent.cpp` | 377 / 806 (47%) |
| `.../Engine/Components/SpotLightComponent.h` | 33 / 104 (32%) |
| `Mundi/Shaders/Common/LightStructures.hlsl` | 29 / 78 (37%) |
| `Mundi/Shaders/Effects/Decal.hlsl` | 105 / 271 (39%) |

**에셋 · 텍스처 파이프라인** — 합계 641 / 1,394줄 (46%)

| 파일 | 남은 줄 |
|---|---:|
| `.../AssetManagement/TextureConverter.cpp` | 240 / 283 (85%) |
| `.../AssetManagement/TextureConverter.h` | 87 / 87 (100%) |
| `.../Renderer/Shader.cpp` (핫 리로드) | 212 / 696 (30%) |
| `.../Renderer/Shader.h` | 15 / 151 (10%) |
| `.../Core/Misc/PathUtils.h` (인코딩 유틸) | 87 / 177 (49%) |

## 팀원이 만든 것

혼동을 막기 위해 **본인이 만들지 않은 주요 기능**을 밝힙니다. 수치는 같은 방법으로 측정한 본인 지분입니다.

| 기능 | 본인 지분 |
|---|---:|
| Depth of Field (셰이더 5개) | 14 / 579 (2%) |
| 카메라 쉐이크 (`CamMod_Shake`) | 0 / 235 (0%) |
| 카메라 모디파이어 스택 골격 (`CameraModifierBase`) | 0 / 34 (0%) |
| 파티클 시스템 (`Engine/Particles/`) | 0 / 7,736 (0%) |
| PhysX 물리 · 래그돌 (`Engine/Physics/`) | 0 / 3,632 (0%) |
| 스켈레탈 애니메이션 (`Engine/Animation/`) | 0 / 3,085 (0%) |
| BVH · 공간 자료구조 (`Engine/Spatial/`) | 0 / 2,250 (0%) |
| 리플렉션 · Lua 바인딩 코드 생성기 (`BuildTools/CodeGenerator/`) | 0 / 3,226 (0%) |

- DoF 는 팀원이 구현했고, 본인은 게임잼 때 ADS 전환에 연동만 했습니다 (커밋 `8d28652b`).
- 카메라 쉐이크는 본인이 다른 주차(WEEK09+)에 다른 코드베이스에서 만든 적이 있으나, **이 게임의 것은 팀원 구현**입니다.
- `Docs/` 의 `DOF_구현_원리.md` · `PROJECT_DOCUMENTATION.md` · `DEBUG_REPORT_PhysX_Crash.md` 와 `Mundi/Docs/` 의 문서들은 팀원이 작성했습니다. 본인 작성 문서는 `Feature_*.md` 3편과 이 문서입니다.

> 이 교육과정은 매 주차 팀과 코드베이스가 바뀌었습니다. 본인이 어느 주차에 만든 기능이라도, 그 주차 코드베이스가 이 게임의 조상이 아니면 이 저장소에 들어있지 않습니다.

## 커밋 이력에 대한 주의사항

- 이 저장소의 이력은 게임잼 이전 주차까지 이어지지만, 엔진 개발 시작(2025-09-02)부터의 전체 이력은 **아닙니다**. 2025-09-26에 당시 팀이 저장소를 새로 만들면서 그때까지의 코드베이스 433개 파일을 한 커밋으로 임포트했고, 그 이전 이력은 여기에 없습니다.
- 따라서 이력 구간은 **WEEK05(2025-09-26) ~ WEEK14(게임잼)** 이며, 매 주차 팀이 새로 짜였으므로 **기여자 27명은 게임잼 팀 4명이 아니라 이 구간 여러 주차 팀의 누적**입니다: `git shortlog -sne`
- 원본 저장소의 커밋 히스토리와 기여자 정보는 그대로 보존했습니다 — 2,547 커밋. 에셋 필터링으로 내용이 비게 된 커밋도 메시지·작성자·날짜를 남겨두었습니다.
- 게임잼 종료 후의 커밋은 저장소 공개 준비(빌드 환경 구성, `.gitignore`, README·문서 재구성) 목적입니다.
