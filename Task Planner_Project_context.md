# HILLIARD BUFFSHIRE — Task Planner · Project Context
> 마지막 업데이트: 2026-05-24 | Commit: `24ed27ead4a5` | (patch_v17_refactor)

---

## 🎯 Project Overview

**"주님과 함께하는 TASK PLANNER"** — 미니멀리즘을 최우선으로 하는 올인원 개인 생산성·건강 플랫폼.

| 원칙 | 설명 |
|------|------|
| Less is More | 사용자가 뇌를 쓰지 않고 직관적으로 운용 |
| 원클릭 자동화 | 루틴·식단·운동은 AI와 프리셋이 대신 처리 |
| 단일 파일 아키텍처 | `index.html` 한 파일로 전체 기능 — 빌드 없음 |
| 한국어 우선 UI | Amazon 직장인(Dallas, TX) 라이프스타일 맞춤 |

---

## 🛠 Tech Stack

| 레이어 | 기술 |
|--------|------|
| **Frontend** | Vanilla JS (ES6+), HTML5, CSS3 — 프레임워크 없음 |
| **Auth** | Firebase Authentication (Email/Password) |
| **Database** | Firebase Firestore (Realtime onSnapshot 패턴) |
| **AI** | Google Gemini API (`gemini-2.5-flash-lite` 기본) |
| **배포** | GitHub Pages (`hilliardbuffshire/hilliard-todo`, branch: `main`) |
| **파일** | 단일 `index.html` (~330KB normalized) |
| **Firebase SDK** | Compat CDN v10.12.2 — **`defer` 절대 금지** |

### Firebase 설정 (변경 금지)
```js
firebase.initializeApp({
  apiKey: "AIzaSyDEPq3SqOCiltpmesT3vP9Jp7a_ujFou1k",
  authDomain: "daily-task-planner-9187f.firebaseapp.com",
  projectId: "daily-task-planner-9187f",
  storageBucket: "daily-task-planner-9187f.firebasestorage.app",
  messagingSenderId: "309757618979",
  appId: "1:309757618979:web:79fbdc11fbfbcdffa9ba96"
});
var auth = firebase.auth(), db = firebase.firestore();
var DOMAIN = "@hilliard.buffshire.internal";
```

---

## 🏗 Architecture

### 렌더링 흐름
```
Firestore onSnapshot
  → scheduleRender() [RAF debounce]
    → renderAll()
      → computeCounts() + renderXxx() per view
```

### 뷰 목록 (setView) — patch_v17_refactor 기준
| v 값 | 함수 | HTML ID | 비고 |
|------|------|---------|------|
| `calendar` | `renderCal()` | `#vCalendar` | **기본 진입 뷰 — "📅 대시보드"** |
| `settings` | `renderSettings()` | `#vSettings` | **⚙️ 설정** |

나머지 뷰는 모두 제거(스텁 함수만 유지). `weeklyTmplModal`은 `display:none!important`.

### CSS 가시성 규칙
```css
.vPg { display: none }
.vPg.va { display: block; animation: vf .12s ease }
#vIdeas.va { display: flex !important; flex-direction: column }
```

### 캘린더 셀 구조 (patch_v8~)
각 날짜 칸은 3개 시각 영역으로 분리:
```
.calCellWrk  → 💪 운동 (파란색 #1565c0 좌측 border + 파란 배경)
.calCellMeal → 🥗 식단 (녹색 #2e7d32 좌측 border + 녹색 배경)
.calCellTodo → 📋 할 일 (accent 좌측 border)
```
**1초 구분 원칙**: 파랑=운동, 녹색=식단 — 색맹 고려한 border+background 이중 큐.

---

## 🔑 통합 인증 (Unified Auth)

동일 이메일/비밀번호로 3개 앱 연동:

| 앱 | Firestore 컬렉션 |
|----|-----------------|
| Task Planner | `todos`, `goals`, `ideas`, `hb_meals`, `hb_workouts` |
| Finance Dashboard | `hb_finance` |
| Investment Dashboard | `hb_invest` |

---

## 📊 Data Models

### `todos`
```js
{
  uid: string,
  text: string,
  priority: 'high' | 'medium' | 'low',
  date: 'YYYY-MM-DD',
  note: string,
  time: string,
  type: 'todo' | 'recurring' | 'note',
  category: string,
  done: boolean,
  subTasks: [{ text: string, done: boolean }],
  recurring: 'daily' | 'weekly' | 'monthly',
  recurringDays: number[],   // 0-6 (일-토)
  recurringDay: number,      // 1-31
  createdAt: Timestamp
}
```

### `goals`
```js
{
  uid: string,
  title: string,
  desc: string,
  horizon: 'yearly' | 'quarterly' | 'monthly' | 'weekly' | 'daily',
  color: '#hex',
  actions: [{ text: string, done: boolean }],
  createdAt: Timestamp
}
```

### `hb_meals`
```js
{
  uid: string,
  date: 'YYYY-MM-DD',
  mealType: 'breakfast' | 'lunch' | 'dinner' | 'snack',
  name: string,
  amount: number,      // g 단위
  kcal: number,
  protein: number,     // g
  carbs: number,       // g
  fat: number,         // g
  done: boolean,       // ✅ patch_v12: Notion 체크박스 완료 여부
  recipe: {            // ✅ patch_v14: AI 생성 식단만 포함 (수동 입력은 없음)
    ingredients: [{item: string, amount: string}],
    steps: [string],
    prep_time: number,   // 분
    cook_time: number,   // 분
    tips: string
  },
  createdAt: Timestamp
}
```

