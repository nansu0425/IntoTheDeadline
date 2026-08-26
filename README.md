# INTO THE DEADLINE

**Mundi 엔진**(팀 자체 제작 DirectX 11 엔진) 위에서 만든 **FPS 좀비 서바이벌 게임**입니다.

마감에 쫓기는 직장인이 밀려드는 좀비 떼를 소총으로 상대합니다. 좀비는 기본형·추격형·자폭형·헐크형 4종에 보스가 있고, 피격 종류에 따라 화면에 화상·점액·감전 이펙트가 걸립니다.

<!-- TODO: 플레이 영상 / GIF -->
<!-- TODO: 스크린샷 -->

- **기간** — 2025-12-05 ~ 12-12 (8일), 게임잼 전 기간
- **과정** — KRAFTON 정글 게임테크랩 2기 WEEK14 (최종 주차)
- **팀** — 4명 (본인 포함)
- **본인 커밋** — 게임잼 49건 (팀 322건 중)

> 이 저장소는 **게임잼 마감 시점(WEEK14)의 스냅샷**입니다.

> **커밋 이력에 대해** — 이 저장소의 이력은 게임잼 이전 주차까지 이어지지만, 엔진 개발 시작(2025-09-02)부터의 전체 이력은 **아닙니다**. 2025-09-26에 당시 팀이 저장소를 새로 만들면서 그때까지의 코드베이스 433개 파일을 한 커밋으로 임포트했고, 그 이전 이력은 여기에 없습니다.
>
> 따라서 이력 구간은 **WEEK05(2025-09-26) ~ WEEK14**이며, 매 주차 팀이 새로 짜였으므로 **기여자 27명은 게임잼 팀이 아니라 이 구간 여러 주차 팀의 누적**입니다.

---

## ⚠️ 이 저장소에 대해

**소스 코드 공개용 저장소입니다. 빌드되지 않습니다.**

원본 팀 저장소에서 **코드만 추출**했습니다. 아래는 라이선스상 재배포할 수 없어 전부 제외했습니다.

| 제외 | 이유 |
|---|---|
| 캐릭터 · 애니메이션 (54개) | Adobe Mixamo — 프로젝트 내 사용은 허용되나 에셋 재배포 금지 |
| Autodesk FBX SDK | SDK 재배포 금지 (Autodesk에서 직접 받아야 함) |
| 맑은 고딕 폰트 | © Microsoft, Windows 번들 폰트 |
| 오디오 33개 | 출처 불명 |
| 모델 · 텍스처 | Quixel Megascans, Sketchfab(크레딧 유실), 상용 VFX 시트 등 |
| 서드파티 라이브러리 | PhysX, NvCloth, DirectXTK/Tex, Dear ImGui, Lua, Sol2, nlohmann — 각 upstream에서 |

**원본 저장소의 커밋 히스토리와 기여자 정보는 그대로 보존했습니다** — 2,544 커밋, 27명 전원. 필터링으로 내용이 비게 된 커밋도 메시지·작성자·날짜를 남겨두었습니다.

---

## 내가 만든 것

수치는 모두 `git blame -w` 기준 **현재 코드에 남아 있는 줄 수 / 파일 전체 줄 수**입니다. 커밋만 세는 것과 달리, 이후 팀원이 고쳐 쓴 부분은 빠집니다.

### 1. 게임잼 주간 — 피격 포스트프로세스 (12-05 ~ 12-12, 본인 49 커밋 / 팀 4명)

**포스트프로세스 / 카메라 1,258 / 1,558줄 (80%)**

몬스터 공격 종류별로 화면 전체에 거는 피격 이펙트 3종을 **셰이더 → 렌더패스 → 카메라 모디파이어**까지 수직으로 구현했습니다.

| 파일 | 남은 줄 |
|---|---:|
| `Mundi/Shaders/PostProcess/Electric_PS.hlsl` | 414 / 414 (100%) |
| `Mundi/Shaders/PostProcess/Slime_PS.hlsl` | 241 / 241 (100%) |
| `Mundi/Shaders/PostProcess/Fire_PS.hlsl` | 191 / 191 (100%) |
| `.../Renderer/PostProcessing/ElectricPass.cpp` | 62 / 62 (100%) |
| `.../Renderer/PostProcessing/SlimePass.cpp` | 62 / 62 (100%) |
| `.../Renderer/PostProcessing/FirePass.cpp` | 61 / 61 (100%) |
| `.../GameFramework/Camera/CamMod_Electric.h` | 61 / 61 (100%) |
| `.../GameFramework/Camera/CamMod_Slime.h` | 60 / 60 (100%) |
| `.../GameFramework/Camera/CamMod_Fire.h` | 57 / 57 (100%) |

