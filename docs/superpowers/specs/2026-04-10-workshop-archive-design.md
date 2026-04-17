# Workshop Archive — 기획 상세 설계 스펙

## 1. 서비스 개요

Workshop Archive는 셀피쉬클럽 워크숍 참여자들의 프로젝트를 한곳에 모아 서로 영감을 주는 커뮤니티형 포트폴리오 사이트다.

- **타겟:** 워크숍 참여자 + 외부 방문자
- **배포 목표:** 워크숍 종료 후 2주 내 MVP 런칭
- **접근 방식:** 풀 MVP 한 번에 (옵션 A)

---

## 2. 핵심 의사결정 요약

| 항목 | 결정 |
|------|------|
| 배포 시점 | 워크숍 종료 후 1~2주 |
| 등록 방식 | 참여자 셀프서비스 (폼) |
| 인증 | 로그인 없음, 관리자 승인으로 품질 관리 |
| 반응 기능 | 영감 반응 + 한 줄 코멘트 |
| 기술 스택 | Next.js + Supabase + Tailwind CSS |
| 이미지 처리 | 파일 업로드 + URL 입력 둘 다 지원 |
| 관리자 도구 | 간단한 관리 페이지 (승인/거절/삭제) |
| 배포 환경 | Vercel (무료 티어) |
| 반응형 | 모바일/데스크탑 동등 대응 |

---

## 3. 사이트 구조

```
Workshop Archive
├── / (메인) ─────────── 프로젝트 갤러리 (카드 그리드)
├── /project/[id] ────── 프로젝트 상세 페이지
├── /submit ──────────── 프로젝트 등록 폼
├── /admin ───────────── 관리자 페이지 (승인/거절/삭제)
└── (공통) ───────────── 헤더, 푸터, 404
```

---

## 4. 데이터베이스 설계

### projects 테이블

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | uuid (PK) | 자동 생성 |
| title | text (필수) | 프로젝트 제목 |
| subtitle | text (필수) | 한 줄 소개 |
| description | text | 상세 설명 (마크다운) |
| thumbnail_url | text (필수) | 썸네일 이미지 URL |
| author_name | text (필수) | 참여자 이름 |
| project_url | text | 외부 프로젝트 링크 |
| tags | text[] | 사용 도구/기술 태그 |
| cohort | text (필수) | 워크숍 기수 |
| status | text | 'pending' / 'approved' / 'rejected' |
| inspiration_count | integer | 영감 카운트 (기본값 0) |
| created_at | timestamptz | 등록 일시 |

### comments 테이블

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | uuid (PK) | 자동 생성 |
| project_id | uuid (FK) | 대상 프로젝트 |
| author_name | text (필수) | 코멘트 작성자 이름 |
| content | text (필수) | 한 줄 코멘트 (100자 제한) |
| created_at | timestamptz | 작성 일시 |

### 이미지 저장소

- Supabase Storage `thumbnails` 버킷
- 업로드 제한: 5MB, jpg/png/webp
- 파일 업로드 시: `thumbnails/{uuid}.{ext}`
- URL 입력 시: 외부 URL을 직접 저장

### 영감 반응

- localStorage 기반 중복 방지 (서버 테이블 없음)
- Supabase RPC로 inspiration_count ±1 토글

### RLS 정책

| 테이블 | SELECT | INSERT | UPDATE | DELETE |
|--------|--------|--------|--------|--------|
| projects | approved만 공개 | 누구나 (pending으로) | 관리자만 | 관리자만 |
| comments | approved 프로젝트 것만 | 누구나 | 불가 | 관리자만 |

---

## 5. 페이지별 상세 동작

### 5-1. 메인 갤러리 (/)

- 카드 그리드: 데스크탑 3열, 태블릿 2열, 모바일 1열
- 카드 구성: 썸네일, 제목, 참여자명, 기수, 한 줄 소개, 영감 수
- 카드 호버: 스케일업 (1.02) + 그림자 강화
- 카드 클릭: 상세 페이지 이동
- 영감 아이콘 클릭: 카드에서 바로 토글
- 필터: 기수별 드롭다운
- 정렬: 최신순 / 영감순 토글
- 빈 상태: 안내 메시지 + 등록 버튼

### 5-2. 프로젝트 상세 (/project/[id])

- 대표 이미지 풀 와이드
- 프로젝트 정보: 제목, 참여자, 기수, 태그 칩
- 설명: 마크다운 렌더링
- 외부 링크: 새 탭 열림
- 영감 버튼: 토글 + 카운트
- 코멘트 섹션: 목록 + 입력 (이름 + 내용, 100자 제한)
- 하단: 같은 기수 프로젝트 3개 추천
- OG 태그: 제목, 소개, 썸네일로 메타 생성

