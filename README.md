# Aloha Hand Tracking Robot Simulator

MediaPipe hand tracking으로 웹캠의 손 움직임을 인식하고, 브라우저 안의 Aloha 양팔 로봇 시뮬레이터를 제어하는 교육용 웹 앱입니다.

이 프로젝트는 로봇팔 제어, 좌표계, inverse kinematics, 손 추적 기반 인터페이스를 수업에서 직접 보여주기 위한 목적으로 정리하고 있습니다.

## 주요 기능

- 브라우저에서 실행되는 MuJoCo WASM 기반 Aloha 로봇 시뮬레이션
- Three.js 기반 3D 렌더링
- MediaPipe HandLandmarker를 이용한 양손 추적
- 손목 위치를 로봇팔 target 위치로 매핑
- 엄지와 검지 pinching 동작으로 gripper 열기/닫기
- 좌/우 팔별 축 매핑 설정
- 일시정지, 리셋, 다크 모드, IK 토글

## 실행 요구사항

- Node.js
- Chrome 또는 Edge 권장
- 웹캠
- 인터넷 연결

브라우저 카메라 권한이 필요합니다. 배포 환경에서는 HTTPS가 필요하며, `localhost` 개발 서버에서는 대부분의 브라우저가 카메라 접근을 허용합니다.

## 로컬 실행

```bash
npm install
npm run dev
```

기본 개발 서버는 다음 주소로 열립니다.

```text
http://localhost:3000
```

## 빌드

```bash
npm run build
```

프로덕션 빌드 결과는 `dist/` 폴더에 생성됩니다.

## GitHub Pages 배포

이 프로젝트의 기본 배포 대상은 GitHub Pages입니다.

`main` 브랜치에 push하면 `.github/workflows/deploy-pages.yml` GitHub Actions workflow가 실행되어 다음 작업을 수행합니다.

1. `npm ci`
2. `npm run build`
3. `dist/`를 GitHub Pages artifact로 업로드
4. GitHub Pages에 배포

저장소에서 GitHub Pages를 Actions 방식으로 켜야 합니다.

```text
Repository Settings > Pages > Build and deployment > Source: GitHub Actions
```

GitHub Pages는 보통 다음 형태의 HTTPS 주소로 배포됩니다.

```text
https://{github-username}.github.io/{repository-name}/
```

카메라 API는 HTTPS에서 동작하므로 GitHub Pages 배포 URL에서는 웹캠 권한 요청이 가능해야 합니다.

## Firebase Hosting 배포 옵션

Firebase Hosting도 정적 파일 배포 대상으로 사용할 수 있습니다.

```bash
npm run build
firebase deploy
```

Firebase를 사용할 경우 hosting public directory는 `dist`로 설정합니다.

## 세종시교육청 배포 기준

현재 배포 기준은 다음처럼 둡니다.

- 기본 배포: GitHub Pages
- 보조 배포: Firebase Hosting
- 제외 대상: Vercel, Netlify 등 학교망에서 접근이 제한될 수 있는 호스팅

GitHub 자체는 접근 가능한 환경을 기준으로 하므로, GitHub Pages와 GitHub-hosted source/asset 경로는 우선 지원 대상으로 봅니다.

## 개발 폴더 주의사항

Google Drive, OneDrive, Dropbox 같은 동기화 폴더 안에서는 `node_modules` 설치 중 파일 쓰기 오류가 발생할 수 있습니다. 이 저장소는 클라우드 동기화로 작업을 이어갈 수 있지만, 의존성 설치와 빌드 검증은 일반 로컬 폴더에서 더 안정적입니다.

현재 확인된 안정적인 검증 방식:

```text
프로젝트를 C:\tmp 같은 일반 로컬 폴더로 복사
npm install
npm run build
npx tsc --noEmit
```

## 런타임 네트워크 의존성

이 앱은 실행 중 여러 외부 리소스를 다운로드합니다. 세종시교육청 환경에서는 GitHub 접근이 가능하다는 전제로 두고, 우선 확인해야 할 외부 의존성은 MediaPipe와 Google model storage 쪽입니다.

- MuJoCo WASM: `unpkg.com`
- Aloha robot assets: `raw.githubusercontent.com` 또는 향후 저장소 내 포함 asset
- MediaPipe WASM: `cdn.jsdelivr.net`
- MediaPipe hand model: `storage.googleapis.com`
- Tailwind CDN: `cdn.tailwindcss.com`
- Google Fonts: `fonts.googleapis.com`

오프라인 수업 또는 외부 CDN 접근이 제한된 환경에서도 안정적으로 쓰려면 향후 MediaPipe WASM/model, MuJoCo WASM, robot asset을 프로젝트에 포함하는 작업이 필요합니다.

## 현재 상태

이 프로젝트는 교육용 배포를 목표로 정리 중입니다. 진행 상황은 [EDUCATION_DEPLOYMENT_TODO.md](EDUCATION_DEPLOYMENT_TODO.md)를 기준으로 관리합니다.

현재 검증 결과:

- `C:\tmp` 임시 로컬 복사본 기준 `npm install` 성공
- `C:\tmp` 임시 로컬 복사본 기준 `npm run build` 성공
- `C:\tmp` 임시 로컬 복사본 기준 `npx tsc --noEmit` 성공

## 코드 구조

- `App.tsx`: 앱 초기화, 카메라, MediaPipe 손 추적, 손 좌표 매핑, UI 상태 관리
- `MujocoSim.ts`: MuJoCo 시뮬레이션과 렌더링 루프 관리
- `AlohaIK.ts`: Aloha 양팔 IK 계산
- `RenderSystem.ts`: Three.js 씬, 카메라, 조명, MuJoCo geometry 렌더링
- `RobotLoader.ts`: MuJoCo Menagerie에서 Aloha 모델과 asset 로딩
- `components/RobotSelector.tsx`: 로봇 상태와 축 매핑 UI
- `components/Toolbar.tsx`: 실행 제어 UI

## 라이선스

소스 파일 헤더는 Apache-2.0을 따릅니다. 공개 배포 전 MuJoCo, MuJoCo Menagerie, MediaPipe, Three.js 및 기타 의존성 라이선스 고지를 정리할 예정입니다.
