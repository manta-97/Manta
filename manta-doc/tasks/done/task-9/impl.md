# Task-9: Electron 환경 세팅 (`packages/desktop`)

## 개요

Manta의 데스크톱 앱 패키지를 monorepo에 추가한다. `@manta/core`를 공유하면서 Electron + React + Tailwind CSS v4 + Tailwind Plus(Catalyst) 환경을 구성하는 것이 목표다. 이 태스크에서는 **환경 세팅만** 하고, 실제 기능 화면은 구현하지 않는다.

## 설계 결정

### electron-forge + Vite

Electron 공식 추천 도구인 electron-forge를 사용한다. Vite 플러그인(`@electron-forge/plugin-vite`)으로 React + Tailwind 빌드를 처리한다.

- **왜 electron-forge인가**: 프로젝트 생성, 개발 서버, 패키징, 배포를 한 도구에서 처리. 설정이 단순하고 공식 지원이 안정적.
- **왜 Vite인가**: Webpack 대비 설정이 적고 HMR이 빠르다. Tailwind CSS v4도 Vite와 자연스럽게 통합된다.

### React + Tailwind CSS v4 + Tailwind Plus

- **React**: Tailwind Plus / Catalyst UI Kit이 React 전용. Headless UI도 React 버전 사용.
- **Tailwind CSS v4**: `@tailwindcss/postcss` + PostCSS로 통합. v4는 CSS-first로 동작한다.
- **Tailwind Plus(Catalyst)**: 구매한 프리미엄 컴포넌트 키트. 프로젝트에 소스 복사 방식으로 포함.

### @manta/core 연결

기존 monorepo의 `@manta/core`를 의존성으로 연결한다. CLI와 동일하게 `"@manta/core": "*"`로 workspace 참조.

단, **빌드 시스템이 다르다**:
- core/cli: `tsc` (CJS 출력)
- desktop: `Vite` (ESM 번들링)

electron-forge의 Vite 플러그인이 `@manta/core`의 CJS 출력을 main process에서 `require`로 로드하므로, 기존 core 빌드 방식을 변경할 필요 없다.

### 패키지 구조

```
packages/desktop/
├── package.json
├── tsconfig.json
├── forge.config.ts              # electron-forge 설정
├── vite.main.config.ts          # main process Vite 설정
├── vite.preload.config.ts       # preload script Vite 설정
├── vite.renderer.config.ts      # renderer process Vite 설정 (React + Tailwind)
├── postcss.config.js            # PostCSS 설정 (@tailwindcss/postcss)
├── index.html                   # renderer HTML 진입점
├── src/
│   ├── main.ts                  # Electron main process 진입점
│   ├── preload.ts               # contextBridge로 IPC 노출
│   └── renderer/
│       ├── index.tsx             # React 진입점
│       ├── app.tsx               # App 컴포넌트 (빈 셸)
│       └── index.css             # Tailwind CSS 임포트
```

### Electron의 프로세스 모델

Electron은 프로세스가 분리되어 있어서 Vite 설정도 3개로 나뉜다:

- **main process** (`vite.main.config.ts`): Node.js 환경. `@manta/core` 호출, 파일 I/O, 시스템 API. `target: 'node'`
- **preload** (`vite.preload.config.ts`): main ↔ renderer 브릿지. `contextBridge.exposeInMainWorld`로 안전한 API만 노출.
- **renderer** (`vite.renderer.config.ts`): 브라우저 환경. React + Tailwind. `target: 'web'`

### monorepo 통합

루트 `package.json`의 build 스크립트에 desktop을 추가한다. 단, desktop 빌드는 core에 의존하므로 순서를 보장해야 한다.

```
build 순서: core → cli (독립) / desktop (독립)
```

## 변경 사항

### 1. `packages/desktop/package.json` (신규)

```json
{
  "name": "@manta/desktop",
  "version": "0.1.0",
  "private": true,
  "main": ".vite/build/main.js",
  "scripts": {
    "start": "electron-forge start",
    "package": "electron-forge package",
    "make": "electron-forge make"
  },
  "dependencies": {
    "@manta/core": "*",
    "react": "^19",
    "react-dom": "^19"
  },
  "devDependencies": {
    "@electron-forge/cli": "^7",
    "@electron-forge/maker-dmg": "^7",
    "@electron-forge/maker-zip": "^7",
    "@electron-forge/plugin-vite": "^7",
    "@tailwindcss/postcss": "^4.2.2",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "@vitejs/plugin-react": "^4.7.0",
    "electron": "^36",
    "postcss": "^8.5.10",
    "tailwindcss": "^4",
    "typescript": "^6",
    "vite": "^6"
  }
}
```

### 2. `packages/desktop/tsconfig.json` (신규)

renderer는 Vite가 번들링하므로 이 tsconfig은 주로 main/preload 타입 체크용이다. `module: "node16"`을 유지하여 core/cli와 일관성을 맞춘다.

### 3. Vite 설정 3개 (신규)

- `vite.main.config.ts`: main process용. Node.js target, externals로 electron 제외.
- `vite.preload.config.ts`: preload용. Node.js target.
- `vite.renderer.config.ts`: renderer용. `@vitejs/plugin-react` 포함. Tailwind는 PostCSS로 처리.

### 4. `forge.config.ts` (신규)

electron-forge 설정. Vite 플러그인과 maker(dmg, zip) 등록.

### 5. `src/main.ts` (신규)

BrowserWindow 생성, renderer HTML 로드. 최소한의 윈도우 설정만.

### 6. `src/preload.ts` (신규)

빈 preload — 향후 IPC API를 여기서 노출.

### 7. `src/renderer/` (신규)

- `index.html` (프로젝트 루트): React root div + 번들 스크립트
- `index.tsx`: `createRoot` + `<App />` 렌더링
- `app.tsx`: "Manta" 텍스트만 표시하는 빈 컴포넌트
- `index.css`: `@import "tailwindcss"` (v4 방식)

### 8. 루트 `package.json` 수정

```diff
 "scripts": {
-  "build": "npm run build --workspace packages/core && npm run build --workspace packages/cli",
+  "build": "npm run build --workspace packages/core && npm run build --workspace packages/cli && npm run build --workspace packages/desktop",
 }
```

### 9. 루트 `jest.config.js` — 변경 없음

desktop은 단위 테스트 대상이 아직 없으므로 jest project를 추가하지 않는다. 필요해지면 그때 추가.

## 완료 기준

- `npm install` 후 `npm start --workspace packages/desktop`으로 빈 Electron 윈도우가 뜬다
- 윈도우에 Tailwind 스타일이 적용된 텍스트가 보인다
- `@manta/core`를 main process에서 import할 수 있다