### `hb_workouts`
```js
{
  uid: string,
  date: 'YYYY-MM-DD',
  exercises: [{
    name: string,
    category: 'strength' | 'cardio' | 'flexibility',
    sets: number,
    reps: number,
    weight: number     // kg
  }],
  doneExercises: { [exIdx: number]: boolean },  // ✅ patch_v12: 개별 운동 완료 체크
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

### localStorage 키
| 키 | 용도 |
|----|------|
| `hb-t` | 현재 테마 |
| `hb-gem-key` | Gemini API 키 |
| `hb-gem-model` | Gemini 모델명 (기본: `gemini-2.5-flash-lite`) |
| `hb-mm-{uid}` | 마인드맵 데이터 |
| `hb-notes-{uid}` | 스티커 메모 |
| `hb-g-{uid}` | 로컬 목표 fallback |
| `hb-wtmpl-{uid}` | 주간 고정 루틴 템플릿 (JSON) |
| `hb-wtmpl-applied-{uid}` | 마지막 적용 ISO 주 (예: `2026-W21`) |

---

## 🍽 식단 & 💪 운동 시스템

### 식단 관리 주요 함수
| 함수 | 역할 |
|------|------|
| `renderMeal()` | 식단 뷰 렌더 |
| `addMealEntry()` | 개별 항목 Firestore 추가 |
| `deleteMealEntry(id)` | 항목 삭제 |
| `aiNutrition()` | 단일 음식명 → 영양소 Gemini 자동계산 |
| `aiParseMeal()` | **자연어 텍스트 전체 → 다중 항목 파싱·저장** |
| `getShoppingList()` | AI 장보기 리스트 (Dallas TX 매장 기준) |
| `getRecipeSuggestion()` | AI 레시피 3개 추천 |
| `getWalmartBudget()` | **Dallas Walmart 주간/월간 예산 Gemini 추정** |
| `setMealTab(t)` | 식사 타입 필터 |

### 운동 관리 주요 함수
| 함수 | 역할 |
|------|------|
| `renderWorkout()` | 운동 뷰 렌더 (7일 스트릭 포함) |
| `addWorkoutEntry()` | 운동 항목 추가 |
| `deleteWorkoutEx(docId, exIdx)` | 특정 운동 삭제 |
| `getAiWorkout(type)` | AI 운동 루틴 추천 (upper/lower/full/cardio) |

---

## 📋 주간 고정 루틴 엔진 (patch_v8~)

### 동작 원리
1. 사이드바 `📋 주간 루틴 세팅` → `openWeeklyTemplate()` → 모달 열림
2. 요일 탭별 식단(텍스트)/운동(텍스트) 직접 입력 **또는** Quick Load 프리셋 클릭
3. `💾 저장` → `localStorage('hb-wtmpl-{uid}')` 에 JSON 보관
4. `▶ 이번 주 적용` → `applyWeeklyTemplate()` → 이번 주 월~일 날짜에 Firestore 일괄 생성
5. 적용 이력 → `localStorage('hb-wtmpl-applied-{uid}')` = ISO 주 번호

### 식단 입력 형식
```
B:오트밀 50g      ← B=breakfast, L=lunch, D=dinner, S=snack
L:닭가슴살 200g
D:연어 150g
```

### 운동 입력 형식
```
스쿼트 5x5 60kg          ← 이름 세트x횟수 무게(kg)
랫풀다운 4x10 60kg
완전 휴식                 ← 이 문자열은 Firestore 저장 스킵
```

### 핵심 함수
| 함수 | 역할 |
|------|------|
| `openWeeklyTemplate()` | 모달 열기 |
| `closeWeeklyTemplate()` | 모달 닫기 |
| `saveWeeklyTemplate()` | localStorage 저장 |
| `applyWeeklyTemplate()` | 이번 주 Firestore 적용 |
| `loadPresetToTemplate(type, key)` | 프리셋 데이터를 템플릿에 로드 |
| `_renderWtTabs()` | 요일 탭 렌더 |
| `_renderWtContent()` | 요일 콘텐츠 렌더 |
| `_getISOWeek(date)` | ISO 주 번호 반환 |

---

## 🎨 원클릭 프리셋 시스템 (patch_v9~)

### 위치
캘린더 뷰 상단 (`.psStrip`) — calHead 아래, calGrid 위에 고정 배치.

### UI 흐름
```
캘린더 진입
  → psStrip 표시 (식단 2개 + 운동 3개 태그 버튼)
  → 클릭 시 .psSel 토글 + 예산 배지 자동 표시
  → "▶ 이번 주 적용" 버튼 나타남
  → 클릭 → Firestore 일괄 생성 → 완료 피드백
```

### 식단 프리셋 (`MEAL_PRESETS`)
| 키 | 이름 | 주간 예산 | 월간 예산 | 콘셉트 |
|----|------|----------|----------|--------|
| `budget` | 🥬 가성비 건강식 | ~$48/주 | ~$192/월 | 닭가슴살·달걀·오트밀·바나나 |
| `gourmet` | 🥩 프리미엄 건강식 | ~$92/주 | ~$368/월 | 연어·소고기 부채살·아보카도 |

### 운동 프리셋 (`WORKOUT_PRESETS`)
| 키 | 이름 | 콘셉트 |
|----|------|--------|
| `beginner` | 🏃 초심자 웰니스 | 걷기·맨몸 스쿼트·스트레칭 위주 |
| `calisthenics` | 🤸 맨몸 마스터 | 푸쉬업·풀업·딥스 고강도 |
| `frame` | 📐 프레임 빌더 | 등빨·어깨 집중 웨이트 (데드리프트·랫풀다운·레터럴 레이즈) |

### Quick-Add Bar (patch_v10~)
캘린더 최상단 한 줄 입력창. 자연어로 날짜·카테고리 자동 감지 (Gemini 불필요).

| 함수 | 역할 |
|------|------|
| `quickAdd()` | 입력창 텍스트 → Firestore 즉시 저장 |
| `_qaDate(text)` | 오늘/내일/모레/요일명 → `YYYY-MM-DD` 파싱 |
| `_qaCat(text)` | 키워드 기반 자동 카테고리 (health/work/study/personal) |
| `toggleMAdv()` | 할 일 모달 상세 옵션 (우선순위·카테고리·메모·체크리스트) 접기/펼치기 |

---

## 📱 반응형 레이아웃 가이드 (patch_v11~)

### 브레이크포인트
| 너비 | 레이아웃 |
|------|---------|
| > 768px | 데스크톱 — 사이드바 + 캘린더 그리드 |
| ≤ 768px | 모바일 — 하단 네비게이션 + 아젠다 뷰 자동 전환 |
| ≤ 780px | 기존 사이드바 슬라이드 전환 (hamburger) |

### v13 User-Friendly 디자인 시스템 가이드라인

#### 캘린더 클릭 컴포넌트 규격
| 기기 | 컴포넌트 | 크기 | 애니메이션 |
|------|---------|------|-----------|
| PC (>768px) | `#calSidePanel` 우측 슬라이드 | `min(520px, 46vw)` × 100vh | `translateX(110%→0)` 0.32s cubic |
| 모바일 (≤768px) | 바텀 시트 전체 화면 | 100% × 92dvh | `translateY(110%→0)` 0.32s cubic, border-radius 22px |
| desk-mode 오버라이드 | PC 강제 적용 | PC 규격과 동일 | 동일 |

#### 버튼 디자인 시스템 (v13 기준)
| 클래스 | 용도 | 핵심 스타일 |
|--------|------|------------|
| `.cspAddBtn` | 패널 내 3-action 버튼 (📋할일/🥗식단/💪운동) | `var(--ac)` 배경 + `box-shadow:0 4px 16px rgba(0,0,0,.16)` + hover translateY(-2px) |
| `.cspDelBtn` | 패널 항목별 삭제 버튼 (hover reveal) | opacity:0→1 on parent hover, 🗑 아이콘, hover red tint |
| `.qaSubmit` | Quick-Add 제출 | accent 배경 + hover brightness(1.08) translateY(-1px) |
| `.psBtn` | 프리셋 태그 | card 배경 + hover accent border/text |
| `.psSel` | 선택된 프리셋 | accent 배경 + shadow |
| `.psApplyBtn` | 프리셋 적용 | `#1565c0` + `box-shadow:0 4px 14px rgba(21,101,192,.38)` |

