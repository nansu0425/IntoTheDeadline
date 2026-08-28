# 텍스처 DDS 베이킹·캐싱과 셰이더 핫 리로드

> 엔진 개발 WEEK07 주차(2025-10-16 ~ 10-23) 작업. 이 문서는 `git blame` 으로 본인 작성이 확인된 부분만 다룹니다 (파일별 지분은 [Contribution.md](Contribution.md) 참고). `TextureConverter` 의 FBX 임베디드 텍스처(.fbm) 캐시 특례와 `PathUtils.h` 의 캐시 경로 매핑 함수는 팀원이 이후 추가·재작성한 부분이라 여기서 다루지 않습니다.

## 문제

1. 텍스처를 매 실행마다 PNG/JPG 원본에서 비압축 포맷으로 로드해 GPU 에 올리고 있었습니다 — 로딩이 느리고 VRAM 을 낭비합니다
2. 셰이더를 한 줄 고칠 때마다 엔진을 재시작해야 했습니다
3. 한글이 포함된 에셋 경로에서 크래시가 났습니다 (Windows API 는 UTF-16, 엔진 내부 문자열은 UTF-8)

## 설계 1 — DDS 자동 베이킹·캐싱

[`TextureConverter.cpp`](../Mundi/Source/Runtime/AssetManagement/TextureConverter.cpp) (도입 커밋 `4cbbf41b`)

- **변환** — DirectXTex 로 원본(PNG·JPG·BMP 는 WIC, TGA·HDR 은 전용 로더)을 읽어 블록 압축합니다. 포맷은 알파·sRGB 여부에 따라 BC1/BC3(±sRGB) 자동 선택, `TEX_COMPRESS_PARALLEL` 멀티스레드 압축 + 디더링, 밉맵 체인 생성. 블록 압축은 4픽셀 정렬이 필요하므로 크기가 안 맞으면 리사이즈 후 압축합니다.
- **캐싱** — 변환 결과를 `Data/` 구조를 미러링한 캐시 디렉터리에 `.dds` 로 저장하고, 다음 로드부터는 원본과 캐시의 파일 타임스탬프를 비교해 원본이 더 새로울 때만 재변환합니다. 원본이 이미 `.dds` 면 변환을 건너뜁니다.
- **경로 정규화** — 같은 텍스처가 `Data\a.png` 와 `Data/a.png` 로 두 번 로드되던 문제를 구분자 통일로 해결했고 (커밋 `48c2f9dc`), DDS 베이킹 시 한글 경로 크래시를 UTF-16 변환 경유로 해결했습니다 (커밋 `876c1457`). 에디터 UI 에는 원본 `.png` 대신 실제 로드된 `.dds` 경로를 표시합니다 (커밋 `432a7696`).

## 설계 2 — include 체인을 추적하는 셰이더 핫 리로드

[`Shader.cpp`](../Mundi/Source/Runtime/Renderer/Shader.cpp) (도입 `2625a7fb` → 매크로 variant 대응 `40bcc080` → include 체인 추적 `691f7324`)

`UShader` 하나가 **매크로 조합별 variant map** 을 소유합니다 (UberLit 처럼 `LIGHTING_MODEL_*`, `GPU_SKINNING` 등으로 여러 벌 컴파일되는 셰이더 대응). 핫 리로드는 세 단계입니다:

1. **변경 감지** (`IsOutdated`) — 메인 `.hlsl` 파일만이 아니라, 로드 시점에 `#include` 지시문을 재귀 파싱해 수집한 **모든 include 파일의 타임스탬프**를 비교합니다. 경로는 canonical 정규화로 통일하고 방문 집합으로 순환 include 를 방지합니다. 이 덕에 `LightingCommon.hlsl` 같은 공용 파일을 저장하면 그걸 include 하는 모든 셰이더가 리로드됩니다.
2. **재컴파일** (`Reload`) — 기존 variant map 을 백업으로 옮겨 두고, 백업의 매크로 조합대로 전부 다시 컴파일합니다.
3. **검증·롤백** — 하나라도 컴파일에 실패하면 새로 만든 것을 버리고 백업을 복원합니다. 컴파일 에러가 있는 저장을 해도 화면이 깨지지 않고 직전 정상 셰이더로 계속 렌더링됩니다. 교체 전에 파이프라인에서 셰이더를 언바인드하고 `Flush` 로 GPU 사용을 끝냅니다.

## 설계 3 — 한글 경로 호환

[`PathUtils.h`](../Mundi/Source/Runtime/Core/Misc/PathUtils.h) (커밋 `b4f0b897`, `3d60d0f1`, `ffa0c31e`)

엔진 내부 문자열은 UTF-8 로 고정하고, Windows API·`std::filesystem` 경계에서만 UTF-16 으로 변환하는 유틸(`UTF8ToWide`/`WideToUTF8`)을 두었습니다. `MultiByteToWideChar(CP_UTF8)` 실패 시 시스템 코드페이지(`CP_ACP`)로 폴백해 인코딩이 섞인 레거시 경로도 처리합니다. 파일을 여는 모든 지점(`TextureConverter`, 셰이더 로드·include 파싱)이 이 경계를 지나므로 한글 경로에서 동작합니다.

## 한계

- 캐시 무효화 기준이 원본 파일 타임스탬프뿐입니다. 압축 포맷이나 밉맵 옵션을 바꿔도 기존 캐시를 그대로 쓰므로 수동 삭제가 필요합니다.
- 핫 리로드가 `Flush` 로 GPU 를 동기 대기시킵니다. 에디터 반복 작업용으로는 충분하지만 런타임 스트리밍에는 부적합합니다.
- sRGB 여부를 호출부가 플래그로 지정합니다. 텍스처 용도(albedo/normal/data)를 파일명이나 메타데이터로 자동 판별하지 않습니다.
- include 파싱이 `#include "..."` 한 형태만 처리하고 전처리기 조건(`#ifdef` 안의 include)을 평가하지 않아, 실제로 쓰이지 않는 include 파일 변경에도 리로드가 일어날 수 있습니다.
