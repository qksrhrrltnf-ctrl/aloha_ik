# Aloha Hand Tracking Robot Simulator - Codebase Notes

이 문서는 현재 코드베이스의 실제 구조를 설명한다. 현재 앱의 중심 기능은 Gemini 기반 물체 탐지가 아니라, MediaPipe 손 추적을 이용한 Aloha 양팔 로봇 제어다.

## System Overview

앱은 브라우저 안에서 다음 루프를 실행한다.

1. MuJoCo WASM을 로드한다.
2. DeepMind MuJoCo Menagerie에서 `aloha` 모델 XML과 asset을 다운로드한다.
3. MuJoCo model/data를 생성한다.
4. Three.js 씬에 MuJoCo geometry를 렌더링한다.
5. MediaPipe HandLandmarker로 웹캠의 손 landmark를 추적한다.
6. 손목 위치와 손 크기를 로봇 좌표계 target으로 변환한다.
7. Aloha IK가 target pose에 맞도록 각 팔의 joint qpos/control을 갱신한다.
8. MuJoCo simulation step 후 Three.js 렌더링을 갱신한다.

## Entry Points

### `index.tsx`

React root를 생성하고 `App`을 마운트한다.

### `App.tsx`

현재 앱의 최상위 controller다.

주요 책임:

- MuJoCo WASM 초기화
- `MujocoSim` 생성 및 Aloha 모델 로딩
- MediaPipe HandLandmarker 초기화
- 웹캠 시작/중지
- 손 landmark를 로봇 target 좌표로 변환
- pinching 거리로 gripper 제어
- 좌/우 팔 상태 표시
- toolbar, camera preview, robot status UI 렌더링

중요한 현재 조건:

- `selectedRobotId`는 `aloha`로 고정되어 있다.
- 앱 시작 후 시뮬레이션이 준비되면 카메라를 자동 시작하려고 한다.
- 카메라 권한, 장치 없음, HTTPS 문제, MediaPipe 로딩 실패는 화면의 error panel로 안내한다.

## Simulation Layer

### `MujocoSim.ts`

MuJoCo simulation과 Three.js rendering을 연결하는 중심 class다.

주요 책임:

- `RobotLoader`로 robot XML과 asset 로딩
- `MjModel`, `MjData` 생성
- simulation loop 실행
- pause/reset/IK toggle 처리
- `RenderSystem.update()` 호출
- Aloha일 때 `AlohaIK.update()` 호출
- Franka legacy 경로 유지

현재 Aloha 경로:

- `robotId === 'aloha'`이면 `AlohaIK.setup()`을 실행한다.
- IK가 활성화되어 있고 pause 상태가 아니면 매 frame `AlohaIK.update()`를 호출한다.
- reset 시 `AlohaIK.syncToSim()`으로 target을 현재 simulation 상태에 맞춘다.

### `RobotLoader.ts`

MuJoCo Menagerie에서 robot model과 의존 asset을 가져온다.

현재 base URL:

```text
https://raw.githubusercontent.com/google-deepmind/mujoco_menagerie/main/{robotId}/
```

동작:

- `scene.xml`부터 시작한다.
- XML의 `include`, `mesh`, `texture`, `hfield` file attribute를 스캔한다.
- 필요한 파일을 MuJoCo WASM virtual filesystem의 `/working` 아래에 쓴다.

주의:

- 다운로드 실패 시 파일명, URL, HTTP 상태 또는 네트워크 오류를 포함해 즉시 error를 던진다.
- `App.tsx`는 이 메시지를 loading error 화면에 표시한다.
- GitHub 접근이 가능한 수업 환경을 기준으로 하되, 요청 실패나 저장소 구조 변경이 발생하면 실패 파일을 사용자가 확인할 수 있어야 한다.

## Rendering Layer

### `RenderSystem.ts`

Three.js scene graph와 renderer를 관리한다.

주요 책임:

- renderer/camera/orbit controls 생성
- scene background, grid, lights 설정
- MuJoCo body별 Three.js group 생성
- MuJoCo geom을 mesh로 변환해 body group에 연결
- 매 frame MuJoCo `xpos`, `xquat` 값을 Three.js object에 반영
- contact marker, ER marker legacy 기능 유지
- canvas snapshot과 2D-to-3D projection legacy helper 제공

현재 교육용 Aloha 흐름에서는 `project2DTo3D`, ER marker 계열은 기본 흐름에서 사용되지 않는다.

### `rendering/GeomBuilder.ts`

MuJoCo geom type을 Three.js geometry로 변환한다.

지원:

- plane
- sphere
- capsule
- box
- cylinder
- mesh

plane은 `Reflector`를 사용해 반사 바닥으로 렌더링한다.

### `Reflector.ts`

반사 바닥용 custom mesh다. 렌더 타깃을 사용해 간단한 planar reflection을 구현한다.

### `CapsuleGeometry.ts`

MuJoCo capsule primitive를 렌더링하기 위한 custom geometry다.

## Hand Tracking And Control

### MediaPipe initialization