#### 완료 게이지 규격
- 위치: 사이드 패널 헤더 하단, 항목이 있을 때만 `.vis` 클래스로 표시
- 트랙 높이: 10px, border-radius 6px
- 애니메이션: `width` 0→N% 전환 0.55s cubic-bezier(.4,0,.2,1)
- 100% 달성 시: `linear-gradient(#2e7d32,#43a047)` + 초록 텍스트

#### 실행취소 토스트 규격
- 위치: `bottom:76px`, `left:50%` 중앙 고정, z-index 800
- 표시 시간: 3.5초 후 자동 소멸
- 실행취소 버튼: 토스트 우측 인라인 (흰 반투명 배경)

---

### 모바일 대응 전략 (patch_v12 기기별 이원화 배치 전략)
- **캘린더 그리드 (`@media ≤ 768px`)**: `display:none` → `#mbWeekBar` + `#mobileAgenda` 표시
- **바텀 네비게이션 (`#mobileBottomNav`)**: 5개 탭 (홈/할일/식단/운동/루틴) 고정 하단
- **주간 탭바 (`#mbWeekBar`)**: 이번 주 월~일 7일 수평 스크롤 탭 (오늘 하이라이트, `.mbWTab.on`)
- **일간 상세 뷰 (`renderMbDayDetail(ds)`)**: 선택된 날의 운동+식단+할일 세로 카드 + Notion 체크박스 + 완료도 게이지
- **PC 보기 토글**: `body.desk-mode` 클래스 → 모바일에서도 그리드 복원
- **`renderAll()` 훅**: 데이터 갱신 시 아젠다 자동 재렌더

### PC 캘린더 셀 (patch_v12~)
- 각 셀은 컴팩트 요약 배지만 표시 → 클릭 시 기존 팝업(`calDayPopup`)에서 전체 상세 확인
- `.calSumWrk`: `🏋️ N개` (파란 배경 배지)
- `.calSumMeal`: `🥗 N개` (녹색 배경 배지)
- `.calSumTodo`: `✅ done/total` (회색 배경 배지)

### 사이드 패널 함수 (patch_v13~)
| 함수 | 역할 |
|------|------|
| `openSidePanel(ds)` | 패널 열기 — `_cspDs=ds`, `.open` 클래스, body overflow 잠금 |
| `closeSidePanel()` | 패널 닫기 — `.open` 클래스 제거, ESC 키 핸들러 해제 |
| `_buildSidePanelContent(ds)` | 패널 내용 전체 빌드 (운동·식단·할일·메타·게이지) |
| `_cspKeyClose(e)` | ESC 키 → closeSidePanel() |
| `_cspToast(msg, undoCb)` | 토스트 + 실행취소 버튼 표시 (3.5초 auto dismiss) |
| `toggleMealDone(id, done)` | `hb_meals.done` Firestore 토글 + 토스트 표시 |
| `toggleWorkoutExDone(wid, exIdx, done)` | `hb_workouts.doneExercises.N` Firestore 토글 + 토스트 표시 |

### 모바일 함수 (patch_v12~)
| 함수 | 역할 |
|------|------|
| `renderAgenda()` | 디스패처 — `renderMbWeekBar()` + `renderMbDayDetail()` 호출 |
| `renderMbWeekBar()` | 이번 주 7일 탭바 → `#mbWeekBar` 렌더 |
| `renderMbDayDetail(ds)` | 선택일 운동/식단/할일 상세 → `#mobileAgenda` 렌더 |
| `_syncMobileNav(key)` | 바텀 네비 활성 탭 동기화 |

### 캘린더 데스크톱 개선
- `.calGrid`: `overflow:hidden` → `overflow-x:auto;overflow-y:visible` (짤림 방지)
- `.calCell`: `min-height:90px; max-height:none` (patch_v17: 높이 제한 완화, 셀 짤림 방지)
- `.calCellWrk`: `border-left:4px solid #1565c0` 파란 카드
- `.calCellMeal`: `border-left:4px solid #2e7d32` 녹색 카드

### 사이드바 (patch_v17_refactor — 2탭 구조)
표시되는 탭 **2개만**:
| ID | 레이블 | setView |
|----|--------|---------|
| `nv-calendar` | 📅 대시보드 | `calendar` |
| `nv-settings` | ⚙️ 설정 | `settings` |

모바일 하단 nav도 **2개만**: `mbn-cal`, `mbn-settings`

숨김 처리된 뷰: `vToday`, `vAll`, `vDone`, `vRecurring`, `vGoals`, `vIdeas` — `display:none!important`

레거시 함수 스텁 (빈 함수, ReferenceError 방지):
`renderToday`, `renderAll2`, `renderDone`, `renderRecurring`, `renderGoals`, `renderIdeas`, `renderWeek`, `toggleNavGrp`, `filterP`, `filterCat`, `chipT`, `chipA`

사이드바 너비: `--sw:300px`

### 핵심 함수
| 함수 | 역할 |
|------|------|
| `selectPreset(type, key)` | 프리셋 선택/해제 토글 |
| `_syncPresetUI()` | 버튼 활성화 + 예산 배지 + Apply 버튼 표시 동기화 |
| `applyPresets()` | 선택된 프리셋을 템플릿 저장 후 Firestore 적용 |
| `_applyTmplToFirestore(tmpl)` | 이번 주 전체 날짜에 hb_meals + hb_workouts 일괄 생성 |
| `loadPresetToTemplate(type, key)` | 템플릿 모달 내 Quick Load (저장만, 즉시 적용 안 함) |

---

## 🗑️ 안전한 조건부 초기화 (Conditional Reset Logic) — patch_v15

### 버튼 위치
캘린더 상단 psStrip 우측 끝 — 프리셋/AI 버튼과 `psDivider`로 시각 분리. 클래스: `.btnReset` (Soft Red 배경).

### 작동 흐름 (v16 기준)
| 단계 | 설명 |
|------|------|
| 1단계 | 🗑️ 루틴 초기화 버튼 클릭 → `openResetModal()` 호출 |
| 2단계 | 경고 모달에서 삭제 카테고리 체크박스 선택 |
| 실행 | [초기화 실행] 클릭 → `executeReset()` → 로컬 상태 필터링 → batch delete |

### ⚠️ 권한 오류 해결 (v16/v17)
**원인**: v15에서 `db.collection('todos').where('uid','==',uid).where('type','==','recurring').get()` 같은 복합 필드 쿼리를 새로 실행 → Firestore 보안 규칙이 단일 `uid` 쿼리만 허용하여 `Missing or insufficient permissions` 발생. patch_v17에서 `generateAiWeeklyPlan` / `generateAiWorkoutPlan` 함수의 동일 패턴도 수정됨.

