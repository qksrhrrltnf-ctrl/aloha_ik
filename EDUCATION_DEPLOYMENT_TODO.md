# Education Deployment TODO

이 문서는 현재 `Aloha IK + MediaPipe` 시뮬레이터를 GitHub에 공개하고, 교육 현장에서 쉽게 접근하고 실행할 수 있게 만들기 위한 작업 목록이다.

현재 코드의 실제 중심 기능은 다음과 같다.

- React/Vite 기반 웹 앱
- Three.js 렌더링
- MuJoCo WASM 시뮬레이션
- DeepMind MuJoCo Menagerie의 `aloha` 모델 원격 로딩
- MediaPipe HandLandmarker 웹캠 손 추적
- 손 위치/크기/핀치 제스처를 Aloha 양팔 IK target과 gripper control로 매핑

현재 문서와 일부 코드는 이전 `Franka Panda + Gemini Embodied Reasoning` 버전의 흔적이 남아 있으므로, 교육용 배포 전 정리가 필요하다.

## 목표

- GitHub 저장소를 처음 보는 교사/학생도 5분 안에 실행할 수 있게 한다.
- 웹캠 권한, 네트워크 의존성, 브라우저 요구사항을 명확히 설명한다.
- Gemini API 키 없이도 기본 기능이 동작하게 한다.
- 배포된 URL에서 바로 체험할 수 있게 한다.
- 수업용 안내, 조작법, 문제 해결 가이드를 포함한다.
- 코드 구조를 현재 기능 중심으로 정리한다.

## 우선순위 기준

- `P0`: 배포/실행을 막는 문제
- `P1`: 교육용 사용성에 직접 영향을 주는 문제
- `P2`: 유지보수와 확장성 개선
- `P3`: 선택적 개선 또는 고급 기능

## Phase 1 - 실행 가능 상태 만들기

- [x] `P0` 의존성 설치 후 실제 빌드 오류 확인
  - 현재 `node_modules`와 lockfile이 없다.
  - `npm install` 또는 패키지 매니저 선택이 필요하다.
  - 확인 명령: `npm run build`
  - 진행 결과:
    - `mujoco_wasm@latest`는 npm 패키지가 아니어서 설치가 실패했다.
    - `mujoco-js@0.0.7`로 교체하고 Vite alias로 기존 import 이름을 유지했다.
    - Google Drive 작업 폴더에서는 `node_modules` 쓰기 중 `EBADF/EPERM`이 반복되어 직접 설치가 불안정했다.
    - `C:\tmp` 임시 복사본에서는 `npm install`, `npm run build`, `npx tsc --noEmit`이 통과했다.
    - `package-lock.json`을 생성했다.

- [ ] `P0` TypeScript private 접근 오류 수정
  - `App.tsx`에서 `simRef.current.isAloha`를 사용한다.
  - `MujocoSim.ts`에서는 `isAloha`가 `private`이다.
  - 해결안: `get isAlohaRobot()` 같은 public getter 추가, 또는 public readonly 상태로 노출.

- [x] `P0` README 실행 절차를 실제 앱 기준으로 수정
  - 현재 README는 AI Studio/Gemini 중심이다.
  - 현재 앱은 기본적으로 Gemini API 키가 필요하지 않다.
  - Node.js 버전, 설치, 실행, 카메라 권한, 네트워크 요구사항을 명시한다.
  - 진행 결과:
    - README를 현재 Aloha + MediaPipe 손 추적 앱 기준으로 재작성했다.
    - Gemini API 키 필수 안내를 제거했다.
    - 카메라 권한, HTTPS/localhost 조건, CDN 의존성, Google Drive 폴더 설치 주의사항을 추가했다.

- [ ] `P0` `.env.local` 의존성 정리
  - 현재 `GEMINI_API_KEY=PLACEHOLDER_API_KEY`가 있지만 메인 기능에서는 사용하지 않는다.
  - 교육용 배포에서는 불필요한 API 키 요구를 제거하거나 선택 기능으로 분리한다.

