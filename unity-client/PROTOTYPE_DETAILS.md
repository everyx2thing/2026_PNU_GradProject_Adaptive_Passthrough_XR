# test_ovr — Context-aware Adaptive Passthrough Framework (Prototype)

Meta Quest(Unity, OpenXR/Meta XR SDK) 기반 몰입형 XR 안전 실험 프로젝트입니다.
사용자가 실제 공간(Room Scene) 안에서 움직일 때 **벽까지의 거리, 접근 속도/가속도,
머리·손 움직임, 시선-경계 각도**를 실시간으로 분석하여 위험도(Risk Score)를 계산하고,
Passthrough 활성화 여부를 판단하는 것을 목표로 합니다.

졸업과제 착안보고서(`졸업과제_착안보고서.pdf`)에서 제안한 "상황 인식 기반 Adaptive
Passthrough Framework"의 정적 경계 위험도(Rcollision) + 사용자 상태 위험도(Rstate)
파트를 Unity/Quest 환경에서 구현한 프로토타입입니다.

## Requirements

- Unity **6000.4.2f1** (Unity 6)
- Meta Quest 계열 헤드셋 (Quest 2 / 3 / 3S / Pro), 개발자 모드 활성화
- 헤드셋에서 **Space Setup(Room Setup)을 먼저 완료**해야 함 (Scene API로 벽 정보를 가져오기 때문)
- Android Build Support 모듈

## Packages / Dependencies

`Packages/manifest.json` 기준 주요 패키지:

- `com.meta.xr.sdk.core` (203.0.0) — OVRManager, OVRCameraRig, Boundary/Scene API
- `com.meta.xr.sdk.interaction.ovr` (203.0.0) — 인터랙션 SDK
- `com.unity.xr.management` / `com.unity.xr.openxr` — XR Plug-in Management, OpenXR
- `com.unity.render-pipelines.universal` (URP)
- `com.unity.inputsystem` — 새 Input System
- `com.coplaydev.unity-mcp`, `com.meta.xr.unity-mcp.extension` — AI 에디팅 툴링(개발 편의용, 런타임 빌드에는 불필요)

전체 의존성 목록/잠금 버전은 `Packages/manifest.json`, `Packages/packages-lock.json` 참고.

## 저장소에 포함된 것 / 제외된 것

이 저장소에는 Unity 프로젝트 중 아래 세 폴더만 커밋합니다.

- `Assets/` — 씬, 스크립트, UI, 세팅 에셋 등 프로젝트 콘텐츠
- `ProjectSettings/` — 빌드 타겟, XR/그래픽스 설정, 태그/레이어 등
- `Packages/` — 패키지 의존성 목록 (`manifest.json`, `packages-lock.json`)

아래 항목은 로컬에서 자동 생성되는 캐시/빌드 산출물이므로 **커밋하지 않습니다** (`.gitignore` 참고).

- `Library/`, `Temp/`, `Obj/`, `Logs/`, `UserSettings/`, `.vs/`
- `Builds/`, `*.apk` (예: `test2.apk`)
- IDE 생성 파일 (`*.csproj`, `*.sln`)

> 새로 clone한 경우 Unity Hub로 프로젝트를 처음 열면 `Library/`가 자동으로 재생성됩니다
> (패키지 재해석 때문에 다소 시간이 걸릴 수 있습니다).

## Project Structure

```
Assets/
├─ Scenes/
│  └─ SampleScene.unity              # 유일한 씬. OVRCameraRig + 아래 로거 스크립트들이 붙어있음
├─ Scripts/
│  ├─ QuestBoundaryLogger.cs         # Guardian(Boundary) 상태/치수/geometry 로깅 (현재 비활성)
│  ├─ QuestSceneDistanceLogger.cs    # 최초 구현한 Scene API 기반 벽 거리 + 모션 피처 로거 (실험 기준선으로 보존, 현재 컴포넌트 비활성)
│  └─ QuestRiskExperimentLogger.cs   # 위험도(Rcollision/Rstate/Rtotal) 계산 및 Passthrough 판단 실험 스크립트 (현재 활성)
├─ Settings/                         # URP 렌더 파이프라인/볼륨 프로파일 (PC/Mobile)
├─ Resources/                        # OVR/XR 관련 설정 에셋 (OVRBuildConfig, ImmersiveDebuggerSettings 등)
└─ Plugins/Android/                  # AndroidManifest.xml
```

## Key Scripts

### `QuestBoundaryLogger.cs`
- `OVRManager.boundary`에서 Guardian 설정 여부(`GetConfigured`), Play Area 치수(`GetDimensions`), 경계 geometry(`GetGeometry`)를 폴링하여 로그/화면(OnGUI)에 출력
- Guardian이 아직 설정되지 않았으면 1초 간격으로 재시도
- 현재 씬에서는 컴포넌트가 비활성 상태로 유지됨

### `QuestSceneDistanceLogger.cs`
- `com.oculus.permission.USE_SCENE` 권한 요청 → 승인 시 `OVRAnchor`로 룸 앵커 및 자식 앵커(벽) 조회
- `OVRSemanticLabels`로 WallFace / InvisibleWallFace만 필터링해 실제 세계 좌표로 벽 위치·법선 저장
- 매 프레임 HMD 위치와 각 벽 사이 거리를 계산해 가장 가까운 벽과 거리를 산출
- Head Speed/Accel/Angular Speed, Hand Speed, Hand-Head Ratio, Toward Wall Speed, TTC 등 모션 피처 계산
- 위험도 실험 이전의 최초 버전으로, 회귀 비교를 위해 파일은 보존하되 씬에서는 비활성화되어 있음

