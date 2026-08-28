# 조명 시스템 — 4종 라이트, 조명 모델 매크로, UE 스타일 감쇠

> 엔진 개발 WEEK07 주차(2025-10-16 ~ 10-23) 작업. 게임잼 두 달 전에 만든 이 조명 시스템이 이 게임의 라이팅을 그대로 담당합니다. 아래 파일들은 이후 주차에 팀원 작업이 겹겹이 쌓인 공동 작업물이며, 이 문서는 `git blame` 으로 본인 작성이 확인된 부분만 다룹니다 (파일별 지분은 [Contribution.md](Contribution.md) 참고). 특히 **그림자 매핑(PCF·VSM·CSM)과 타일 기반 라이트 컬링은 팀원 작업**입니다 — 본인이 만든 조명 함수 안에 이후 통합되었습니다.

## 문제

WEEK07 시작 시점의 코드베이스는 메시를 텍스처·버텍스 색으로만 그렸고 조명 계산이 없었습니다 (도입 커밋: `8dc75617` "조명 표현 가능"). 요구사항:

1. Directional / Point / Spot / Ambient 4종 라이트
2. 셰이더별로 조명 모델(Gouraud / Lambert / Phong)을 선택할 수 있을 것
3. 물리 기반 감쇠와 아티스트 제어형 감쇠를 모두 지원할 것
4. OBJ/MTL 머티리얼(Kd·Ka·Ks·Ns·Ke·Tr)을 조명 계산에 반영할 것

## 설계 1 — 공용 조명 라이브러리와 매크로 훅

조명 계산을 [`LightingCommon.hlsl`](../Mundi/Shaders/Common/LightingCommon.hlsl) 하나에 모으고, 이를 include 하는 셰이더가 **include 전에 정의하는 매크로**로 동작을 바꿉니다:

- `USE_BLINN_PHONG` — specular 를 Blinn-Phong(half-vector)과 전통 Phong(reflection vector) 중 선택 (커밋 `7ab0a01c`)
- `SPECULAR_COLOR` — specular 색 결정식을 셰이더가 주입. UberLit 은 `Material.SpecularColor` 를 넣어 금속 재질의 유색 specular 를 지원하고, 정의하지 않으면 흰색 기본값 (커밋 `96da95fc`)
- `LIGHTING_MODEL_GOURAUD / _LAMBERT / _PHONG` — 모델에 따라 specular 포함 여부·viewDir 필요 여부가 자동 결정되고, 통합 헬퍼 `CalculateAllLights()` 가 이를 따릅니다

기본 단위 함수(`CalculateAmbientLight` / `CalculateDiffuse` / `CalculateSpecular`) 위에 라이트 종류별 통합 함수(`CalculateDirectionalLight` / `CalculateSpotLight` / `CalculatePointLight`)를 쌓은 구조입니다.

[`UberLit.hlsl`](../Mundi/Shaders/Materials/UberLit.hlsl) 은 이 매크로들로 조명 모델 3종을 한 파일에서 컴파일하는 uber 셰이더입니다 — Gouraud 는 버텍스 셰이더에서, Lambert/Phong 은 픽셀 셰이더에서 조명을 계산합니다. 기존 StaticMeshShader 로직을 여기로 이관했습니다 (커밋 `46bc65fc`).

## 설계 2 — Unreal Engine 방식 감쇠 2모드

Point/Spot 라이트가 `bUseInverseSquareFalloff` 플래그로 감쇠 모델을 선택합니다 (커밋 `f3d4a394`):

- **Inverse Square Falloff** — 물리 기반 역제곱 감쇠에 `(1-(d/r)⁴)²` 윈도우 함수를 곱해 감쇠 반경 경계에서 부드럽게 0 에 도달
- **Exponent Falloff** — `(1-(d/r)²)^exponent`. 지수로 감쇠 곡선을 아티스트가 제어

Spot 원뿔 감쇠는 코사인 공간이 아니라 **각도 공간에서 보간**합니다 — `acos` 로 각도를 복원한 뒤 inner/outer cone 각도 사이를 `smoothstep` 으로 보간합니다. 코사인은 각도에 비선형이라 코사인 공간 보간은 감쇠가 한쪽으로 쏠리는데, 각도 공간에서는 원뿔 단면 기준으로 균등하게 떨어집니다.

색온도는 셰이더가 아니라 **C++ 라이트 컴포넌트에서 Intensity 와 함께 사전 곱셈**해 최종 색 하나만 셰이더에 넘깁니다 (커밋 `0faa756e`) — 셰이더의 per-pixel 비용과 파라미터 수를 줄입니다.

## 설계 3 — OBJ/MTL 머티리얼 반영

`FMaterial` 상수 버퍼 구조체가 MTL 의 Kd·Ka·Ks·Ns·Ke·Tr·illum 을 담습니다. Ambient 항은 MTL 표준(`La × Ka`)을 따르되, 내보내기 도구들이 Ka 를 (0,0,0) 이나 (1,1,1) 기본값으로 채워 오는 경우가 많아 **Ka 가 기본값이면 Kd 를 대신 사용**하는 하이브리드 규칙을 넣었습니다 (커밋 `d792282f`). Emissive(Ke)는 조명 계산 후 가산합니다.

## 설계 4 — 에디터 지원과 파생 작업

- **SpotLight 원뿔 시각화** ([`SpotLightComponent.cpp`](../Mundi/Source/Runtime/Engine/Components/SpotLightComponent.cpp), 커밋 `092e2d3b`) — Unreal Engine 스타일로 방사형 라인 24개 + 밑면 원 + 구면 보간(slerp) 아크를 그립니다. 아크의 가장 볼록한 지점이 정확히 감쇠 반경이 되도록 `ArcRadius = AttenuationRadius / cos(angle)` 로 보정합니다.
- **데칼 조명** ([`Decal.hlsl`](../Mundi/Shaders/Effects/Decal.hlsl), 커밋 `9a12b3a4`, `b44adc7f`) — 투사 데칼에도 같은 조명 모델을 동적으로 적용했습니다.
- **게임잼 추가분** — 식생(풀·나무) 렌더링을 위해 UberLit 에 alpha test 를 넣어 텍스처 알파 0.5 미만 픽셀을 `discard` 합니다 (커밋 `40c3e541`).

## 한계

- 포워드 렌더링 단일 패스 Blinn-Phong 입니다. PBR(에너지 보존, 러프니스 기반) 이 아닙니다.
- 이후 주차에 그림자(PCF·VSM·CSM)·타일 컬링 파라미터가 본인이 만든 조명 함수 시그니처에 계속 추가되면서 인자가 10개를 넘는 함수가 생겼습니다. 구조체로 묶는 정리를 하지 못했습니다.
- 조명 모델·그림자 기법·스키닝이 전부 전처리기 매크로 분기라 UberLit 의 분기 조합이 컴파일 시점에 폭발적으로 늘어납니다. variant 관리는 [셰이더 핫 리로드](Feature_AssetPipeline.md)의 variant map 이 담당하지만, 조합 수 자체를 줄이는 설계는 없습니다.