- [x] `P0` GitHub Pages/Firebase 중 배포 방식을 결정하고 설정
  - 세종시교육청 환경에서는 GitHub Pages와 Firebase Hosting을 우선 지원 대상으로 한다.
  - Vercel, Netlify 등 학교망에서 접근이 제한될 수 있는 호스팅은 기본 대상에서 제외한다.
  - GitHub 자체는 접근 가능하다는 전제로 둔다.
  - GitHub Pages를 기본 배포 경로로 설정한다.
  - Firebase Hosting은 보조 배포 옵션으로 문서화한다.
  - 진행 결과:
    - Vite `base`를 `./`로 설정해 GitHub Pages 하위 경로와 Firebase Hosting 양쪽에서 asset 경로가 깨지지 않게 했다.
    - `.github/workflows/deploy-pages.yml`을 추가했다.
    - README에 GitHub Pages Actions 배포 방법과 Firebase Hosting 옵션을 추가했다.

## Phase 2 - 현재 기능 중심으로 코드 정리

- [x] `P1` 프로젝트 제목과 metadata 정리
  - `index.html` title이 `Franka Panda | Spatial UI`로 되어 있다.
  - 앱 이름을 `Aloha Hand Tracking Robot Simulator` 등으로 통일한다.
  - `metadata.json` 이름/설명도 교육용 문구로 수정한다.
  - 진행 결과:
    - `index.html` title을 `Aloha Hand Tracking Robot Simulator`로 변경했다.
    - `metadata.json` 이름과 설명을 교육용 Aloha hand tracking 앱 기준으로 변경했다.

- [x] `P1` 오래된 Franka/Gemini 문서 정리
  - `Code.md`는 현재 기능과 맞지 않는 내용이 많다.
  - 현재 Aloha/MediaPipe 구조 기준으로 다시 작성한다.
  - 진행 결과:
    - `Code.md`를 Aloha + MediaPipe hand tracking 앱 구조 기준으로 재작성했다.
    - Franka/Gemini 흐름은 현재 기본 기능이 아닌 legacy 경로로 명시했다.
    - 실제 실행 흐름, 주요 모듈, UI, 기술 부채, 다음 refactor 후보를 정리했다.

- [x] `P1` 미사용 UI 컴포넌트 처리
  - `components/UnifiedSidebar.tsx`는 현재 `App.tsx`에서 사용되지 않는다.
  - Gemini 탐지/집기 UI를 나중에 살릴 계획이 없다면 제거한다.
  - 살릴 계획이면 `experimental` 또는 `legacy`로 명확히 분리한다.
  - 진행 결과:
    - `components/UnifiedSidebar.tsx` 참조 여부를 확인했다.
    - 현재 앱에서 import/render되지 않으므로 삭제했다.
    - `UnifiedSidebar` 전용 타입인 `DetectType`, `LogEntry`, `DetectedItem`도 제거했다.
    - `Code.md`의 legacy sidebar 설명을 삭제 완료 상태로 갱신했다.

- [x] `P1` 미사용 의존성 정리
  - 현재 메인 앱에서 `@google/genai`, `uuid`, `react-markdown`는 사용되지 않는다.
  - 제거 전 `rg`로 참조를 재확인한다.
  - 제거 후 `npm install`로 lockfile 갱신한다.
  - 진행 결과:
    - `@google/genai`, `uuid`, `react-markdown`를 제거했다.
    - 현재 메인 앱 기준 의존성은 React, lucide, Three.js, MuJoCo JS, MediaPipe로 축소했다.

- [ ] `P1` Three.js 버전 불일치 정리
  - `package.json`: `three ^0.181.0`
  - `index.html` importmap: `three 0.164.1`
  - Vite 배포에서는 importmap이 불필요할 가능성이 크다.
  - 직접 HTML 실행 지원 여부를 결정하고 정리한다.

## Phase 3 - 교육용 UX 개선

- [ ] `P1` 첫 화면 안내 추가
  - 카메라 권한이 필요한 이유
  - 손을 카메라에 보여주는 방법
  - 왼손/오른손이 각각 어떤 로봇 팔을 제어하는지
  - 핀치 제스처가 gripper를 닫는다는 점

- [x] `P1` 카메라 권한 실패 UX 개선
  - 현재 콘솔 에러만 출력한다.
  - 브라우저 권한 거부, 카메라 없음, HTTPS 아님 상황을 사용자에게 보여줘야 한다.
  - 진행 결과:
    - camera error state를 추가했다.
    - 권한 거부, 카메라 없음, 다른 앱에서 사용 중, HTTPS/localhost 문제를 사용자 메시지로 표시한다.
    - 카메라 재시도 버튼을 추가했다.

