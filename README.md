# 기자 취재 동향 대시보드

기자 취재 동향을 실시간으로 시각화하는 정적 SPA입니다.

## Vercel 배포 3단계

1. **GitHub에 푸시**
   ```bash
   git init && git add . && git commit -m "init"
   gh repo create journalist-dashboard --public --source=. --push
   ```

2. **Vercel에 연결**
   Vercel 대시보드에서 해당 GitHub 저장소를 Import합니다.
   Framework Preset: **Other** (빌드 명령 없음, 출력 디렉토리: `.`)

3. **환경변수 설정 (선택)**
   별도 빌드 없이 브라우저 로컬스토리지를 사용하므로, 첫 접속 시 UI에서 직접 입력합니다.

## 로컬 실행

```bash
open index.html
# 또는
npx serve .
```

## 데이터 구조

| 테이블 | 역할 |
|--------|------|
| `articles` | 기사 데이터 (journalist_id, title, published_at, section 등) |
| `journalists` | 기자 정보 (name, outlet, email 등) |

- RLS: 비활성화 (anon key로 전체 읽기 가능)
- Realtime: `articles` INSERT 이벤트 구독
