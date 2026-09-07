# 흠 — 배포 안내

빌드 과정이 없습니다. 파일을 그대로 올리면 됩니다.

## 올려야 할 파일

```
index.html          ← 본체
og.png              ← 기본 미리보기 이미지
og-dawn.png ~ og-soldout.png   ← 유형별 미리보기 8장
t/                  ← 유형별 공유 페이지 (폴더째로)
  dawn.html, just.html, ...
```

`heum-site.zip`을 풀면 이 구조 그대로 나옵니다.

---

## 지금 상태

**구현됨**
- ① 랜딩 → ② 12문항 → ③ 결과 → ④ 궁합 → ⑤ 고민 등록 → 완료
- 채점 로직, 8유형 분기
- 캐릭터 흠 (유형별 눈동자)
- 로그 이벤트 전체
- 공유 — 카톡 공유창 → 클립보드 → 링크 복사창 순으로 시도
- 유형별 카톡 미리보기 (OG)

**아직 없음**
- 웹 푸시 (3일/2주 알림)
- 서버 저장 — 지금은 브라우저에만 쌓입니다

---

## 배포하기

### 1. 깃허브 계정 만들기
github.com 에서 가입.

### 2. 저장소 만들기
- `New repository` 클릭
- 이름 `heum`
- **Public** 선택
- `Create repository`

### 3. 파일 올리기
- `uploading an existing file` 링크 클릭
- `heum-site.zip`을 **압축 푼 뒤, 폴더 안 내용물 전체**를 끌어다 놓기
  (zip 파일째로 올리면 안 됩니다)
- `t` 폴더도 같이 끌어다 놓으면 구조가 유지됩니다
- `Commit changes`

### 4. Vercel 연결
- vercel.com 접속 → **Continue with GitHub**
- `Add New... > Project`
- `heum` 저장소 옆 `Import`
- 설정 건드리지 말고 `Deploy`

1~2분 뒤 `heum-xxxx.vercel.app` 주소가 나옵니다. 끝입니다.

이후 깃허브에서 파일을 고치면 Vercel이 알아서 다시 배포합니다.

> GitHub Pages로도 돌아갑니다. Settings > Pages에서 브랜치를 지정하면 `아이디.github.io/heum` 으로 열립니다. 다만 나중에 푸시 알림을 붙이려면 Vercel이 필요합니다.

---

## 배포 직후 확인

**폰에서** 주소를 열고 테스트를 끝까지 해보세요.

1. **친구한테 보내기** → 카톡 공유창이 뜨는지
2. 나온 링크를 실제 카톡방에 보내기 → 유형 이름과 캐릭터가 있는 미리보기 카드가 뜨는지

미리보기가 안 뜨면 카톡이 이전 정보를 캐시한 겁니다. `developers.kakao.com/tool/debugger/sharing` 에 주소를 넣고 **초기화**를 누르면 다시 읽어갑니다.

그래도 이미지가 안 보이면 `t/` 폴더 안 html 파일들의 `og:image`를 전체 주소로 바꿔주세요.

```html
<meta property="og:image" content="https://heum-xxxx.vercel.app/og-dawn.png">
```

---

## 데이터 저장 켜기

지금은 응답이 **브라우저에만** 저장됩니다. 내 폰에서 테스트한 건 볼 수 있지만, 다른 사람이 한 건 못 봅니다. 실제로 데이터를 모으려면 Supabase를 붙여야 합니다.

### 1. supabase.com 가입 → 새 프로젝트 생성

### 2. SQL Editor에서 아래 실행

```sql
create table responses (
  id bigserial primary key,
  result_id text unique,
  answers jsonb,
  score_decision int,
  score_motive int,
  score_regret int,
  type_code text,
  created_at timestamptz default now()
);

create table concerns (
  id bigserial primary key,
  result_id text,
  item text,
  amount int,
  created_at timestamptz default now()
);

create table events (
  id bigserial primary key,
  session_id text,
  result_id text,
  name text,
  payload jsonb,
  created_at timestamptz default now()
);

alter table responses enable row level security;
alter table concerns  enable row level security;
alter table events    enable row level security;

create policy "insert only" on responses for insert with check (true);
create policy "insert only" on concerns  for insert with check (true);
create policy "insert only" on events    for insert with check (true);
```

RLS를 켜고 insert만 허용했기 때문에, 키가 노출돼도 남의 데이터를 읽어갈 수 없습니다.

### 3. index.html 상단 CONFIG 수정

```js
const CONFIG = {
  supabaseUrl: "https://xxxxx.supabase.co",
  supabaseKey: "eyJhbGci..."   // anon public key
};
```

Settings > API 에서 `Project URL`과 `anon public` 키를 복사하면 됩니다.

---

## 배포 후 첫 주에 볼 것

Supabase Table Editor에서 바로 확인할 수 있습니다.

**1. 문항별 이탈**
```sql
select payload->>'q' as q, count(*)
from events where name = 'question_view'
group by 1 order by 1;
```
숫자가 뚝 떨어지는 문항이 지루한 문항입니다.

**2. 문항별 고민 시간**
```sql
select payload->>'q' as q, avg((payload->>'ms')::int) as ms
from events where name = 'question_answer'
group by 1 order by 1;
```
오래 걸리는 문항은 애매하다는 뜻이라 문구를 고쳐야 합니다.

**3. 유형 분포**
```sql
select type_code, count(*) from responses group by 1 order by 2 desc;
```
한 유형에 몰리면 채점 기준을 조정해야 합니다. 이 숫자가 결과 화면의 %가 됩니다.

**4. 퍼널**
```sql
select name, count(distinct session_id)
from events
where name in ('landing_view','test_start','test_complete','result_view','match_view','ask_view','ask_submit')
group by 1;
```

**5. 공유가 어떤 방식으로 됐나**
```sql
select payload->>'ch' as method, count(*)
from events where name = 'share_done' group by 1;
```
`native`가 많으면 정상, `sheet`가 많으면 공유 버튼을 손봐야 합니다.

**6. 고민 금액대**
```sql
select case
  when amount < 30000 then '3만 미만'
  when amount < 100000 then '3~10만'
  when amount < 300000 then '10~30만'
  else '30만 이상' end as band,
  count(*)
from concerns group by 1;
```

---

## 다음에 붙일 것

1. **웹 푸시 + 3일/2주 알림** — 종단 데이터의 핵심. Vercel 서버 기능이 필요합니다
2. **결과 이미지 저장** — 인스타 스토리용
3. **비율(%) 실시간 집계** — 응답 100건 넘으면 결과 화면에 실제 비율 표시