- [x] `P1` MediaPipe 로딩 실패 UX 개선
  - 현재 실패해도 앱 화면에서 원인을 알기 어렵다.
  - CDN 접근 실패, GPU delegate 실패, 모델 다운로드 실패를 안내한다.
  - 진행 결과:
    - hand tracking error state를 추가했다.
    - MediaPipe WASM/CDN 또는 hand landmark model 로딩 실패 시 화면에 안내를 표시한다.
    - 모델 로딩 실패 후 reload 버튼을 제공한다.

- [x] `P1` MuJoCo/모델 asset 로딩 실패 UX 개선
  - `RobotLoader.ts`는 파일 다운로드 실패 시 경고 후 계속 진행한다.
  - GitHub 접근은 가능한 환경을 기준으로 하되, 요청 실패/파일 누락/저장소 구조 변경 시 실패 파일명과 해결 안내가 필요하다.
  - 진행 결과:
    - `RobotLoader`가 asset 다운로드 실패 시 더 이상 경고만 남기고 계속 진행하지 않도록 수정했다.
    - 실패한 파일명, URL, HTTP 상태 또는 네트워크 오류를 포함해 error를 던진다.
    - `App.tsx`가 로봇 로딩 실패 메시지를 generic 문구로 덮어쓰지 않고 화면에 표시하도록 수정했다.

- [ ] `P1` 손 추적 상태 표시 개선
  - 현재 `Hands: N`만 표시된다.
  - `손 인식 안 됨`, `왼손 인식`, `오른손 인식`, `양손 인식`처럼 교육자가 설명하기 쉬운 상태가 필요하다.

- [ ] `P2` 축 매핑 UI 설명 개선
  - Red/Green/Blue 축이 로봇 좌표계에서 무엇을 의미하는지 학생이 이해하기 어렵다.
  - 툴팁 또는 간단한 도움말 패널을 추가한다.

- [ ] `P2` 안전한 기본 제어 범위 조정
  - 현재 손 좌표를 로봇 좌표로 변환하는 값이 하드코딩되어 있다.
  - 교육용으로는 workspace boundary, clamp, smoothing이 필요하다.

- [ ] `P2` 움직임 smoothing 추가
  - 손 추적 좌표가 흔들리면 IK target도 흔들릴 수 있다.
  - exponential moving average 또는 velocity clamp 적용을 검토한다.

## Phase 4 - 배포 안정성

- [ ] `P0` HTTPS 배포 확인
  - 브라우저 웹캠 API는 보통 HTTPS 또는 localhost에서만 정상 동작한다.
  - GitHub Pages 배포 URL에서 카메라 권한 동작을 확인한다.
  - Firebase Hosting을 쓰는 경우 Firebase 배포 URL에서도 확인한다.

- [ ] `P1` 브라우저 호환성 문서화
  - Chrome/Edge 권장
  - Safari/Firefox에서 MediaPipe GPU delegate 또는 WASM 차이가 있을 수 있다.

- [ ] `P1` CDN 의존성 목록 문서화
  - MuJoCo WASM: `unpkg.com`
  - MediaPipe WASM: `cdn.jsdelivr.net`
  - MediaPipe model: `storage.googleapis.com`
  - MuJoCo Menagerie assets: `raw.githubusercontent.com`
  - 세종시교육청 기준 GitHub 접근은 가능하므로, 우선 MediaPipe/Google Storage/CDN 접근성을 확인 대상으로 둔다.

- [ ] `P2` CDN 장애 대비 옵션 검토
  - 핵심 WASM/model/robot asset을 `public/`에 vendoring할지 결정한다.
  - 라이선스 확인 후 저장소 포함 여부를 판단한다.

- [ ] `P2` GitHub Actions CI 추가
  - PR/Push 시 `npm ci`와 `npm run build` 실행.
  - 타입 오류와 번들 오류를 조기에 잡는다.

## Phase 5 - 수업 자료화

- [ ] `P1` `docs/lesson-guide.md` 작성
  - 10분 체험 활동
  - 로봇팔 좌표계 설명
  - IK 개념 설명
  - 손 추적과 로봇 제어 연결 설명
  - 토론 질문

- [ ] `P1` `docs/troubleshooting.md` 작성
  - 카메라 권한이 안 뜰 때
  - 화면이 검게 보일 때
  - 로봇 모델이 안 뜰 때
  - 손 인식이 안 될 때
  - MediaPipe 또는 Google model storage 접근이 실패할 때

- [ ] `P2` `docs/controls.md` 작성
  - Pause
  - Reset
  - IK toggle
  - Dark mode
  - Axis mapping
  - Pinch gripper