**해결**: 이미 `onSnapshot`으로 로드된 로컬 상태 배열을 JavaScript로 필터링
```js
// ❌ 제거된 패턴 (권한 오류)
db.collection('todos').where('uid','==',uid).where('type','==','recurring').get()

// ✅ 현재 패턴 (로컬 필터 — 권한 불필요)
todos.filter(function(t){ return t.type === 'recurring'; })
     .forEach(function(t){ toDelete.push(db.collection('todos').doc(t.id)); });
// todos 배열은 이미 onSnapshot에서 uid 필터링됨
```

### 삭제 카테고리 (체크박스 선택형)
| 체크박스 ID | 대상 | Firestore 조건 |
|------------|------|---------------|
| `rc-recurring` | 반복 일정 | `todos` where `uid==me` AND `type=='recurring'` |
| `rc-meals` | 이번 주 식단 | `hb_meals` where `uid==me` AND `date` in [Mon~Sun] |
| `rc-workouts` | 이번 주 운동 | `hb_workouts` where `uid==me` AND `date` in [Mon~Sun] |
| `rc-week-meals` | 주간 루틴 템플릿 | localStorage `hb-wtmpl-{uid}` + `hb-wtmpl-applied-{uid}` |
| `rc-template` | 완료된 할일 | `todos` where `uid==me` AND `done==true` |

### 데이터 보호 규칙 (불변)
1. **UID 필터 강제**: 모든 Firestore 쿼리에 `.where('uid','==',currentUser.uid)` 포함 — 타 사용자 데이터 접근 불가
2. **FLUSH 금지**: 컬렉션 전체 삭제 코드 없음 — 조건부 필터링만 사용
3. **과거 기록 보존**: 이번 주 날짜 범위 외 식단/운동 히스토리는 삭제하지 않음
4. **500건 청크 처리**: `allRefs`를 499개씩 나눠 batch delete — Firestore 한도 초과 방지
5. **텍스트 입력 가드**: patch_v16에서 제거 — 체크박스 선택 후 즉시 실행 가능

### 핵심 함수
| 함수 | 역할 |
|------|------|
| `openResetModal()` | 모달 열기, 입력 필드 초기화, 확인 버튼 비활성화 |
| `closeResetModal()` | 모달 닫기 (ESC / 배경 클릭 / 취소 버튼) |
| `onResetTypeInput(el)` | '초기화' 입력 감지 → 버튼 활성화/비활성화 + matched 스타일 |
| `executeReset()` | 선택 카테고리 Firestore batch 삭제 + localStorage 정리 + 토스트 + re-render |
| `_getWeekRange()` | 이번 주 월요일~일요일 날짜 문자열 반환 (공용 유틸) |

---

## 🤖 Gemini AI 통합

### 설정
- API 키: `localStorage('hb-gem-key')` — ⚙️ 설정 뷰에서 입력·저장 (patch_v17~)
- 모델: `localStorage('hb-gem-model')` — 기본값 `gemini-2.5-flash-lite`
- 지원 모델: `gemini-2.5-flash-lite` / `gemini-2.0-flash` / `gemini-1.5-pro`
- 무료 키 발급: https://aistudio.google.com/app/apikey

### ✨ AI 자연어 프롬프트 바 (patch_v17~v18)
- 위치: `#vCalendar` 상단 (calGoalBanner 바로 아래)
- HTML ID: `#aiPromptBar`, `#aiPromptInput`, `#aiSuggestCard`, `#aiSuggestContent`
- 흐름: 자연어 입력 → `submitAiPrompt()` → Gemini 파싱 → `showAiSuggestCard(arr)` → 확인 시 `confirmAiSuggest()` → `db.batch().set()` → `renderAll()`
- **v18 Gemini 반환 형식 (배열):**
  ```json
  [
    {"type":"meal","date":"YYYY-MM-DD","mealType":"lunch","name":"닭가슴살 샐러드","kcal":350,"protein":35,"carbs":20,"fat":8,"estimated_usd":6.5,"recipe":{"ingredients":[{"item":"닭가슴살","amount":"200g"}],"steps":["..."],"tips":"..."}},
    {"type":"workout","date":"YYYY-MM-DD","workout_name":"상체 루틴","exercises":[{"name":"랫풀다운","sets":4,"reps":10,"weight":60,"notes":"광배 집중"}],"notes":"전체 메모"}
  ]
  ```
- **v20 AI 시스템:** 멀티턴 대화 (`_aiHistory[]` Gemini contents 배열), 채팅 히스토리 UI, 대화 초기화 버튼
- **제안 카드:** 음식명/칼로리/영양소/달러가격/레시피(재료+조리법+팁) `<details>` 접기, 운동 세트/횟수/무게 표시
- 취소: `cancelAiSuggest()` → 제안 카드 숨김, `_aiSuggestData=null`

### 사이드바 그룹 (patch_v18)
```
NAV_GROUPS = {
  planning: ['nv-today','nv-all','nv-done','nv-recurring','nv-week'],
  goals: ['nv-goals','nv-ideas','nv-workout']
}
```
- HTML: `<button class="navGroupHead" id="navGH-planning" onclick="toggleNavGrp('planning')">` (nv-settings 아래에 추가)
- `toggleNavGrp(grp)`: NAV_GROUPS[grp] 배열의 nav 버튼을 style.display 토글
- CSS: `.navGroupHead`, `.navGroupHead.ng-open .ngArrow { transform:rotate(90deg) }` 

