# 피격 포스트프로세스 3종 — Fire · Slime · Electric

> 게임잼 주차(2025-12-05 ~ 12-12) 작업. 이 문서가 다루는 셰이더·렌더패스·카메라 모디파이어 코드는 전부 본인 단독 작성입니다 (파일별 지분은 [Contribution.md](Contribution.md) 참고). 단, 토대가 된 카메라 모디파이어 스택(`CameraModifierBase`, `APlayerCameraManager` 골격)과 포스트프로세스 파이프라인 골격은 팀원 구현이며, 본인은 그 위에 이펙트 3종을 추가했습니다.

## 문제

몬스터 공격이 파이어볼·자폭(Boomer)·보스 전기 빔으로 나뉘는데, 피격 피드백은 공용 비네트 하나뿐이었습니다. 요구사항:

1. 어떤 공격에 맞았는지 화면만 보고 구분되게 — 공격 종류별로 다른 전화면 이펙트
2. 텍스처 에셋 없이 셰이더 연산만으로 표현 (에셋 준비 없이 즉시 수치 조정·반복 가능)
3. 지속 시간이 끝나면 뚝 끊기지 않고 자연스럽게 사라질 것
4. 게임 재시작 시 이펙트가 화면에 남지 않을 것

## 설계 — Lua 게임플레이 이벤트에서 셰이더까지 수직 연결

이펙트 1종마다 같은 구조를 관통합니다. Electric 기준:

```
w14_Player.lua  OnBeginOverlap(보스 빔 피격)
  └─ cm:StartElectric(0.8, 0.8, Color(1.0, 0.7, 0.3, 1))     ← Sol2 오버로드 바인딩 (LuaManager.cpp)
       └─ APlayerCameraManager::StartElectric()
            └─ ActiveModifiers.Add(new UCamMod_Electric)       ← 기존 모디파이어 스택(팀원 구현)에 추가
                 └─ 매 프레임 CollectPostProcess()
                      └─ FPostProcessModifier { Type, Weight, Payload(Color, Params0/1) }
                           └─ FSceneRenderer::RenderPostProcessingPasses()
                                └─ FElectricPass::Execute()    ← FElectricBufferType(b2) 갱신, 풀스크린 드로우
                                     └─ Electric_PS.hlsl
```

- **모디파이어가 상태를 소유합니다.** 지속 시간(`Duration`), 경과 시간, 색·강도 파라미터를 모디파이어가 들고, 매 프레임 `FPostProcessModifier` payload 로 직렬화해 렌더러에 넘깁니다. 렌더패스는 payload 를 상수 버퍼에 채우고 그리기만 하는 무상태(stateless) 코드입니다.
- **페이드 아웃** — `Duration` 의 일정 비율(`FadeOutRatio`, Fire 0.4 / Electric 0.6) 지점부터 `Weight` 를 1→0 으로 선형 감소시키고, 셰이더는 모든 효과 항에 `Weight` 를 곱합니다. 효과가 끝날 때 잔상 없이 사라집니다 (요구사항 3).
- **재시작 잔류 제거** — `ClearAllModifiers()` 를 Lua 에 노출하고 재시작 시 호출해 스택을 비웁니다 (커밋 `3820a7d7`, 요구사항 4).
- 새 이펙트 1종을 추가하는 데 필요한 것: HLSL 1개 + 렌더패스 1쌍 + 모디파이어 1개 + 상수 버퍼 구조체 + `Start*` 함수와 Lua 바인딩. Fire → Slime → Electric 을 같은 패턴 반복으로 마감 전날 저녁부터 약 10시간 사이에 순차 추가했습니다 (커밋 `6c2caecd` → `7713bca6` → `f27277e7`).

