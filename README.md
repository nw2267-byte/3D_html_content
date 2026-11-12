# Three.js 기반 3D 디지털 트윈 프로젝트 분석 보고서

본 문서는 첨부된 두 개의 단일 파일(Single-File) 웹 애플리케이션을 분석하여 **실행 방법**과 **전체적인 코드 및 문법 구조**를 정리한 가이드입니다.

* **파일 1 (`smart_classroom_camera_stable.html`)**: 스마트 학교 교실 환경 시뮬레이션 및 카메라 안정화 버전
* **파일 2 (`ThreeJS_korean_bus.html`)**: 한국형 저상 전기 시내버스 스마트팩토리 8단계 조립 공정 시뮬레이션

---

## 1. 프로젝트 실행 방법

두 파일 모두 외부 빌드 도구(Webpack, Vite 등)나 별도의 로컬 에셋(3D 모델 `.glb`, 텍스처 이미지 등) 없이 **브라우저에서 단독으로 실행 가능한 형태**로 작성되어 있습니다.

### ① 인터넷 연결 필수 확인
두 파일 모두 `<script type="importmap">`을 통해 **jsDelivr CDN**에서 Three.js 코어 라이브러리와 `OrbitControls` 애드온을 실시간으로 불러옵니다.
* 스마트 교실: `three@0.164.1`
* 전기 시내버스: `three@0.180.0`
> ⚠️ 오프라인 환경에서는 CDN 모듈 로딩이 실패하여 에러/폴백(Fallback) 화면이 표시됩니다.

### ② 실행 방법 (2가지)

#### 방법 A: 로컬 웹 서버를 통한 실행 (권장)
ES 모듈(`type="module"`) 기반 스크립트는 브라우저의 보안 정책(CORS)상 로컬 웹 서버 환경에서 가장 안정적으로 동작합니다.
1. **VS Code 사용 시**:
   * `Live Server` 확장 프로그램 설치
   * HTML 파일 우클릭 → **`Open with Live Server`** 클릭
2. **Python 내장 서버 사용 시** (터미널/명령 프롬프트):
   ```bash
   # 파일이 위치한 폴더로 이동 후 실행
   python -m http.server 8000
   ```
   * 브라우저 주소창에 `http://localhost:8000/smart_classroom_camera_stable.html` 또는 `http://localhost:8000/ThreeJS_korean_bus.html` 입력

#### 방법 B: 파일 직접 열기
* 탐색기에서 `.html` 파일을 더블클릭하여 **Chrome, Edge, Whale** 등 최신 브라우저로 바로 열어도 실행되도록 설계되어 있습니다. (외부 텍스처 파일을 로드하지 않고 HTML5 `<canvas>`로 동적 텍스처를 생성하기 때문입니다.)

---

## 2. 공통 아키텍처 및 핵심 문법 구조

두 프로젝트는 모두 **HTML + CSS + Vanilla JS(ES6+) + Three.js**가 하나의 HTML 파일 안에 캡슐화된 **모놀리식(Monolithic) 구조**를 취하고 있습니다.

### ① HTML & 메타데이터 구조
```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <!-- 반응형 뷰포트 및 다크 테마 색상 지정 -->
  <meta name="viewport" content="width=device-width,initial-scale=1...">
  <style> /* CSS 변수(:root) 및 글래스모피즘(backdrop-filter) UI 스타일 */ </style>
  
  <!-- Import Map: 번들러 없이 브라우저에서 bare specifier('three') 해석 -->
  <script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@.../build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@.../examples/jsm/"
    }
  }
  </script>
</head>
<body>
  <!-- 3D 캔버스가 마운트될 컨테이너 -->
  <div id="stage"></div> <!-- 또는 <div id="world"></div> -->
  
  <!-- 오버레이 HUD / 사이드바 / 컨트롤 패널 UI -->
  ...
  
  <!-- 메인 3D 로직 (ES Module) -->
  <script type="module"> ... </script>
  
  <!-- 모듈 로딩 실패 대비 타임아웃 및 전역 에러 핸들러 (일반 script) -->
  <script> ... </script>
</body>
</html>
```

### ② CSS 설계 문법
* **CSS Custom Properties (`:root`)**: 테마 색상(`--deep`, `--ink`, `--teal` 등)을 변수화하여 일관된 다크 UI 유지.
* **레이아웃 기법**:
  * 스마트 교실: `position: fixed; inset: 0` 전체 화면 3D 캔버스 위에 `position: absolute`와 `backdrop-filter: blur()`를 적용한 플로팅 패널 배치.
  * 전기 버스: CSS Grid(`grid-template-columns: 292px minmax(0,1fr) 272px`)를 활용한 좌측 공정 메뉴 - 중앙 3D 뷰포트 - 우측 관제 패널 3단 대시보드 레이아웃.
* **반응형 미디어 쿼리 (`@media`)**: 태블릿 및 모바일 해상도(`1250px`, `1070px`, `990px`, `780px`, `690px` 등)에 맞춰 사이드바 슬라이드 토글 및 하단 도크 재배치.

### ③ Three.js 3D 그래픽스 파이프라인 문법
1. **렌더러 설정 (`WebGLRenderer`)**:
   * `preserveDrawingBuffer: true`: 화면 캡처(PNG 다운로드 기능)를 위해 프레임 버퍼 보존.
   * `outputColorSpace = THREE.SRGBColorSpace` 및 `ACESFilmicToneMapping`: 실사풍 색감과 명암 표현.
   * `PCFSoftShadowMap`: 부드러운 그림자 처리.
