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

### 코드로 보는 체인

체인의 시작 — 보스 빔 피격 시 Lua 게임플레이 코드가 호출합니다 ([`w14_Player.lua:783`](../Mundi/Data/Scripts/Player/w14_Player.lua#L783-L794)):

```lua
-- [함수 4-1] Boss Beam 피격 시 감전 효과
function PlayElectricHitEffect(duration)
    local cm = GetCameraManager()
    if not cm then return end

    -- Electric 포스트 프로세스 효과 시작
    -- StartElectric(duration, intensity, color, priority)
    cm:StartElectric(
        duration,            -- 지속 시간
        0.8,                 -- 효과 강도
        Color(1.0, 0.7, 0.3, 1)  -- 주황-노랑 (CoreBeam 색상)
    )
end
```

`StartElectric` 은 Sol2 오버로드로 바인딩해 Lua 쪽에서 뒤 인자를 생략하면 C++ 기본값이 쓰입니다 ([`LuaManager.cpp:1089`](../Mundi/Source/Runtime/Engine/Scripting/LuaManager.cpp#L1089-L1117)):

```cpp
// --- StartElectric (감전 효과) ---
"StartElectric", sol::overload(
    // (Full) 4개 인수
    [](APlayerCameraManager* Self, float InDuration, float Intensity, const FLinearColor& InColor, int32 InPriority)
    {
        if (Self) Self->StartElectric(InDuration, Intensity, InColor, InPriority);
    },
    // ... (3·2·1개 인수 오버로드 생략)
),

// --- ClearAllModifiers (모든 모디파이어 제거) ---
"ClearAllModifiers", [](APlayerCameraManager* Self)
{
    if (Self) Self->ClearAllModifiers();
},
```

모디파이어가 상태(경과 시간·지속 시간·파라미터)를 소유하고, 매 프레임 페이드 아웃 가중치를 계산해 payload 로 직렬화합니다 ([`CamMod_Electric.h:29`](../Mundi/Source/Runtime/Engine/GameFramework/Camera/CamMod_Electric.h#L29-L60)):

```cpp
virtual void CollectPostProcess(TArray<FPostProcessModifier>& Out) override
{
    if (!bEnabled) return;

    // 페이드 아웃 계산: Duration의 FadeOutRatio 지점부터 서서히 사라짐
    float fadeWeight = 1.0f;
    if (Duration > 0.0f)
    {
        float fadeStartTime = Duration * FadeOutRatio;
        if (Elapsed > fadeStartTime)
        {
            // fadeStartTime ~ Duration 구간에서 1.0 -> 0.0으로 페이드
            float fadeProgress = (Elapsed - fadeStartTime) / (Duration - fadeStartTime);
            fadeWeight = 1.0f - FMath::Clamp(fadeProgress, 0.0f, 1.0f);
        }
    }

    FPostProcessModifier M;
    M.Type = EPostProcessEffectType::Electric;
    M.Priority = Priority;
    M.bEnabled = true;
    M.Weight = Weight * fadeWeight;  // 페이드 아웃 적용
    M.SourceObject = this;

    M.Payload.Color = Color;
    // Params0: X=Intensity, Y=Time, Z=FlickerSpeed, W=BoltCount
    M.Payload.Params0 = FVector4(Intensity, ElapsedTime, FlickerSpeed, BoltCount);
    // Params1: X=ChromaticStrength
    M.Payload.Params1 = FVector4(ChromaticStrength, 0.0f, 0.0f, 0.0f);

    Out.Add(M);
}
```

렌더패스는 payload 를 상수 버퍼에 채우고 풀스크린 드로우 한 번을 실행할 뿐, 자체 상태가 없습니다 ([`ElectricPass.cpp:46`](../Mundi/Source/Runtime/Renderer/PostProcessing/ElectricPass.cpp#L46-L58)):

```cpp
// 5) 상수 버퍼 업데이트
FElectricBufferType ElectricConstant;
ElectricConstant.ElectricColor = M.Payload.Color;
ElectricConstant.Intensity = M.Payload.Params0.X;
ElectricConstant.Time = M.Payload.Params0.Y;
ElectricConstant.FlickerSpeed = M.Payload.Params0.Z;
ElectricConstant.Weight = M.Weight;
ElectricConstant.BoltCount = M.Payload.Params0.W;
ElectricConstant.ChromaticStrength = M.Payload.Params1.X;

RHIDevice->SetAndUpdateConstantBuffer(ElectricConstant);

// 6) Draw
RHIDevice->DrawFullScreenQuad();
```

관련 소스: [`Electric_PS.hlsl`](../Mundi/Shaders/PostProcess/Electric_PS.hlsl) · [`ElectricPass.cpp`](../Mundi/Source/Runtime/Renderer/PostProcessing/ElectricPass.cpp) · [`CamMod_Electric.h`](../Mundi/Source/Runtime/Engine/GameFramework/Camera/CamMod_Electric.h) · [`w14_Player.lua`](../Mundi/Data/Scripts/Player/w14_Player.lua)

## 셰이더 3종 — 텍스처 없는 프로시저럴 표현

공통 기반은 hash → value noise → FBM(Fractal Brownian Motion) 함수 계층이고, UV 는 `ViewportRect`/`ScreenSize` 로 뷰포트 정규화를 거쳐 에디터 멀티 뷰포트에서도 올바르게 나옵니다.

- **Fire** ([`Fire_PS.hlsl`](../Mundi/Shaders/PostProcess/Fire_PS.hlsl), 파이어볼 피격) — 화면 하단에서 위로 흐르는 FBM 을 높이에 따라 두 계층으로 블렌딩합니다: 하단은 부드러운 노이즈, 상단은 X축을 촘촘하게·Y축을 길게 샘플링한 세로로 긴 불꽃 혀 패턴. 마스크 강도에 따라 코어(흰색)→노랑→주황→빨강 4단 그라데이션을 적용하고, 주파수가 다른 sin 3개를 곱해 일렁임을 만듭니다.
- **Slime** ([`Slime_PS.hlsl`](../Mundi/Shaders/PostProcess/Slime_PS.hlsl), Boomer 자폭 피격) — hash 기반으로 배치한 방울 12개가 시간에 따라 흘러내리고(`frac` 래핑으로 화면 아래로 나가면 위에서 재시작), 흘러내린 거리만큼 늘어난 꼬리가 붙습니다. 방울 중심 방향으로 UV 를 밀어 씬을 굴절시키고, 두께 기반 색 변화·하이라이트·광택 노이즈로 점성 표현을 더했습니다.
- **Electric** ([`Electric_PS.hlsl`](../Mundi/Shaders/PostProcess/Electric_PS.hlsl), 보스 전기 빔 피격) — 세그먼트 기반 번개입니다. 시작점·방향 패턴 8종(가장자리 4방향·대각선·중앙 방사형·코너) 중 해시로 고르고, 8~14개 세그먼트를 지그재그로 꺾으며(세그먼트당 약 30~60°), 확률적으로 1~3차 가지를 분기합니다. 여기에 RGB 채널을 중심 기준 반대 방향으로 분리 샘플링하는 색수차, 화면 전체의 아크 노이즈, 가장자리 글로우를 합성합니다.

Electric 의 핵심 두 조각 — 번개 한 세그먼트는 점-선분 거리로 코어와 글로우를 만들고 ([`Electric_PS.hlsl:121`](../Mundi/Shaders/PostProcess/Electric_PS.hlsl#L121-L134)):

```hlsl
// 세그먼트 기반 번개 - 실제 번개처럼 꺾이는 효과
float lightningSegment(float2 uv, float2 start, float2 end, float thickness, float glowSize)
{
    float2 pa = uv - start;
    float2 ba = end - start;
    float h = saturate(dot(pa, ba) / dot(ba, ba));
    float d = length(pa - ba * h);

    // 코어 (밝은 중심)
    float core = smoothstep(thickness, thickness * 0.1, d);
    // 글로우 (주변 빛번짐)
    float glow = smoothstep(glowSize, 0.0, d) * 0.5;

    return core + glow;
}
```

세그먼트 루프가 지그재그 방향을 번갈아 꺾으며 끝으로 갈수록 가늘어지는 줄기를 만듭니다 ([`Electric_PS.hlsl:225`](../Mundi/Shaders/PostProcess/Electric_PS.hlsl#L225-L254), 가지 분기 부분은 생략):

```hlsl
for (int seg = 0; seg < numSegments; seg++)
{
    float segSeed = seed + float(seg) * 3.7;
    float2 perpDir = float2(-mainDir.y, mainDir.x);

    // 극적인 지그재그 꺾임 - 매 세그먼트마다 방향 전환
    // bendAmount를 0.6~1.2 사이로 크게 (약 30~60도 꺾임)
    float baseBend = 0.6 + hash(segSeed * 1.5) * 0.6;

    // 지그재그: 번갈아가며 좌우로 꺾임
    float bendAmount = baseBend * zigzagDir;

    // 추가 랜덤성 (완전히 규칙적이지 않게)
    bendAmount += sharpNoise(float2(segSeed, t * 3.0), t) * 0.3;

    // 가끔 같은 방향으로 두 번 꺾이기도 함 (더 자연스럽게)
    if (hash(segSeed * 1.7) > 0.8)
        zigzagDir = zigzagDir; // 방향 유지
    else
        zigzagDir = -zigzagDir; // 방향 전환

    float2 segDir = normalize(mainDir + perpDir * bendAmount);
    float2 nextPos = currentPos + segDir * segmentLength;

    // 세그먼트별 두께 변화 (끝으로 갈수록 가늘게)
    float progress = float(seg) / float(numSegments);
    float thickness = 0.007 * (1.0 - progress * 0.6);
    float glowSize = 0.03 * (1.0 - progress * 0.4);

    lightning += lightningSegment(uv, currentPos, nextPos, thickness, glowSize);

    // ... (확률적 1~3차 가지 분기: L256-L295)

    currentPos = nextPos;
}
```

### 사례: 감전 이펙트의 눈 피로 조정

초기 버전은 번쩍임이 step 함수 기반 온/오프여서 플레이 테스트에서 눈이 아프다는 피드백을 받았습니다. 커밋 `c6dea0bd` 에서 조정했습니다:

- 플리커를 `smoothstep` 전환으로 바꾸고 밝기를 0.2~0.8 범위로 클램프 (완전 소등/최대 밝기 제거)
- 랜덤 스파이크 항 제거, 전화면 플래시 기여도를 0.08 로 축소
- 노출 과다(전체 곱셈 부스트) 항 제거

그 커밋의 `getFlicker` 와 `mainPS` 핵심 diff:

```diff
-// 불규칙한 번쩍임 생성 - 더 빠르고 급격하게
+// 불규칙한 번쩍임 생성 - 부드럽게 조정
 float getFlicker(float t, float seed)
 {
-    // 빠른 불규칙 펄스
-    float f1 = sin(t * FlickerSpeed * 2.0 + seed * 100.0);
-    float f2 = sin(t * FlickerSpeed * 5.7 + seed * 200.0);
-    float f3 = sin(t * FlickerSpeed * 11.3 + seed * 300.0);
-    float f4 = sin(t * FlickerSpeed * 17.9 + seed * 400.0);
+    // 느린 불규칙 펄스 (속도 감소)
+    float f1 = sin(t * FlickerSpeed * 0.5 + seed * 100.0);
+    float f2 = sin(t * FlickerSpeed * 1.2 + seed * 200.0);
+    float f3 = sin(t * FlickerSpeed * 2.1 + seed * 300.0);

-    float combined = f1 * 0.3 + f2 * 0.3 + f3 * 0.25 + f4 * 0.15;
+    float combined = f1 * 0.4 + f2 * 0.35 + f3 * 0.25;

-    // 더 급격한 온/오프
-    float flicker = smoothstep(0.1, 0.3, combined);
+    // 부드러운 전환 (급격한 온/오프 제거)
+    float flicker = smoothstep(-0.2, 0.5, combined);

-    // 랜덤 스파이크 추가
-    float spike = step(0.92, hash(floor(t * FlickerSpeed * 3.0) + seed));
-
-    return saturate(flicker + spike * 0.5);
+    // 랜덤 스파이크 제거 (눈 피로 방지)
+    return saturate(flicker * 0.6 + 0.2); // 항상 최소 0.2, 최대 0.8 정도
 }

@@ mainPS 합성부 @@
-    // 4. 전체 화면 플래시 (급격한 번쩍임)
-    float flashIntensity = mainFlicker * effectStrength * 0.25;
-    flashIntensity += step(0.95, mainFlicker) * effectStrength * 0.3; // 강한 플래시
+    // 4. 전체 화면 플래시 (크게 감소 - 눈 피로 방지)
+    float flashIntensity = mainFlicker * effectStrength * 0.08;

-    // 6. 약간의 노출 과다 효과 (번쩍일 때)
-    finalColor *= 1.0 + mainFlicker * effectStrength * 0.2;
+    // 6. 노출 과다 효과 제거 (눈 피로의 주요 원인)
+    // finalColor *= 1.0 + mainFlicker * effectStrength * 0.2;
```

## 한계

- 이펙트 3종의 파이프라인 코드(렌더패스·모디파이어·상수 버퍼·바인딩)가 거의 같은 구조의 복사-변형입니다. 공통 베이스로 묶지 않았습니다 — 게임잼 속도 우선의 대가입니다.
- 색·강도 등 파라미터가 셰이더/모디파이어 기본값과 Lua 호출 인자에 하드코딩돼 있습니다. 프리셋 데이터화가 없습니다.
- Electric 은 픽셀마다 번개 12줄기 × 세그먼트 루프를 전부 순회합니다. 프로파일링 없이 화질 우선으로 출시했습니다.
- 이펙트 2종 이상이 동시에 걸리면 각 패스가 순차로 씬 컬러에 가산될 뿐, 합성 규칙(우선순위별 감쇠 등)은 없습니다.