- [ ] `P2` 데모 스크린샷 또는 GIF 추가
  - GitHub README 첫 화면에 실제 앱 이미지를 넣는다.
  - 카메라 영상 자체가 포함되지 않도록 개인정보에 주의한다.

## Phase 6 - 코드 품질과 유지보수

- [ ] `P1` `RobotLoader` 실패 처리 강화
  - 필수 XML/mesh 다운로드 실패 시 명확히 throw한다.
  - 누락된 파일 목록을 사용자에게 전달한다.

- [ ] `P2` `App.tsx` 분리
  - 현재 앱 초기화, 카메라, MediaPipe, hand-to-robot mapping, UI 렌더링이 한 파일에 모여 있다.
  - 후보 분리:
    - `hooks/useMujocoSimulation.ts`
    - `hooks/useHandTracking.ts`
    - `utils/handMapping.ts`
    - `components/CameraPreview.tsx`

- [ ] `P2` 손 좌표 매핑 로직 테스트 가능하게 분리
  - `computeAxisValue`가 `App.tsx` 내부 함수라 테스트하기 어렵다.
  - 순수 함수로 분리하고 기본 케이스를 테스트한다.

- [ ] `P2` Aloha IK 파라미터 설정화
  - `ikDamping`, `ikStepSize`, `ikIterations`, `ikRotWeight` 등을 상수/설정으로 분리한다.
  - 교육용/성능용 프리셋을 만들 수 있다.

- [ ] `P2` TransformControls 타입 처리 개선
  - `AlohaIK.ts`에 `@ts-ignore`가 있다.
  - 타입 보강 또는 좁은 type assertion으로 정리한다.

- [ ] `P3` Franka/Gemini legacy 기능 분리
  - 유지할 경우 별도 branch 또는 `legacy/` 문서로 분리한다.
  - 현재 교육용 Aloha 앱의 기본 경로에서는 제거하는 편이 좋다.

## Phase 7 - 저장소 공개 준비

- [ ] `P0` 라이선스 확인
  - 현재 파일 헤더는 Apache-2.0 표기를 사용한다.
  - MuJoCo Menagerie, MediaPipe, Three.js, MuJoCo WASM 관련 라이선스 문서를 확인하고 README에 정리한다.

- [ ] `P1` 저장소 설명 문구 작성
  - 예: `Browser-based Aloha robot arm simulator controlled by MediaPipe hand tracking for robotics education.`

- [ ] `P1` README 배지/링크 추가
  - Live Demo
  - Build status
  - License

- [ ] `P1` Issue template 추가
  - 실행 오류
  - 교육 활용 제안
  - 기능 요청

- [ ] `P2` CONTRIBUTING 문서 추가
  - 개발 환경
  - 브랜치 전략
  - PR 전 확인사항

## 권장 작업 순서

1. 의존성 설치 및 빌드 오류 확인
2. private 접근 오류 수정
3. README와 title/metadata를 현재 앱 기준으로 수정
4. 미사용 Gemini 의존성/문서 정리
5. 카메라/MediaPipe/MuJoCo 로딩 실패 UX 개선
6. 배포 플랫폼 설정
7. 교육용 문서 작성
8. CI 추가
9. 데모 URL과 스크린샷으로 README 완성

## 현재 확인된 실행 상태

- Git 저장소로 초기화되어 있지 않다.
- 원래 Google Drive 작업 폴더에서는 `npm install` 중 `node_modules` 쓰기 오류가 발생할 수 있다.
- lockfile을 생성했다.
- `C:\tmp` 임시 복사본 기준 `npm run build`가 통과한다.
- `C:\tmp` 임시 복사본 기준 `npx tsc --noEmit`이 통과한다.
- 네트워크 접근이 필요한 런타임 의존성이 많다.

## 다음에 바로 할 수 있는 첫 작업

첫 번째 실제 수정 후보는 다음 중 하나다.

- `MujocoSim.ts`의 `private isAloha` 접근 문제 수정
- README를 현재 Aloha/MediaPipe 앱 기준으로 재작성
- `index.html` title과 `metadata.json` 이름/설명 수정
- `.env.local`/Gemini API 키 관련 오해 제거

권장 첫 작업은 `private isAloha` 접근 문제 수정과 README 재작성이다. 이 둘은 배포 전 신뢰도에 바로 영향을 준다.
