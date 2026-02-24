## 1. 문제 상황

- 일부 페이지에서 `toast.success()`, `toast.error()` 로직은 타는데
    
    **알림 UI가 아예 안 뜨는 문제 발생**
    
- 어떤 페이지는 정상 동작, 어떤 페이지는 아예 안 뜸
- 페이지마다 `<Toaster />`를 넣어둔 상태였음 (중복/비일관 구조)

---

## 2. 원인

- **Sonner의 `<Toaster />`는 화면 트리에 1번만 존재해야 함**
- Next.js App Router 구조에서
    
    특정 라우트 트리(`/vendor`, `(main)` 등)가
    
    **RootLayout의 Toaster를 안 타는 구조**였음
    
- 페이지별로 `<Toaster />`를 넣으면:
    - 레이아웃이 다를 때 안 뜨는 페이지 생김
    - hydration / 마운트 타이밍 꼬일 수 있음
    - 유지보수성 최악

---

## 3. 해결 전략

> 📌 Toaster를 전역 Layout에 1번만 설치
> 
> 
> 📌 **각 페이지에서는 `toast`만 사용**
> 
> 📌 **페이지별 `<Toaster />` 전부 제거**
> 

---

## 4. 구현 내용

### 4-1. `Providers.tsx` 생성

**frontend/src/app/providers.tsx**

```tsx
'use client';

import { Toaster } from 'sonner';

export default function Providers() {
  return <Toaster position="top-center" richColors />;
}

```

---

### 4-2. `RootLayout`에 Providers 연결

**frontend/src/app/layout.tsx**

```tsx
import Providers from './providers';

...

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="ko">
      <head>
        <script src="//t1.daumcdn.net/mapjsapi/bundle/postcode/prod/postcode.v2.js" async></script>
      </head>
      <body
        className={`${geistSans.variable} ${geistMono.variable} ${inter.variable} ${jetbrainsMono.variable} antialiased`}
      >
        <AuthProvider>
          <Providers />   {/* 🔔 Sonner 전역 설치 */}
          {children}
        </AuthProvider>
      </body>
    </html>
  );
}
```

---

## 5. 각 페이지에서 변경 사항

### 5-1. import 정리

기존:

```tsx
import { toast,Toaster } from 'sonner';
```

변경:

```tsx
import { toast } from 'sonner';
```

---

### 5-2. JSX에서 Toaster 제거

기존:

```tsx
<Toaster position="top-center" richColors />
```

변경:

```tsx
// ❌ 완전 삭제
```

---

## 6. 최종 구조 요약

- `<Toaster />`
    
    → `RootLayout`에서 **1번만 렌더링**
    
- 각 페이지
    
    → `toast.success()`, `toast.error()`만 사용
    
- `/vendor`, `/main`, 기타 모든 라우트
    
    → 동일한 Toaster 인스턴스 공유
    

---

## 7. 체크 포인트

- ✅ 모든 페이지에서 toast 정상 출력됨
- ✅ 레이아웃 경로 달라도 toast 일관 동작
- ✅ 중복 Toaster 제거
- ✅ 유지보수성 개선