- **Fire** — 파이어볼 피격 시 화면 가장자리 화염 왜곡
- **Slime** — 자폭 좀비(Boomer) 피격 시 점액 번짐
- **Electric** — 보스 전기 공격 피격 시 감전. 눈 피로를 고려해 강도를 조정했습니다

그 밖에 **Red Dot Sight / ADS 전환**(조준 시 총기를 DoF로 블러 처리, 크로스헤어 숨김), **UberLit Alpha Test/Discard**(풀·나무 식생 렌더링)를 작업했습니다.

### 2. 이 게임에 쓰인 엔진 기능 — WEEK07 조명·셰이더 인프라 (10-16 ~ 10-23, 본인 133 커밋 / 팀 4명)

게임잼 두 달 전에 만든 조명 시스템이 이 게임의 라이팅을 그대로 담당합니다.

**라이팅 / 셰이딩 1,357 / 4,221줄 (32%)**

| 파일 | 남은 줄 |
|---|---:|
| `Mundi/Shaders/Common/LightingCommon.hlsl` | 334 / 932 (35%) |
| `Mundi/Shaders/Materials/UberLit.hlsl` | 313 / 686 (45%) |
| `.../Engine/Components/SpotLightComponent.cpp` | 377 / 806 (46%) |
| `Mundi/Shaders/Common/LightStructures.hlsl` | 29 / 78 (37%) |
| `Mundi/Shaders/Effects/Decal.hlsl` | 105 / 271 (38%) |

4종 라이트(Directional / Point / Spot / Ambient), Blinn-Phong과 전통 Phong을 셰이더 매크로로 전환, 색온도 기반 광원 색, 감쇠 모델.

**에셋 · 텍스처 파이프라인 725 / 3,099줄 (23%)**

| 파일 | 남은 줄 |
|---|---:|
| `.../AssetManagement/TextureConverter.cpp` | 240 / 283 (84%) |
| `.../AssetManagement/TextureConverter.h` | 87 / 87 (100%) |
| `.../Renderer/Shader.cpp` | 212 / 696 (30%) |
| `.../Core/Misc/PathUtils.h` | 87 / 177 (49%) |

DirectXTex 기반 **DDS 자동 베이킹·캐싱**, **셰이더 핫 리로드**(include 체인을 따라가 의존 셰이더까지 리로드), 한글 경로 호환 처리.

---

## 팀원이 만든 것

이 저장소의 대부분은 제 코드가 아닙니다(전체의 3.1%). 혼동을 막기 위해 **제가 만들지 않은 주요 기능**을 밝힙니다.

| 기능 | 제 지분 |
|---|---:|
| Depth of Field (5개 셰이더) | 14 / 579 (2%) |
| 카메라 쉐이크 (`CamMod_Shake`) | 0 / 235 (0%) |
| 파티클 시스템 | 0% |
| PhysX 물리 · 래그돌 | 0% |
| BVH · 스켈레탈 애니메이션 | 0% |
| Lua 바인딩 자동화 | 398 / 3,084 (12%) |

DoF는 팀원이 구현했고, 저는 게임잼 때 **ADS 전환에 연동**만 했습니다. 카메라 쉐이크는 제가 다른 주차(WEEK09+)에 다른 코드베이스에서 만든 적이 있으나, **이 게임의 것은 팀원 구현**입니다.

> 이 과정은 매 주차 팀과 코드베이스가 바뀌었습니다. 제가 어느 주차에 만든 기능이라도, 그 주차 코드베이스가 이 게임의 조상이 아니면 여기 들어있지 않습니다.

## 전체 지분

| 구분 | 남은 줄 | 비율 |
|---|---:|---:|
| 전체 | 4,392 / 141,958 | 3.1% |
| HLSL 셰이더 | 1,746 / 5,416 | 32.2% |
| C++ | 2,455 / 117,767 | 2.1% |

---

## 원본 · 기여자

- 팀이 작성한 기존 최상위 README(엔진 좌표계 규약)는 [`Mundi/Docs/Mundi_CoordinateSystem.md`](Mundi/Docs/Mundi_CoordinateSystem.md) 로 옮겼습니다.
- 원본 저장소 — `nansu0425/GameTechLab-WEEK14` (비공개 전환)
- 여러 주차 팀 기여자 27명의 커밋이 그대로 보존되어 있습니다: `git shortlog -sne`
- 주차별 팀·담당 영역은 [`nansu0425/KRAFTON-GameTechLab-Engine`](https://github.com/nansu0425/KRAFTON-GameTechLab-Engine)의 주차별 작업 문서를 참고하세요.
- KRAFTON 정글 게임테크랩 2기 교육과정 산출물이며, **포트폴리오 목적으로 코드만 공개**합니다.
- 별도 라이선스를 두지 않았습니다. 공동 저작물이므로 코드 재사용을 원하시면 문의해 주세요.