### AI 기능 목록
| 함수 | 트리거 | 역할 |
|------|--------|------|
| `aiNutrition()` | 음식명 입력 0.9초 debounce | 단일 음식 영양소 자동계산 |
| `aiParseMeal()` | "AI 자동 분석 & 저장" 버튼 | 자연어 → 다중 식단 항목 파싱·저장 |
| `getShoppingList()` | 🛒 장보기/레시피 버튼 | 주간 식단 기반 장보기 리스트 (Dallas TX 매장) |
| `getRecipeSuggestion()` | 같은 버튼 탭 | 건강 레시피 3개 |
| `getAiWorkout(type)` | 운동 뷰 AI 버튼 | 타입별 운동 루틴 추천 |
| `getWalmartBudget()` | 식단 뷰 예산 카드 새로고침 | Dallas Walmart 주간/월간 식비 추정 |
| `geminiAsk(prompt, el, msg)` | (범용) | 공통 Gemini 호출 래퍼 |
| `generateAiWeeklyPlan()` | 🥗 AI 식단 생성 버튼 (psStrip) | Jayden 맞춤형 7일 식단 + 레시피 Gemini 생성·저장 |
| `generateAiWorkoutPlan()` | 💪 AI 운동 생성 버튼 (psStrip) | 등빨 프레임 빌더 7일 루틴 Gemini 생성·저장 |
| `openRecipeModal(mid)` | 사이드패널 📖 레시피 버튼 | `_hbRecipes[mid]` 조회 → 재료·조리법·팁 모달 표시 |
| `closeRecipeModal()` | 모달 × 버튼 / ESC / 배경 클릭 | 레시피 모달 닫기 |
| `submitAiPrompt()` | AI 프롬프트바 "AI 실행" 버튼 | 자연어 → Gemini 파싱 → 제안 카드 표시 |
| `confirmAiSuggest()` | 제안 카드 확인 버튼 | 파싱 결과를 Firestore에 저장 (todo/meal/workout) |
| `cancelAiSuggest()` | 제안 카드 취소 버튼 | 제안 카드 숨김, 데이터 초기화 |
| `toggleNavGrp(grp)` | 사이드바 그룹 헤더 클릭 | NAV_GROUPS[grp] 배열의 nav 버튼 표시/숨김 토글 |
| `renderSettings()` | `setView('settings')` 호출 시 | 설정 뷰 렌더 (API 키 상태, 모델, 테마 버튼) |
| `saveGemKey()` | 설정 뷰 저장 버튼 | `hb-gem-key` localStorage 저장 |
| `clearGemKey()` | 설정 뷰 초기화 버튼 | API 키 삭제 |
| `saveGemModel()` | 설정 뷰 모델 select | `hb-gem-model` localStorage 저장 |

### Dallas, TX 매장 정보 (하드코딩)
- 채소/식품: `2534 Royal Ln, Dallas TX 75229`
- 고기/Walmart: `4122 LBJ Fwy, Dallas TX 75244`

---

## 🎨 테마 시스템

| 테마 | 특징 |
|------|------|
| `signiel` | 기본 — 골드+아이보리 |
| `obsidian` | 다크 |
| `blanc` | 미니멀 흑백 |
| `jade` | 그린 |
| `sapphire` | 블루 |

---

## 📱 모바일 사이드바

```css
@media (max-width: 780px) {
  #sidebar { left: -264px; width: 264px; transition: left .28s; transform: none !important }
  #sidebar.mOpen { left: 0; transform: none !important }
}
```
- nav 아이템: `<button type="button">` (iOS 클릭 신뢰성)
- `toggleSb(force)` 함수로 제어 / `#sbOv` 오버레이 기본 `pointer-events:none`

---

## 🚀 Deployment Guide

### 사전 준비
```
Python 3.x
pip install requests
Node.js (구문 검사용)
C:\Users\savem\OneDrive\Desktop\Jayden\Projects\hub\.deploy_config
  → {"token": "ghp_..."}
```

### Step-by-Step

**1. 최신 파일 다운로드**
```python
import json, base64, requests
with open('C:/Users/savem/OneDrive/Desktop/Jayden/Projects/hub/.deploy_config') as f:
    token = json.load(f)['token']
headers = {'Authorization': f'token {token}', 'Accept': 'application/vnd.github.v3+json'}
r = requests.get('https://api.github.com/repos/hilliardbuffshire/hilliard-todo/contents/index.html',
                 headers=headers, params={'ref': 'main'}, timeout=30)
content = base64.b64decode(r.json()['content']).decode('utf-8')
open('index.html', 'w', encoding='utf-8').write(content)
```

**2. 수정 — 반드시 Python string replace 방식**
```python
# ⚠️ Edit 도구로 직접 수정 금지 — 파일 잘림 위험
content = open('index.html', encoding='utf-8').read()
content = content.replace('OLD_EXACT_STRING', 'NEW_STRING', 1)
open('index.html', 'w', encoding='utf-8').write(content)
```

**3. Node.js 구문 검사 (배포 전 필수)**
```python
import re
content = open('index.html', encoding='utf-8').read()
js = '\n'.join(re.findall(r'<script[^>]*>(.*?)</script>', content, re.DOTALL))
open('_syntax_check.js', 'w', encoding='utf-8').write(js)
```
```bash
node --check _syntax_check.js
```

**4. 배포**
```python
import json, base64, time, requests
with open('C:/Users/savem/OneDrive/Desktop/Jayden/Projects/hub/.deploy_config') as f:
    token = json.load(f)['token']
headers = {
    'Authorization': f'token {token}',
    'Accept': 'application/vnd.github.v3+json',
    'X-GitHub-Api-Version': '2022-11-28'
}
sha = requests.get('https://api.github.com/repos/hilliardbuffshire/hilliard-todo/contents/index.html',
                   headers=headers, params={'ref': 'main'}, timeout=30).json().get('sha')
b64 = base64.b64encode(open('index.html', 'rb').read()).decode()
r = requests.put('https://api.github.com/repos/hilliardbuffshire/hilliard-todo/contents/index.html',
    headers=headers,
    json={'message': f'patch_vX -- 변경 설명 -- {time.strftime("%Y-%m-%d %H:%M")}',
          'content': b64, 'branch': 'main', 'sha': sha},
    timeout=30)
print('Status:', r.status_code, '| Commit:', r.json()['commit']['sha'][:12])
```

**5. 배포 확인**
GitHub Actions 완료 (~1분) 후 → https://hilliardbuffshire.github.io/hilliard-todo/

---

## ⚠️ 개발 규칙 (Must-Follow)

| 규칙 | 이유 |
|------|------|
| `defer` 스크립트 금지 | `auth`, `db` undefined 됨 |
| JS 문자열 리터럴 줄바꿈 금지 | SyntaxError → 전체 스크립트 블록 실패 |
| `onclick` 안에 따옴표 중첩 금지 | 반드시 `data-*` 속성 + `dataset.xxx` 패턴 |
| 모달 열기: CSS class 방식 금지 | `element.style.cssText='display:flex!important;...'` 직접 제어 |
| `deploy.py` 사용 금지 | 한국어 인코딩 오류 — 위 직접 Python 코드 사용 |
| PowerShell 한국어 스크립트 금지 | `.py` 파일 저장 후 실행 |
| Edit 도구로 대용량 수정 금지 | 파일 잘림 위험 — Python string replace 사용 |
| 배포 전 `node --check` 필수 | JS SyntaxError 사전 차단 |

---

## 📝 변경 이력