### 5-3. 프로젝트 등록 (/submit)

- 필수 필드: 제목(2~50자), 한 줄 소개(2~100자), 썸네일, 참여자 이름(2~20자), 기수
- 선택 필드: 설명(5000자), 프로젝트 링크, 태그(최대 10개)
- 썸네일: 탭 전환 (파일 업로드 / URL 입력), 미리보기 표시
- 실시간 유효성 검사 (인라인 에러)
- 제출 → pending 상태로 저장 → 확인 화면
- 스팸 방지: 허니팟 필드, 10분 내 중복 제출 방지

### 5-4. 관리자 페이지 (/admin)

- Supabase Auth 로그인 (이메일/비밀번호)
- 관리자 이메일: 환경변수로 관리
- 탭 필터: 대기중 / 승인됨 / 거절
- 액션: 승인, 거절, 삭제 (확인 다이얼로그)
- 미리보기: 모달로 프로젝트 상세 확인

---

## 6. 기술 스택

| 영역 | 기술 | 용도 |
|------|------|------|
| 프레임워크 | Next.js 14+ (App Router) | SSR/SSG, 라우팅 |
| 스타일링 | Tailwind CSS | 반응형 UI |
| DB/Auth/Storage | Supabase | PostgreSQL, 로그인, 이미지 |
| 마크다운 | react-markdown + remark-gfm | 설명 렌더링 |
| 폼 | React Hook Form + Zod | 유효성 검사 |
| 배포 | Vercel | 자동 배포 |
| 패키지 | pnpm | 빠른 설치 |

### 렌더링 전략

| 페이지 | 렌더링 | 이유 |
|--------|--------|------|
| / 갤러리 | SSR (dynamic) | 최신 데이터 반영 |
| /project/[id] | SSR + revalidate 60s | OG 태그 + 캐싱 |
| /submit | CSR | 폼 인터랙션 |
| /admin | CSR | 인증 후 클라이언트 |

---

## 7. 프로젝트 폴더 구조

```
workshop-archive/
├── public/
│   ├── favicon.ico
│   └── og-default.png
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx              # 갤러리
│   │   ├── project/[id]/page.tsx # 상세
│   │   ├── submit/page.tsx       # 등록
│   │   └── admin/page.tsx        # 관리자
│   ├── components/
│   │   ├── ProjectCard.tsx
│   │   ├── ProjectGrid.tsx
│   │   ├── InspirationButton.tsx
│   │   ├── CommentSection.tsx
│   │   ├── SubmitForm.tsx
│   │   ├── ImageUpload.tsx
│   │   ├── FilterBar.tsx
│   │   ├── Header.tsx
│   │   └── Footer.tsx
│   ├── lib/
│   │   ├── supabase.ts
│   │   ├── queries.ts
│   │   └── storage.ts
│   ├── types/
│   │   └── index.ts
│   └── constants/
│       └── cohorts.ts
├── .env.local
├── tailwind.config.ts
├── next.config.ts
├── package.json
└── tsconfig.json
```

---

## 8. 개발 일정 (2주)

### Week 1 (4/14~4/18): 핵심 기능

| 일차 | 마일스톤 |
|------|---------|
| Day 1 (월) | 프로젝트 셋업, DB/Storage/RLS 구성, Vercel 연결 |
| Day 2 (화) | 갤러리 페이지 (카드 그리드, 반응형) |
| Day 3 (수) | 상세 페이지 (마크다운, OG 태그, 추천) |
| Day 4 (목) | 등록 폼 (업로드, 유효성, pending INSERT) |
| Day 5 (금) | 영감 토글 + 코멘트 + 필터/정렬 |

### Week 2 (4/21~4/25): 관리 + 마무리

| 일차 | 마일스톤 |
|------|---------|
| Day 6 (월) | 관리자 페이지 (로그인, 승인/거절/삭제) |
| Day 7 (화) | 디자인 다듬기 (상태 처리, 애니메이션) |
| Day 8 (수) | QA + 버그 수정 (크로스 테스트) |
| Day 9 (목) | 런칭 준비 (도메인, 시드 데이터, 최종 점검) |
| Day 10 (금) | 런칭 + 커뮤니티 공유 + 참여자 안내 |

### 리스크 대응

| 리스크 | 대응 |
|--------|------|
| 디자인 피드백 지연 | Day 7 버퍼 |
| 이미지 업로드 이슈 | URL 입력 폴백 |
| 일정 초과 | 관리자 페이지 → Supabase 대시보드 대체 |
| 코멘트 스팸 | 허니팟 + 글자수 제한 + 관리자 삭제 |

---

*이 스펙은 Superpowers brainstorming 프로세스를 통해 작성되었습니다.*
*2026-04-10 — 기획 파트*
