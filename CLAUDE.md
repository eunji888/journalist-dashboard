# 기자 취재 동향 대시보드 — Claude Code 작업 지시서

## 🎯 목표

텔레그램 봇이 Supabase에 쌓고 있는 뉴스 기사 데이터를 읽어서,
기자 30명의 취재 동향을 실시간으로 보여주는 대시보드를 완성한다.
Vercel에 배포 가능한 정적 사이트(HTML/CSS/JS) 형태로 만든다.

---

## 🔐 환경변수 (`.env` 또는 Vercel 환경변수에 설정됨)

```
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_ANON_KEY=eyJ...
```

> 작업 시작 전 반드시 이 두 값이 설정되어 있는지 확인할 것.
> 없으면 사용자에게 요청하고 진행하지 말 것.

---

## 🔍 Step 1 — DB 구조 자동 탐색 (반드시 먼저 실행)

코드 작성 전에 아래 순서로 Supabase 테이블 구조를 직접 파악한다.

```bash
# 1. 테이블 목록 + 컬럼 구조 한 번에 확인
curl "$SUPABASE_URL/rest/v1/?apikey=$SUPABASE_ANON_KEY" \
  -H "Authorization: Bearer $SUPABASE_ANON_KEY"

# 2. 기사 관련 테이블 샘플 데이터 확인 (테이블명은 탐색 후 교체)
curl "$SUPABASE_URL/rest/v1/[테이블명]?limit=5&select=*" \
  -H "apikey: $SUPABASE_ANON_KEY" \
  -H "Authorization: Bearer $SUPABASE_ANON_KEY"

# 3. 기자 식별 컬럼 값 종류 확인
curl "$SUPABASE_URL/rest/v1/[테이블명]?select=[기자컬럼]&limit=50" \
  -H "apikey: $SUPABASE_ANON_KEY" \
  -H "Authorization: Bearer $SUPABASE_ANON_KEY"
```

탐색 후 다음을 파악하고 이 파일 하단 [탐색 결과] 섹션에 채워넣을 것:
- 기사 테이블명
- 기자 식별 컬럼명 (journalist / author / reporter / user_id / telegram_id 등)
- 제목 컬럼명
- 발행시각 컬럼명
- 카테고리 컬럼명 (없으면 제목 키워드로 자동 분류)
- 기자 테이블이 별도 존재하는지 (JOIN 필요 여부)

---

## 🏗️ Step 2 — 구현 기능

탐색한 실제 컬럼명으로 쿼리를 작성한다. 컬럼명을 임의로 가정하지 말 것.

### 필수 기능
1. **기자별 기사량 랭킹** — 기간 필터(7일/30일/90일)
2. **주제/카테고리 분포** — 카테고리 컬럼 없으면 제목 키워드 기반 추정
3. **시간대별 발행 패턴** — 0~23시 히스토그램
4. **실시간 피드** — Supabase Realtime 구독 (INSERT 이벤트)
5. **기자 상세** — 클릭 시 해당 기자의 패턴 드릴다운
6. **주간 증감률** — 이번주 vs 지난주 기사량 비교

### Realtime 구독 패턴
```javascript
import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2'
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

supabase
  .channel('new-articles')
  .on('postgres_changes',
    { event: 'INSERT', schema: 'public', table: '실제테이블명' },
    (payload) => updateFeed(payload.new)
  )
  .subscribe()
```

---

## 📁 Step 3 — 파일 구조 (Vercel 정적 배포용)

```
/
├── index.html       # 단일 파일 SPA — 모든 JS/CSS 인라인
├── vercel.json      # 배포 설정
├── .env.example     # 환경변수 템플릿 (실제 값 없이)
└── CLAUDE.md
```

**vercel.json 기본값:**
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [{ "key": "X-Frame-Options", "value": "SAMEORIGIN" }]
    }
  ]
}
```

**환경변수 처리 방법:**
정적 HTML에서는 `process.env`를 쓸 수 없으므로,
Vercel 환경변수를 `window._env` 객체로 주입하는 방식 사용:

```html
<!-- index.html 상단 -->
<script>
  window._env = {
    SUPABASE_URL: '%%SUPABASE_URL%%',       // Vercel이 빌드 시 치환
    SUPABASE_ANON_KEY: '%%SUPABASE_ANON_KEY%%'
  }
</script>
```

또는 사용자에게 직접 값을 입력받는 설정 화면을 초기에 표시하는 방식도 허용.

---

## 🎨 디자인 시스템 (기존 유지)

```css
--bg: #0d0f14;  --surface: #151820;  --surface2: #1c2030;
--accent: #4f8ef7;  --green: #2ecc8f;  --amber: #f5a623;  --red: #e85d5d;
--font: 'Noto Sans KR', sans-serif;  --mono: 'JetBrains Mono', monospace;
```

외부 라이브러리 (CDN만 사용):
- Chart.js: `cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js`
- Supabase: `cdn.jsdelivr.net/npm/@supabase/supabase-js@2`

---

## ⚠️ 코드 규칙

- `var` 금지, `const`/`let` 사용
- 비동기는 `async/await`
- Chart.js 재생성 전 `.destroy()` 필수
- 상태는 `state` 객체 하나로 중앙 관리
- 컬럼명 하드코딩 금지 — 탐색 결과 기반으로만 작성
- Supabase anon key 노출은 괜찮으나 RLS 활성화 여부 확인 후 명시할 것

---

## ✅ 완료 기준

- [ ] Supabase 실제 스키마 탐색 완료 및 하단에 문서화
- [ ] 실제 컬럼명 기반 쿼리 사용
- [ ] Realtime 구독 작동
- [ ] 기간 필터 (7/30/90일) 작동
- [ ] 기자 클릭 → 상세 패널
- [ ] vercel.json 포함
- [ ] README.md에 Vercel 배포 3단계 기재
- [ ] 환경변수 하드코딩 없음

---

## 📋 탐색 결과 (Claude Code가 Step 1 완료 후 여기에 채울 것)

```
기사 테이블명:       articles
기자 컬럼명:         journalist_id (FK → journalists.id), reporter_name (비정규화)
제목 컬럼명:         title
발행시각 컬럼명:     published_at
카테고리 컬럼명:     section (값: IT, 경제, 마켓시그널, 세계, 종합, 증권, 크립토허브)
기자 별도 테이블:    있음 (journalists 테이블 — id, name, outlet, email, source_url)
RLS 활성화:         비활성화 (anon key로 전체 읽기 가능, 총 기자 39명 / 기사 6338건)
```