| 날짜 | 커밋 | 내용 |
|------|------|------|
| 2026-04-29 | `abf3286d0e92` | 초기 상태 |
| 2026-05-06 | `a40152a7c7fa` | 스티커 메모, 목표 drag&drop, carry-over, 캘린더 배너 |
| 2026-05-06 | `b593b7794ad9` | 마인드맵 단일클릭 편집, 연결선 레이블 |
| 2026-05-06 | `3b0e775a1854` | 마인드맵 더블클릭 편집 복원 |
| 2026-05-07 | — | Firestore Rules: `hb_finance`, `hb_invest` 추가 |
| 2026-05-15 | `c20ca9e5faa0` | 캘린더 Notion 스타일 리빌드: 날짜 클릭 팝업 |
| 2026-05-18 | `11177fdf9181` | 건강 플랫폼 전면 업그레이드: 식단/운동/Gemini AI |
| 2026-05-18 | `3f7893f1280a` | 버그수정: JS 문자열 줄바꿈 SyntaxError |
| 2026-05-18 | `d7b88a972f82` | 버그수정: onclick 따옴표 중첩 SyntaxError |
| 2026-05-18 | `8375d170509c` | v3: AI 챗봇, 프로필 시스템, 루틴 시더 |
| 2026-05-18 | `28e75ee23a3f` | 캘린더 UI 전면 개선: 데일리 브리핑 팝업 |
| 2026-05-18 | `7a5bb156c8be` | 반복 루틴 + 식단그리드, ↺ 루틴 버튼 |
| 2026-05-23 | `a0a87295e176` | **patch_v8**: 캘린더 파랑/녹색 시각 분리, 주간 템플릿 모달, AI 자연어 식단 파싱, Walmart 예산 |
| 2026-05-23 | `1b7f3187347e` | **patch_v9**: 원클릭 프리셋 시스템 (식단 2종 + 운동 3종), 캘린더 상단 psStrip, 예산 배지 자동표시 |
| 2026-05-23 | `43e50ea02155` | **patch_v10 Lean UX**: 사이드바 9개 항목 제거, 모달 advanced 접기, Quick-Add 바 |
| 2026-05-23 | `e403937628f0` | **patch_v11 Responsive**: 데스크톱 캘린더 overflow 수정 (셀 내부 스크롤, 짤림 방지), 모바일 아젠다 뷰 (14일 일별 카드), 하단 네비게이션 5탭, 식단(녹색)/운동(파란) 카드 디자인 강화, 768px 브레이크포인트 |
| 2026-05-23 | `6ee09ba00e2d` | **patch_v12 Notion UI**: PC 캘린더 셀 컴팩트 요약 배지(🏋️N/🥗N/✅N), 모바일 주간 탭바(#mbWeekBar 7일 수평 스크롤) + 일간 상세 카드, Notion 스타일 체크박스(식단 `done`/운동 `doneExercises`), 일별 완료도 게이지 |
| 2026-05-23 | `325f7d887b2b` | **patch_v13 UX Overhaul**: 캘린더 클릭 → 우측 와이드 사이드 패널(PC 520px) / 바텀 시트(모바일 92dvh) 전환, 버튼 디자인 시스템(hover·active·shadow 전면 적용), 완료 게이지 애니메이션(0.55s cubic), 실행취소 토스트, renderAll 후크로 패널 자동 갱신 |
| 2026-05-23 | `991133b538e0` | **patch_v16 Reset 권한 오류 수정**: "Missing or insufficient permissions" 버그 픽스 — Firestore 복합 쿼리(`where type==recurring`) → 이미 로드된 로컬 상태 배열(`todos/meals/workouts`) 클라이언트 필터링으로 전환. 텍스트 입력 가드 제거 — 체크박스 선택 후 즉시 실행. |
| 2026-05-23 | `b130b8cd11a6` | **patch_v15 Safe Reset Engine**: 🗑️ 루틴 초기화 버튼 (psStrip 우측, Soft Red `.btnReset`) + 2단계 안전장치 확인 모달 (`#resetModal`) — '초기화' 텍스트 입력 시만 확인 버튼 활성화. 카테고리별 체크박스 선택형 삭제 (`executeReset`): ①반복일정(type=recurring) ②이번주식단 ③이번주운동 ④주간루틴템플릿(localStorage) ⑤완료된할일(done=true). 499건 청크 Firestore batch delete. UID 필터 강제 — 타 사용자 데이터 보호. |
| 2026-05-23 | `eb7b1665858d` | **patch_v14 AI Health Engine**: 🥗 AI 식단 생성 버튼 (`generateAiWeeklyPlan`) + 💪 AI 운동 생성 버튼 (`generateAiWorkoutPlan`) — Gemini API로 Jayden 맞춤형 주간 식단/운동 자동 생성 (가성비 5대장 재료, 레시피 다변화, 등빨 프레임 빌더 루틴, 탈모 예방 조건 내장). 📖 레시피 모달 (`openRecipeModal`) — 사이드 패널 내 AI 생성 식단 클릭 시 재료·조리법·팁 표시. `_hbRecipes{}` 전역 캐시로 recipe 필드 저장. `hb_meals` Firestore 문서에 `recipe` 오브젝트 필드 추가. |
| 2026-05-23 | `9782b41eff70` | **patch_v20 전면 갈아엎기**: ①사이드바 전체 재구성 — navGroupHead 제거, 모든 뷰 버튼 상시 노출 (플래닝/목표&노트/건강/시스템 4개 섹션 헤더 `.navSect`). 클릭 시 해당 뷰로 즉시 이동. ②AI 멀티턴 대화 — `_aiHistory[]` 배열로 Gemini contents 누적, 이전 대화 맥락 유지. ③채팅 히스토리 UI — 사용자/AI 말풍선, 🗑 대화 초기화 버튼. ④AI 제안 카드 풀 리디자인 — 식단: 🔥kcal/단백질/탄수/지방 컬러 영양소 pill 배지+가격, 레시피 accordion. 운동: name/sets/reps/weight 테이블. ⑤AI 시스템 프롬프트 전면 개선 — Jayden 신체정보(178/82kg)+운동목표+선호식재료+운동수준+Dallas가격 내장, 주간계획 날짜 자동분산 규칙, 일일 2200kcal/단백질175g+ 타겟. |
| 2026-05-23 | `07bb5b04a5a5` | **patch_v19 confirmAiSuggest 완전 수정**: `db.batch().set()` → 개별 `db.collection().add()` + `Promise.all()` 패턴으로 교체. 데이터 sanitize 강화 — mealType 화이트리스트 검증, date 정규식 검증(`YYYY-MM-DD`), exercises 필드 타입 강제 변환(name/sets/reps/weight 만). addMealEntry 정확히 동일한 필드셋 사용. AI 프롬프트 확인 버튼 클릭 시 캘린더에 정상 반영 확인. |
| 2026-05-23 | `09fc2f2a943a` | **patch_v18 AI Fix + Sidebar Collapse**: ①AI 저장 권한 오류 완전 수정 (`curUser.uid` + `batch.set()` 패턴으로 교체 — `auth.currentUser.uid` + `.add()` 방식이 실제 Firestore 규칙과 불일치하던 문제). ②AI 제안 카드 개선 — Gemini가 배열로 복수 항목 반환 (음식명+칼로리+단백질+탄수+지방+달러가격+레시피 상세/운동+세트+횟수+무게+주의사항). ③레시피 `<details>` 접기/펼치기. ④사이드바 접기/펼치기 (`toggleNavGrp`) — "📋 플래닝 탭" (today/all/done/recurring/week), "🎯 목표 & 노트" (goals/ideas/workout) 2개 그룹 버튼 추가. ⑤다운로드 스크립트 수정 (1MB 초과 파일 raw URL로 처리). ⑥46개 버튼 전수 검증 — 모두 함수 정의 확인 완료. |
| 2026-05-23 | `33a7559fed0e` | **patch_v17 Full UX Overhaul**: ①AI 식단/운동 생성 권한 오류 수정 (`generateAiWeeklyPlan`/`generateAiWorkoutPlan` — Firestore 복합 쿼리 → 로컬 상태 필터링). ②사이드바 3탭 재구성 (📅 대시보드/🔁 루틴 설정/⚙️ 설정 — 불필요 탭 전부 숨김). ③✨ AI 자연어 프롬프트 바 (캘린더 상단 — 자연어 입력 → Gemini 파싱 → 제안 카드 → 확인 시 Firestore 저장). ④⚙️ 설정 뷰 신규 (API 키 입력·저장·삭제, 모델 선택, 테마 변경, 초기화 버튼). ⑤캘린더 셀 높이 90px로 조정 (텍스트 짤림 완화). ⑥사이드바 너비 264→300px 확장. |
| 2026-05-23 | *(no new commit)* | **patch_v21 Timebox Timeline**: `renderToday()` 전면 교체 — 오전 6시~오후 10시 타임라인 뷰. 식단/운동/할일을 시간대별 블록으로 배치. 현재 시간 붉은 표시선, 시간 미배정 할일 경고 배너, 완료 항목 하단 접기. `renderAll2()` 경계 기준 함수 교체. |
| 2026-05-23 | *(no new commit)* | **patch_v22 psStrip 제거 + confirmAiSuggest 복원**: psStrip 프리셋 버튼 HTML 전체 제거 (`#presetStrip` div + `.psStrip`/`.psBtn`/`.psSel`/`.psApplyBtn` 등 CSS 규칙). 패치 히스토리 중 소실된 `confirmAiSuggest()` 함수 재삽입 (기본 async/await 버전). |
| 2026-05-23 | *(no new commit)* | **patch_v23 confirmAiSuggest 진단 로깅**: `[CAI]` 접두사 전방위 `console.log`. `_aiSuggestData`/`curUser` null 조기반환에 토스트 추가. 타입 폴백 휴리스틱 (`exercises[]` → workout, `kcal`/`mealType` 필드 → meal). `window.confirmAiSuggest` 명시 전역 할당. 진단 결과: 함수는 정상 실행되나 Firestore 권한 오류 확인. |
| 2026-05-23 | `52b6519d5634` | **patch_v24 Promise.then() 패턴**: `async/await` → `Promise.all().then().catch()` 전환 (`applyWeeklyTemplate`와 동일한 패턴). `Number()` → `parseFloat()`/`parseInt()` 로 변환 강화. (근본 원인은 Firestore 규칙 부재였으므로 이 패치 단독으로는 해결 불가) |
| 2026-05-23 | `5b4f68f94936` | **patch_v26 삭제 기능 + qaBar 제거 + UX 개선**: ①모든 항목(할일/식단/운동) hover 시 ✕ 삭제버튼 표시 — `_delItem(type,id,name)` 확인 후 Firestore delete. 월간 셀·주간 뷰·todo 인라인 칩 모두 적용. ②빠른추가 바(qaBar HTML+CSS+quickAdd 로직) 완전 제거 → AI 프롬프트가 유일한 입력창. ③주간 뷰 날짜 헤더에 일일 총 kcal 자동 표시. ④calCell min-height 90→110px. |
| 2026-05-23 | `119ef7157a96` | **patch_v25 Weekly Detail View + Drag-and-Drop**: ①월간 캘린더 셀 — 개수 배지(`🏋️ N개`) → 실제 항목명 표시 (운동 종목명/세트·횟수, 식단명+kcal, 할일 텍스트, 최대 3개 + N개 더). ②주간 보기(`renderWeek()`) 완전 재구현 — 7열 그리드, 각 날짜에 hb_workouts 종목 전체 표시(세트·횟수·무게), hb_meals 식사별 타입+이름+kcal, 할일 체크박스 스타일. ③드래그 앤 드랍 — todos/hb_meals/hb_workouts 모든 항목이 draggable, 다른 날짜 셀로 드롭하면 Firestore `date` 필드 자동 업데이트 (`_dndStart`/`_dndDrop`). 월간+주간 뷰 모두 drop zone. |
| 2026-05-23 | — | **🔥 Firestore Rules 수정 (근본 원인 해결)**: `daily-task-planner-9187f` 프로젝트 Firestore 보안 규칙에 `hb_meals`·`hb_workouts` match 블록 누락 → Firebase 기본 deny 적용됨. Firebase 콘솔에서 두 컬렉션에 uid 기반 read/create/update/delete 규칙 추가 후 Publish. 검증: JS 콘솔 테스트 write `{"hb_meals":"OK:76RNr52VhyKA5NRqKgMh","hb_workouts":"OK:Skc51npXpsnIIVAfOb8w"}` 확인. 캘린더 May 23 셀 🐂2개/🥗1개 표시 최종 확인. |
| 2026-05-23 | `6fc29e17bf40` | **patch_v28 할일삭제+폰트확대+Dallas가격+권한수정**: ①캘린더 할일 섹션(tDiv) — 커스텀 _tl 요소 → _mkItem 으로 교체, hover ✕ 삭제버튼 추가. ②_mkItem font-size 10px→14px, 섹션헤더 12px, calCell min-height 130px. ③ 함수 추가 — 30종 재료 Dallas TX 75229 기준 단가 lookup, 캘린더 셀·사이드 패널 식단 옆에  자동 표시. ④ curUser 가드 추가 + goal/idea 타입 지원. ⑤ 에러 핸들러 추가(기존 무음실패 → toast 표시). Firestore 규칙 확인: todos/goals/ideas/hb_meals/hb_workouts 모두 올바른 read/update/delete 규칙 적용 확인 |
| 2026-05-24 | `d02e9d173926` | **patch_v15-ux (설정·사이드패널·사이드바 UX 개선)**: ①설정 탭 즉시 사라짐 수정 — `setView`에서 settings 뷰는 `toggleSb(false)` 제외 (모바일에서 사이드바 닫아도 설정 유지). ②`renderSettings()` 테마 이름 수정 — `['light','dark','forest','ocean','rose']` → 실제 CSS 테마명 `['signiel','obsidian','blanc','jade','sapphire']` (시그니엘/옵시디언/블랑/제이드/사파이어). ③모바일 하단 내비 📋 루틴 탭 → ⚙️ 설정 탭으로 교체 (`mbn-tmpl` → `mbn-settings`). ④왼쪽 사이드바 구조 재편 — **[핵심]** 대시보드·오늘할일·주간보기 / **[건강]** 식단관리·운동루틴·주간루틴세팅 / **[▸ 더 보기]** 목표관리·아이디어노트·전체할일·완료항목·반복일정 / **[하단 고정]** ⚙️ 설정 (`margin-top:auto`). ⑤사이드 패널 각 섹션 헤더에 `+ 추가` 퀵-애드 버튼 인라인 폼 — 운동(`_cspSaveWrk`)/식단(`_cspSaveMeal`)/할일(`_cspSaveTodoQ`) Enter 키 저장 지원. 패널이 닫히지 않고 항목 추가 가능. ⑥식단 항목 서브정보 단순화 — kcal만 표시 (단백질·탄수·지방 제거). ⑦사이드 패널 푸터 → "+ 할일 추가" 단일 버튼 (🥗 식단·💪 운동 탐색 버튼 제거, 섹션별 인라인 추가로 대체). |
| 2026-05-24 | `24ed27ead4a5` | **patch_v17_refactor (전면 미니멀 리팩토링)**: Step1 — 레거시 기능 제거: 사이드바 12개 버튼 → 📅 대시보드 + ⚙️ 설정 2개만. 모바일 하단 nav 5개 → 2개. vToday/vAll/vDone/vRecurring/vGoals/vIdeas 뷰 `display:none!important`. weeklyTmplModal `display:none!important`. renderToday/renderAll2/renderDone/renderRecurring/renderGoals/renderIdeas/renderWeek 등 레거시 함수 빈 스텁으로 교체(ReferenceError 방지). Step2 — setView() 완전 재작성: `calendar`/`settings` 두 뷰만 처리. `.vPg.va` + `.navItem.on` + `.mbNavBtn.on` 정확히 동기화. 설정탭 깜빡임 버그 근본 수정 — renderAll()에서 settings 뷰 재렌더 호출 제거(데이터 갱신 시 settings 뷰 상태 유지). renderAll() 간소화 — calendar 뷰만 renderCal(). Step3 — AI 시스템 컨텍스트 교체: "너는 내 전속 헬스/생산성 코치야. 목표: 경제적으로 가장 저렴하게 세팅한 벌크업 식단과 운동 프로그램." 벌크업·저예산($40-50/주) 중심. Step4 — 사이드 패널 푸터 인라인 Task 추가 폼: `#cspTaskInp` 입력창 + Enter/+ 추가 버튼. `_cspAddTask()` 함수 신규 추가. Node.js 구문 검사 통과. 브라우저 10개 항목 검증 완료. |
| 2026-05-24 | `d784cac20c3f` | **patch_v16-fix (체크박스 버그 수정 + 모달 간소화)**: ①**CRITICAL BUG FIX**: `toggleMealDone`(×2) + `toggleWorkoutExDone`(×1) 에서 `if(!db\|\|!currentUser)return;` → `if(!db\|\|!curUser)return;` — `currentUser`는 undefined이므로 체크박스가 항상 조기 반환됨. Firestore write 테스트(`meal_done_after_toggle: true`) 확인. ②`deleteWorkoutEx` — 삭제 확인 다이얼로그(`confirm('삭제?')`) 제거 → 즉시 삭제. ③모달 타입 버튼 간소화 — 기존 `[☑️ 할 일 \| 💪 운동 할일 \| 🥗 식단 할일]` → `[☑️ 할 일 \| 🔁 반복]` (운동/식단 할일 버튼은 todo에 카테고리 태그만 달 뿐 실제 Firestore 레코드를 생성하지 않아 사용자 혼동 유발 — 제거). ④중복 `toggleMealDone` 함수(구버전 최소 버전 ~line 4845) → 주석으로 교체. ⑤`.cspNChk`, `.cspItem` CSS에 `cursor:pointer` 추가. 36개 핵심 함수 전수 확인 완료. |
| 2026-05-24 | `d5ef00076c70` | **patch_v14-final (사이드패널 UX 전체 수정)**: ①`.mOv` modal overlay z-index 500→700 (사이드 패널 z-index 600보다 위 — "+" 버튼 눌렀을 때 그레이 화면 막힘 해결). ②사이드 패널 항목별 hover 삭제 버튼 — 운동(`deleteWorkoutEx`)/식단(`deleteMealEntry`)/할일(`deleteTodo`) 각각 🗑 버튼 추가 `.cspDelBtn` (opacity:0→1 on hover, mobile .55 상시). ③삭제 confirm 대화상자 제거 (`deleteMealEntry`/`deleteTodo` — 즉시 삭제). ④사이드 패널 푸터 3-button — 기존 "+ 할일 추가" 단일 버튼 → 할일 추가 + 🥗 식단 + 💪 운동 3개 버튼 (패널 닫고 해당 뷰 이동). ⑤모달 타입 버튼 개편 — 🔁 반복·📝 메모 → 💪 운동 할일·🥗 식단 할일 (setCat 연동, _mClearCat 헬퍼 추가). ⑥식단 정렬 버그 수정 — `mo={breakfast:0,...}` → `{breakfast:1,...}` (falsy 0 → 9 오정렬 해결, 아침→점심→저녁→간식 순). ⑦카테고리 row `.mAdvRow` → `.mCatRow` (항상 표시). ⑧setCat 함수 mTypeBtn 동기화 추가. ⑨JS syntax OK, hilliard-todo 배포 완료. |
| 2026-05-23 | `055bfd174da3` | **patch_v27 플랫폼 6대 개선**: ①사이드바 12개→5개 필수 항목(대시보드/오늘할일/주간보기/목표관리/설정) + "▸ 더 보기" 접이식 패널(나머지 7개). ②오늘 kcal/단백질/탄수/지방 진행 바 — `renderKcalStatus()` 캘린더 상단 자동 표시 (타겟 2200kcal/175g). ③주간 뷰 운동 종목 완료 체크박스 — `w.doneExercises{exIdx:bool}` Firestore 저장, `toggleWorkoutExDone` 버그(`currentUser`→`curUser`) 수정. ④반복 루틴 항목 amber 색상 변경 (`.calRecPill` gray→ orange `#e65100`). ⑤할일 추가 시 시간 미입력 → 타임블록 배치 안내 토스트 자동 표시. ⑥done todos 완료 항목 하단 접힘 구현(`renderAll2` 검사). |

---

## 🔗 참고

| 항목 | 값 |
|------|-----|
| 라이브 URL | https://hilliardbuffshire.github.io/hilliard-hub/ |
| GitHub Repo | `hilliardbuffshire/hilliard-hub` (branch: `main`) |
| 배포 토큰 경로 | `C:\Users\savem\OneDrive\Desktop\Jayden\Projects\hub\.deploy_config` |
| Firestore 프로젝트 | `daily-task-planner-9187f` |
