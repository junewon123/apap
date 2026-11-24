# 올플랜트 (AllPlant)

식물 이름 검색, 생김새·색·향기·유사 식물 취향 추천, 433종 통합 식물 도감, 실시간 가드너 커뮤니티 및 관리자 회원/방문자 관리를 제공하는 웹 애플리케이션입니다.

## 기술 스택
- **Frontend:** React 19, Vite, TypeScript, Tailwind CSS
- **Backend / Serverless API:** Express (`/server/createApiApp.ts`), Vercel Serverless Catch-all (`/api/[...path].ts`)
- **Database / Auth:** Firebase Firestore & Google Authentication (`firebase-applet-config.json`, `firestore.rules`)

---

## GitHub + Vercel 배포 가이드

### 1. GitHub 저장소 업로드
```bash
git init
git add .
git commit -m "Initial commit: AllPlant"
git branch -M main
git remote add origin https://github.com/본인계정/저장소이름.git
git push -u origin main
```

### 2. Vercel 프로젝트 연결
1. [Vercel](https://vercel.com) 대시보드에서 **Add New → Project**를 클릭해 GitHub 저장소를 가져옵니다.
2. `vercel.json`이 포함되어 있으므로 빌드 설정은 자동 인식됩니다.
   - **Build Command:** `npm run build`
   - **Output Directory:** `dist`
3. 별도 외부 검색 API 키 없이도 내장 433종 통합 백과사전·유사 식물 추천·커뮤니티가 바로 동작합니다.
4. **Deploy**를 눌러 배포합니다.

### 3. Firebase 로그인 도메인 허용 (필수)
Vercel 배포 주소(`https://프로젝트명.vercel.app`)에서 Google 로그인이 동작하도록 아래 설정을 추가하세요:
1. [Firebase Console](https://console.firebase.google.com)에서 프로젝트(`learned-outlet-5drm3`) 선택
2. **Authentication → Settings → Authorized domains** 이동
3. **Add domain**으로 배포된 Vercel 도메인(예: `allplant.vercel.app`) 추가