`App.tsx`에서 `FilesetResolver.forVisionTasks()`와 `HandLandmarker.createFromOptions()`를 사용한다.

현재 설정:

- `runningMode: "VIDEO"`
- `numHands: 2`
- `delegate: "GPU"`
- model: Google Storage의 hand landmarker task file

### Hand-to-robot mapping

`App.tsx` 내부의 `computeAxisValue()`가 손 landmark를 로봇 좌표로 변환한다.

입력:

- axis: `red`, `green`, `blue`
- mode: `Up/Down`, `Left/Right`, `Close/Far`, `Static`
- hand: `left` 또는 `right`
- wrist landmark
- hand size

출력:

- 해당 로봇 target axis 값

현재 기본 매핑:

- red: side-to-side
- green: depth
- blue: height

향후 개선 후보:

- `computeAxisValue()`를 별도 pure function으로 분리
- workspace boundary와 clamp 명시화
- smoothing 추가
- 좌표계 설명 UI 추가

### Gripper control

엄지 끝 landmark와 검지 끝 landmark 사이의 거리를 계산한다.

- 거리가 작으면 pinching으로 판단해 gripper를 닫는다.
- 거리가 크면 gripper를 연다.

현재 threshold는 코드에 하드코딩되어 있다.

## IK Layer

### `AlohaIK.ts`

Aloha 양팔을 위한 수치 IK solver다.

setup 과정:

- `right/gripper`, `left/gripper` site를 찾는다.
- 각 팔의 joint와 actuator를 이름으로 찾는다.
- 팔별 Three.js target group과 TransformControls를 만든다.
- target을 현재 simulation site 위치와 회전에 맞춘다.

update 과정:

1. target의 world position/quaternion을 읽는다.
2. MuJoCo local space 기준 goal pose로 변환한다.
3. finite difference 방식으로 6D Jacobian을 계산한다.
4. damped least squares 형태의 선형 시스템을 푼다.
5. joint range로 qpos를 clamp한다.
6. actuator control을 qpos에 맞춰 hold한다.

주요 파라미터:

- `ikDamping`
- `ikStepSize`
- `ikPosWeight`
- `ikRotWeight`
- `ikIterations`

향후 개선 후보:

- 파라미터를 설정값으로 분리
- target smoothing 추가
- unreachable target 처리 UI 추가
- `@ts-ignore` 제거

### `IkSystem.ts` and `FrankaAnalyticalIK.ts`

Franka Panda용 legacy IK 경로다.

현재 Aloha 고정 앱에서는 기본 흐름에 사용되지 않는다. 단, `MujocoSim`에는 Franka 경로가 아직 남아 있으므로 제거 또는 legacy 분리 여부를 결정해야 한다.

## UI Components

### `components/RobotSelector.tsx`

좌측 상단 robot status panel이다.

표시:

- robot name
- webcam 상태
- 인식된 손 개수
- 왼팔/오른팔 target position과 rotation
- 팔별 red/green/blue axis mapping select

### `components/Toolbar.tsx`

하단 toolbar다.

기능:

- pause/resume
- IK toggle
- reset
- dark mode toggle

### Camera preview overlay

`App.tsx` 안에 직접 구현되어 있다.

표시:

- webcam video
- MediaPipe landmark canvas overlay
- live indicator
- start/stop camera button

향후 `components/CameraPreview.tsx`로 분리할 수 있다.

### Removed legacy sidebar

이전 Gemini Embodied Reasoning UI였던 `components/UnifiedSidebar.tsx`는 현재 Aloha hand tracking 앱에서 사용되지 않아 제거했다.

## Utility Modules

### `MatMath.ts`

Aloha IK에서 사용하는 최소 행렬 연산 helper다.

포함:

- transpose
- matrix multiplication
- matrix-vector multiplication
- Gaussian elimination linear solver

### `utils/StringUtils.ts`

MuJoCo WASM memory의 null-terminated string을 JS string으로 변환한다.

## Known Technical Debt

- README는 정리되었지만, 수업용 docs는 아직 없다.
- Google Drive 동기화 폴더에서는 `node_modules` 설치가 불안정하다.
- MuJoCo/robot asset 로딩 실패 메시지는 파일명과 URL을 포함하지만, 수업용 troubleshooting 문서는 아직 없다.
- `App.tsx`에 초기화, MediaPipe, mapping, UI가 많이 모여 있다.
- Aloha hand mapping 값이 하드코딩되어 있다.
- 일부 legacy Franka/Gemini 로봇 동작 코드와 현재 Aloha 앱 코드가 아직 혼재되어 있다.
- runtime CDN 의존성이 많다.
- 직접 HTML importmap 방식과 Vite bundle 방식이 함께 남아 있어 정리가 필요하다.

## Recommended Next Refactors

1. `computeAxisValue()`를 `utils/handMapping.ts`로 분리
2. `CameraPreview` component 분리
3. legacy `SequenceAnimator`와 Franka IK 경로 정리 여부 결정
4. CDN asset vendoring 여부와 라이선스 확인