2. **절차적 3D 모델링 (Procedural Geometry)**:
   * 외부 3D 에셋(`.gltf`, `.obj`) 대신 `BoxGeometry`, `CylinderGeometry`, `SphereGeometry`, `TorusGeometry`, `TubeGeometry` 등 기본 기하학 도형을 조합하는 헬퍼 함수(`bx()`, `cyl()`, `sph()`, `rod()`, `tube()`)를 정의하여 모든 사물(책상, 학생, 버스 차체, 로봇 팔 등)을 코드로 직접 생성.
   * `THREE.Group()`을 계층적으로 구성하여 부품 단위 제어(가시성 토글, 분해 보기, JSON 좌표 추출) 용이.
3. **동적 캔버스 텍스처 (`CanvasTexture`)**:
   * 이미지 파일 로딩 없이 메모리 상에서 `document.createElement('canvas')`로 2D 그래픽(전자칠판 UI, 센서 라벨 스프라이트, 버스 행선지 전광판, 공장 안전 표지판, 바닥 나무결/벽면 질감)을 그린 뒤 `new THREE.CanvasTexture(canvas)`로 변환하여 재질(`map`)에 적용.
4. **카메라 제어 및 부드러운 시점 전환 (Tweening / Lerp)**:
   * `OrbitControls`로 마우스 회전/줌/패닝 지원.
   * 프리셋 버튼 클릭 시 `Vector3.lerp()` 또는 `lerpVectors()`와 Easing 수식을 이용해 카메라 위치와 타겟(`controls.target`)을 부드럽게 보간 이동.

### ④ 방어적 프로그래밍 (에러 핸들링)
* **이중 스크립트 구조**: `type="module"` 스크립트는 네트워크 오류나 문법 오류 시 조용히 실패할 수 있으므로, 하단에 일반 `<script>`를 배치하여 `window.addEventListener('error')`, `unhandledrejection`, `setTimeout(12~14초)`을 통해 로딩 실패 안내 UI를 띄움.
* 스마트 교실 파일의 경우 `webglcontextlost` 및 `webglcontextrestored` 이벤트를 감지해 GPU 컨텍스트 유실 시 복구 안내를 표시.

---

## 3. 두 파일의 구조적 차이점 비교

| 구분 | `smart_classroom_camera_stable.html` | `ThreeJS_korean_bus.html` |
| :--- | :--- | :--- |
| **주요 테마** | 스마트 학교 교실 환경 IoT 디지털 트윈 | 한국형 저상 전기버스 스마트팩토리 조립 공정 |
| **Three.js 버전** | `v0.164.1` | `v0.180.0` |
| **데이터 명세 관리** | `<script type="application/json" id="sceneSpec">` 태그 내부에 JSON 메타데이터 내장 (Blender Python 변환 호환 목적) | JS 배열(`stageData`) 및 객체(`viewPresets`, `colors`)로 하드코딩 관리 |
| **핵심 시뮬레이션 로직** | 미분방정식 근사(`1 - Math.exp(-dt/tau)`)를 활용한 실시간 온도·CO₂·조도·전력 소비량 물리 연산 및 `<canvas>` 시계열 그래프 렌더링 | 8단계 공정 상태 머신(`state.stage`), 자동 조립 시퀀스 타이머, 부품별 애니메이션(`from` → 목표 위치) 및 `explode`(분해 보기) 오프셋 |
| **특화 렌더링 기법** | • **카메라 안정화**: `clampTarget()`으로 카메라 이탈 방지<br>• **자동 벽면 절개(`updateWallCutaway`)**: 카메라가 벽 밖으로 나갈 때 앞벽/창문을 투명화<br>• 정적 그림자 맵 고정(`shadowMap.autoUpdate = false`)으로 GPU 부하 최소화 | • **레이캐스팅(`Raycaster`)**: 3D 부품을 직접 클릭하여 선택 및 상세 정보 조회<br>• **X-RAY 모드**: 외판/유리 재질(`MeshPhysicalMaterial`)의 `opacity`와 `depthWrite`를 동적 제어해 내부 골격 투시 |
| **데이터 내보내기** | • 장면 구조 스냅샷 (`.json`)<br>• 시계열 센서 로그 (`.csv`)<br>• 현재 3D 뷰 캡처 (`.png`) | • 현재 조립 단계 3D 뷰 캡처 (`.png`) |
| **애니메이션 루프** | 표준 `requestAnimationFrame(animate)` 사용 | `renderer.setAnimationLoop(...)` 사용 |

---

## 4. 코드 흐름 요약 (Execution Flow)

두 파일 모두 아래와 같은 생명주기(Lifecycle) 순서로 동작합니다.

1. **초기화 (`Init`)**: DOM 요소 참조(`$`), `WebGLRenderer`·`Scene`·`PerspectiveCamera`·`OrbitControls` 생성 및 조명(`HemisphereLight`, `DirectionalLight`) 배치.
2. **에셋 및 씬 구축 (`Build`)**: 절차적 재질(`MeshStandardMaterial` / `MeshPhysicalMaterial`) 정의 → 헬퍼 함수를 통한 3D 오브젝트 계층(`Group`) 조립 → 동적 `CanvasTexture` 생성.
3. **상태 및 이벤트 바인딩 (`State & Events`)**: UI 슬라이더·버튼·단축키·윈도우 리사이즈(`ResizeObserver` / `resize` 이벤트) 리스너 연결.
4. **렌더 루프 (`Render Loop`)**: 매 프레임 `clock.getDelta()`로 경과 시간(`dt`)을 계산하여 물리/조립 상태 갱신, 카메라 보간 이동, `controls.update()`, 최종 `renderer.render(scene, camera)` 호출.
5. **준비 완료 플래그**: 전역 변수(`__SMART_CLASSROOM_READY__` / `__busReady`)를 `true`로 전환하고 로딩 오버레이를 제거.