관련 소스: [`Electric_PS.hlsl`](../Mundi/Shaders/PostProcess/Electric_PS.hlsl) · [`ElectricPass.cpp`](../Mundi/Source/Runtime/Renderer/PostProcessing/ElectricPass.cpp) · [`CamMod_Electric.h`](../Mundi/Source/Runtime/Engine/GameFramework/Camera/CamMod_Electric.h) · [`w14_Player.lua`](../Mundi/Data/Scripts/Player/w14_Player.lua)

## 셰이더 3종 — 텍스처 없는 프로시저럴 표현

공통 기반은 hash → value noise → FBM(Fractal Brownian Motion) 함수 계층이고, UV 는 `ViewportRect`/`ScreenSize` 로 뷰포트 정규화를 거쳐 에디터 멀티 뷰포트에서도 올바르게 나옵니다.

- **Fire** ([`Fire_PS.hlsl`](../Mundi/Shaders/PostProcess/Fire_PS.hlsl), 파이어볼 피격) — 화면 하단에서 위로 흐르는 FBM 을 높이에 따라 두 계층으로 블렌딩합니다: 하단은 부드러운 노이즈, 상단은 X축을 촘촘하게·Y축을 길게 샘플링한 세로로 긴 불꽃 혀 패턴. 마스크 강도에 따라 코어(흰색)→노랑→주황→빨강 4단 그라데이션을 적용하고, 주파수가 다른 sin 3개를 곱해 일렁임을 만듭니다.
- **Slime** ([`Slime_PS.hlsl`](../Mundi/Shaders/PostProcess/Slime_PS.hlsl), Boomer 자폭 피격) — hash 기반으로 배치한 방울 12개가 시간에 따라 흘러내리고(`frac` 래핑으로 화면 아래로 나가면 위에서 재시작), 흘러내린 거리만큼 늘어난 꼬리가 붙습니다. 방울 중심 방향으로 UV 를 밀어 씬을 굴절시키고, 두께 기반 색 변화·하이라이트·광택 노이즈로 점성 표현을 더했습니다.
- **Electric** ([`Electric_PS.hlsl`](../Mundi/Shaders/PostProcess/Electric_PS.hlsl), 보스 전기 빔 피격) — 세그먼트 기반 번개입니다. 시작점·방향 패턴 8종(가장자리 4방향·대각선·중앙 방사형·코너) 중 해시로 고르고, 8~14개 세그먼트를 지그재그로 꺾으며(세그먼트당 약 30~60°), 확률적으로 1~3차 가지를 분기합니다. 여기에 RGB 채널을 중심 기준 반대 방향으로 분리 샘플링하는 색수차, 화면 전체의 아크 노이즈, 가장자리 글로우를 합성합니다.

### 사례: 감전 이펙트의 눈 피로 조정

초기 버전은 번쩍임이 step 함수 기반 온/오프여서 플레이 테스트에서 눈이 아프다는 피드백을 받았습니다. 커밋 `c6dea0bd` 에서 조정했습니다:

- 플리커를 `smoothstep` 전환으로 바꾸고 밝기를 0.2~0.8 범위로 클램프 (완전 소등/최대 밝기 제거)
- 랜덤 스파이크 항 제거, 전화면 플래시 기여도를 0.08 로 축소
- 노출 과다(전체 곱셈 부스트) 항 제거

## 한계

- 이펙트 3종의 파이프라인 코드(렌더패스·모디파이어·상수 버퍼·바인딩)가 거의 같은 구조의 복사-변형입니다. 공통 베이스로 묶지 않았습니다 — 게임잼 속도 우선의 대가입니다.
- 색·강도 등 파라미터가 셰이더/모디파이어 기본값과 Lua 호출 인자에 하드코딩돼 있습니다. 프리셋 데이터화가 없습니다.
- Electric 은 픽셀마다 번개 12줄기 × 세그먼트 루프를 전부 순회합니다. 프로파일링 없이 화질 우선으로 출시했습니다.
- 이펙트 2종 이상이 동시에 걸리면 각 패스가 순차로 씬 컬러에 가산될 뿐, 합성 규칙(우선순위별 감쇠 등)은 없습니다.
