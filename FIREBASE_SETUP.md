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
