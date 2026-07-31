# 마인드맵 전용 Firebase 설정 가이드

투두 캘린더와 **별도의** Firebase 프로젝트를 사용합니다. 한 번만 설정하면 됩니다 (약 5분).

## 1. 프로젝트 만들기

1. [Firebase 콘솔](https://console.firebase.google.com) → **프로젝트 추가**
2. 이름: 예) `nahyoni-mindmap` → 계속 (Google 애널리틱스는 꺼도 됨) → 프로젝트 만들기

## 2. Google 로그인 켜기

1. 왼쪽 메뉴 **빌드 → Authentication** → 시작하기
2. **로그인 방법** 탭 → **Google** → 사용 설정 → 저장
3. **설정 탭 → 승인된 도메인**에 앱을 여는 도메인이 있는지 확인
   (GitHub Pages라면 `nahyun824-sys.github.io` 추가)

## 3. Firestore 만들기 + 규칙

1. **빌드 → Firestore Database** → 데이터베이스 만들기 → 위치 기본값 → **프로덕션 모드**로 시작
2. **규칙** 탭에 아래 내용을 통째로 붙여넣고 **게시**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      // 프로필: 누구나(로그인 사용자) 읽기, 본인만 쓰기
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.uid == userId;

      // 마인드맵 데이터: 본인 또는 공유받은 사람이 읽고 쓸 수 있음 (공동편집)
      match /mindmaps/{docId} {
        allow read, write: if request.auth != null && (request.auth.uid == userId
          || exists(/databases/$(database)/documents/users/$(userId)/mmSharedWith/$(request.auth.uid)));
      }
      // 공유 명단: 본인 관리 + 초대링크로 들어온 사람이 자기 자신을 등록
      match /mmSharedWith/{docId} {
        allow read: if request.auth != null;
        allow write: if request.auth != null && (request.auth.uid == userId || request.auth.uid == docId);
      }
      // 내가 공유받은 목록: 본인만
      match /mmShared/{docId} {
        allow read, write: if request.auth != null && request.auth.uid == userId;
      }
    }
  }
}
```

## 4. 웹 앱 등록 + 설정값 붙여넣기

1. **프로젝트 개요 옆 ⚙️ → 프로젝트 설정** → 아래 **내 앱** → **웹(</>)** 아이콘 → 앱 등록 (호스팅 체크 불필요)
2. 나오는 `firebaseConfig` 값(6줄)을 복사
3. `index.html`에서 `const firebaseConfig = {` 부분을 찾아 빈 값들을 복사한 값으로 교체

```js
const firebaseConfig = {
  apiKey: "AIza...",
  authDomain: "nahyoni-mindmap.firebaseapp.com",
  projectId: "nahyoni-mindmap",
  storageBucket: "nahyoni-mindmap.firebasestorage.app",
  messagingSenderId: "...",
  appId: "1:..."
};
```

## 5. 끝

- 앱을 열면 Google 로그인 화면이 나오고, 첫 로그인 때 기존 로컬 데이터가 자동 업로드됩니다.
- 설정 탭 → 친구 초대 링크로 공유하면 상대도 함께 수정할 수 있어요.
- 설정값이 비어 있는 동안에는 이전처럼 기기 로컬 저장으로만 동작합니다.

## 로그인이 안 될 때

로그인 실패 시 로그인 화면에 **오류 코드**가 함께 표시됩니다. 코드별 원인:

| 오류 코드 | 원인과 해결 |
|---|---|
| `auth/operation-not-allowed` | Firebase 콘솔 → Authentication → 로그인 방법에서 **Google**이 꺼져 있음. 사용 설정으로 바꾸면 됩니다. |
| `auth/unauthorized-domain` | 앱을 여는 주소가 **승인된 도메인**에 없음. Authentication → 설정 → 승인된 도메인에 추가 (GitHub Pages라면 `nahyun824-sys.github.io`). |
| `auth/redirect-no-result` | 구글 로그인 화면에서 돌아왔지만 로그인이 완료되지 않음. 브라우저의 쿠키 차단이나 "사이트 간 추적 방지"가 원인인 경우가 많습니다. |
| `auth/web-storage-unsupported` | 시크릿 모드이거나 저장소가 차단됨. 일반 창에서 다시 시도. |
| `auth/popup-blocked` | 팝업 차단. 앱이 자동으로 리다이렉트 방식으로 다시 시도하므로 보통 그냥 로그인됩니다. |

앱은 환경에 따라 로그인 방식을 자동으로 고릅니다. 홈 화면에 설치한 상태(PWA)와
iOS에서는 팝업이 결과를 돌려주지 못하므로 처음부터 **리다이렉트** 방식으로 로그인하고,
그 외 환경에서는 팝업을 먼저 쓰되 막히면 리다이렉트로 자동 전환합니다.
