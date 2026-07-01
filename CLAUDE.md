# Colinch Flutter 블로그 — 전용 하네스

> 공통 규칙(파일명·frontmatter·SEO·인간 글쓰기·워크플로)은 상위 폴더 `../CLAUDE.md` 참조

---

## 블로그 정보

| 항목 | 값 |
|------|----|
| 사이트 URL | https://colinchflutter.github.io |
| 저자 | Colin Flutter |
| 언어 | **English** (모든 포스트 영어 작성) |
| 주제 | Flutter deep-dives — packages, widgets, advanced patterns |
| 특징 | 특정 패키지·위젯·기능을 시리즈로 쪼개서 상세 커버 |
| Jekyll 설정 | `_config.yml` 참조 |

---

## 이 블로그의 SEO 키워드 전략

특정 Flutter 패키지·위젯을 영어로 검색하는 개발자를 타깃으로 한다.

| 그룹 | 주요 키워드 예시 | 의도 |
|------|----------------|------|
| How-to | `how to use X in Flutter`, `Flutter X example` | 방법 탐색 |
| Problem | `Flutter X not working`, `X error in Flutter` | 문제 해결 |
| Deep-dive | `Flutter X advanced`, `customizing X in Flutter` | 심화 학습 |
| Comparison | `Flutter X vs Y`, `best way to X in Flutter` | 의사결정 |

**description 작성 기준 (영어):**
- 50~160자, 영어
- 핵심 패키지명·동작 포함
- 예: `"Learn how to implement custom scroll physics in Flutter's PageView with working code examples and edge case handling."`

---

## 포스트 작성 언어·톤

- **전체 영어** 작성
- 톤: practical, developer-to-developer — "Here's what I found", "The trick is..."
- 반말체 해당 없음 — 영어 특성상 직접 2인칭("you", "your")
- AI 감지 트리거 (영어): 아래 표현 사용 금지
  ```
  "In conclusion", "To summarize", "It's worth noting that"
  "This allows you to", "In order to", "It is important to"
  "Leverage", "Utilize", "Facilitate", "Robust solution"
  "Let's dive into", "In this article, we will explore"
  ```
- 자연스러운 영어 패턴:
  ```
  "Turns out, ...", "The problem is ...", "I spent an hour on this:"
  "Short answer: ...", "Here's the catch:", "Works fine on Android, breaks on iOS."
  ```

---

## 태그 목록

첫 번째 태그는 해당 패키지·위젯명. 대소문자 혼용 주의 — 기존 포스트와 일치시킨다.

```
# 패키지 (패키지명 그대로)
ScrollPhysics, CustomPainter, PageView, ListView
sqflite, firebase, GetX, FlutterHooks, WorkManager
audio, FlutterWeb, webassembly, reactnative

# 기능·패턴
animation, state_management, navigation, localization
networking, caching, testing, performance

# 플랫폼
Android, iOS, Web, Desktop
```

---

## 시리즈 현황

이 블로그는 패키지·위젯 하나를 집중적으로 파고드는 시리즈 구조.  
새 시리즈 시작 시 아래 표에 추가한다.

| 시리즈 | 주제 | 기간 | 상태 | 편수 |
|--------|------|------|------|------|
| ScrollPhysics | Flutter scroll physics customization | 2023.09 | 완결 | ~30 |
| CustomPainter | Custom drawing in Flutter | 2023.09 | 완결 | ~40 |
| sqflite | Local DB with sqflite | 2023.09 | 완결 | ~30 |
| audio | Audio playback & recording | 2023.09 | 완결 | ~40 |
| SVG | SVG rendering in Flutter | 2023.10 | 완결 | ~50 |
| video trimming | Video trimming features | 2023.10 | 완결 | ~50 |
| flutter_animate | Animation package deep-dive | 2026.06 | **진행 중** | 25+ |

마지막 포스트: `_posts/2026/06/29/2026-06-29-14-00-00-631062-flutter-animate-go-router-page-transitions.md`  
다음 일련번호: `643407`

### 다음 시리즈 후보 (새 시리즈 시작 시 선택)

```
flutter_animate     — Animation package deep-dive
go_router           — Navigation 2.0 routing
riverpod            — State management deep-dive
flutter_hooks       — Hooks pattern in Flutter (심화)
drift                — Type-safe SQLite ORM
```

---

## 시리즈 포스트 구조 (이 블로그 특화)

이 블로그는 하나의 주제를 10~50편으로 쪼개서 다룬다.  
시리즈 내 각 편은 **하나의 구체적 기능이나 케이스**만 다룬다.

시리즈 목차 예시 (ScrollPhysics):
```
01. What are Scroll Physics in Flutter?
02. Applying custom scroll physics to PageView
03. Creating spring-effect scroll physics
04. Implementing iOS-style scroll physics on Android
...
```

새 시리즈 시작 시:
1. 시리즈 목차 10~20편 먼저 설계
2. 첫 편은 "Introduction to X in Flutter" 또는 "What is X?"
3. 마지막 편은 "Advanced use cases" 또는 종합 예제

---

## 기존 포스트 검색

```bash
grep -rl "KEYWORD" _posts/ --include="*.md"
find _posts -name "*.md" | sort | tail -10
```

---

## 글 유형별 참조 파일

| 유형 | 참조 파일 |
|------|-----------|
| 시리즈 첫 편 | `_posts/2023/09/20/2023-09-20-08-46-49-907184-applying-custom-scroll-physics-to-pageview-in-flutter.md` |
| 심화 편 | `_posts/2023/10/31/2023-10-31-15-38-13-046463-using-svg-to-create-interactive-floor-plans-and-architectural-visualizations-in-flutter.md` |