### `QuestRiskExperimentLogger.cs` (현재 핵심 스크립트)
`QuestSceneDistanceLogger.cs`의 벽 거리/모션 계산 로직을 독립적으로 재구현한 뒤,
아래 위험도 계산 구조를 추가한 실험용 스크립트입니다.

코드는 가중합을 그대로 쓰지 않고, 가중치 합(`weightSum`)으로 나누어 정규화합니다.
README 수식도 이를 반영합니다.

```
collisionWeightSum = weightDistance + weightTTC + weightApproachAccel + weightBlind

Rcollision = (weightDistance      * Rd
            + weightTTC           * RTTC
            + weightApproachAccel * Ra
            + weightBlind         * Rblind) / collisionWeightSum

totalWeightSum = weightCollisionTotal + weightStateTotal + weightDynamicTotal + weightIntentTotal

Rtotal = (weightCollisionTotal * Rcollision
        + weightStateTotal     * Rstate
        + weightDynamicTotal   * Rdynamic   (현재 0 고정 — AI/ML 미구현)
        + weightIntentTotal    * Rintent    (현재 0 고정 — AI/ML 미구현)) / totalWeightSum

shouldEnablePassthrough = Rtotal >= passthroughOnThreshold
```

(각 분모가 0이 되는 경우에는 코드에서 1로 처리하여 0으로 나누는 것을 방지합니다.)

- `Rd`: 최소 경계 거리 기반 위험도
- `RTTC`: Time-To-Collision 기반 위험도
- `Ra`: 경계 방향 접근 가속도 기반 위험도
- `Rblind`: 시선 방향(`_hmdTransform.forward`)과 벽 접근 방향 사이 각도 기반 사각지대 위험도
- `Rstate`: 사용자 움직임 상태(Static/Dynamic/Agitated) 기반 위험도. **코드의 `Rstate`는 보고서 3.3.3절의 `Rstatic`에 해당하는 구현 변수명이다.**

Inspector에서 조정 가능한 항목 (모두 `[SerializeField]`). 이 중 **가중치(weight로 시작하는 필드)와
`passthroughOnThreshold`에만 `[Range(0,1)]`이 적용**되어 슬라이더로 표시되며, User State
Thresholds/Risk Parameters의 속도·가속도·거리·시간 기준값들은 `[Range]` 없이 자유 입력
필드(0~1 범위에 묶이지 않음)입니다.

| Header | 필드 |
|---|---|
| User State Thresholds | `staticHeadSpeedThreshold`, `staticHeadAccelThreshold`, `staticHeadAngularThreshold`, `agitatedHeadSpeedThreshold`, `agitatedHeadAccelThreshold`, `agitatedHeadAngularThreshold` |
| Risk Parameters | `safeDistance`, `safeTime`, `maxApproachAccel` |
| Collision Risk Weights | `weightDistance`, `weightTTC`, `weightApproachAccel`, `weightBlind` |
| Total Risk Weights | `weightCollisionTotal`, `weightStateTotal`, `weightDynamicTotal`, `weightIntentTotal` |
| Passthrough Decision | `passthroughOnThreshold` |

계산 결과는 실제 Passthrough를 제어하지 않고, world-space UI 캔버스(`DistanceCanvas`)의
좌/우 두 텍스트 패널에 표시됩니다. 왼쪽엔 Scene Distance/Motion Features/Wall Approach,
오른쪽엔 User State, Collision Risk, Total Risk 및 `Passthrough Decision: ON/OFF`가 출력됩니다.

## Setup

1. Unity Hub에서 프로젝트 열기 (Unity 6000.4.2f1, Android Build Support 모듈 포함)
2. `Assets/Scenes/SampleScene.unity` 오픈
3. Build Settings → Android 플랫폼으로 전환, Quest 기기 연결 후 Build & Run
4. 헤드셋에서 아직 Space Setup을 안 했다면 먼저 진행 (첫 실행 시 SCENE 권한 승인 필요)
5. 실행 후 화면에 벽 거리/모션 피처(왼쪽)와 위험도/Passthrough 판단 결과(오른쪽)가 실시간 출력됨

## 알려진 제한 사항 / TODO

- `Rdynamic`(동적 객체 충돌 위험도), `Rintent`(접근 의도 위험도)는 YOLOv8n 객체 탐지 + TCN-GRU 예측 모델이 필요한 영역으로 아직 미구현, 현재 0으로 고정
- 실제 Passthrough 온/오프 제어(카메라 레이어 전환)는 연결되어 있지 않음 — 현재는 판단 결과만 UI 텍스트로 표시
- 로그 수집 기반 ML 개인화(가중치/임계값 자동 조정)는 미구현
- 다양한 시각화 방식(Directional Passthrough, Augmented Virtuality, Volumetric Cue) 비교 실험 미구현
- 사용자 상태 분류는 현재 HMD 속도, 가속도, 각속도의 순간값 기반으로 동작하며, 보고서의 이동 지속 시간 조건 및 최근 n프레임 평균/최대값 기반 안정화 로직은 아직 미구현이다

전체 시스템 설계와 향후 계획은 `졸업과제_착안보고서.pdf`를 참고하세요.

## Notes

- `Assets/_Recovery/`(자동복구 씬 잔여물), `Assets/MetaSkills/`(AI 에이전트 툴링 문서), `Assets/TutorialInfo/`(Unity 기본 템플릿)는 실제 프로젝트 동작과 무관한 항목으로 저장소에서 제외 검토 중
- `test2.apk`, `Builds/`, `Library/`, `Logs/`, `UserSettings/` 등 빌드/캐시 산출물은 `.gitignore` 대상 (저장소 루트의 `.gitignore` 참고)
