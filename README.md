# 가계부 API (클라우드컴퓨팅실습 4주차)
- GitHub: https://github.com/platina310/ledger-api
- Render: https://ledger-api-7ls0.onrender.com/docs

FastAPI + SQLAlchemy로 만든 계좌·거래·카테고리 API. 데이터는 Supabase(클라우드 PostgreSQL)에 저장하고, 앱은 Render에 배포했다.

## ① 결과 확인
   ![Supabase Table Editor](supabase.png)
   ![Render GET /accounts](render-docs.png)

로컬에서 만든 「월급통장」과 Render 주소에서 만든 「배포테스트」가 같은 응답에 함께 나온다. 로컬 앱과 Render 앱이 같은 Supabase DB를 보고 있다는 뜻이다.

## ② 핵심 개념

- 계좌·거래를 두 테이블로 나눈 이유(1:N): 계좌 하나에 거래가 여러 건 붙는다. 거래마다 계좌 정보를 복사하지 않고 `account_id`(외래키)로 계좌를 가리키게 하면 중복이 없고, 존재하지 않는 계좌의 거래는 DB가 처음부터 거부한다.
- 모델 클래스와 테이블의 대응: 클래스 하나가 테이블 하나, 속성 하나가 컬럼 하나다. `Mapped[str]`은 NOT NULL, `ForeignKey`는 외래키가 되고, `create_all()`이 이 정의로 `CREATE TABLE`을 만든다. `relationship`은 테이블에 컬럼을 만들지 않고, 파이썬에서 `account.transactions`로 관련 거래를 조회하게 해 준다.
- 접속 문자열을 `.env`로 분리하는 이유: 연결 문자열에는 DB 비밀번호가 들어 있어서, 코드에 쓰면 GitHub에 그대로 공개된다. `.env`를 `.gitignore`로 제외하고, 로컬은 `.env`, Render는 환경변수로 같은 값을 넣는다. 그래서 코드를 한 줄도 바꾸지 않고 환경마다 다른 접속 정보를 쓸 수 있다.

## ③ 자유 로그

- 중첩 응답이 안 나왔던 문제: 계좌 상세를 조회했는데 `transactions`가 없었다. 응답에 키 자체가 없다는 점에서 다른 응답 스키마(`AccountRead`)가 쓰였다는 걸 알았고, 실제로는 `/detail`이 없는 단건 조회 경로를 실행하고 있었다. Swagger의 Request URL로 어떤 경로를 호출했는지 확인하는 습관이 필요하다.
- 거래가 9건으로 중복된 문제: `POST /transactions`를 여러 번 눌러 같은 거래가 쌓였다. API에 삭제 경로가 없어서 Supabase SQL Editor에서 먼저 `SELECT`로 대상을 확인하고 `DELETE ... WHERE id > 2`로 지웠다. 지운 뒤 새로 만든 행의 id가 이어서 붙는 걸 보고, PostgreSQL이 한 번 쓴 번호를 재사용하지 않는다는 걸 알았다.
- 카테고리 집계가 비어 보였던 문제: `/detail` 응답에는 `category_id`가 나오지 않아서 거래와 카테고리가 연결됐는지 알 수 없었다. SQL Editor에서 거래와 카테고리를 `LEFT JOIN`으로 조회하고 `UPDATE`로 `category_id`를 연결했다. SQL Editor는 여러 문장을 실행하면 마지막 `SELECT` 결과만 보여 준다는 것도 이때 알았다. 결과가 계속 비어 보인 건 `/docs`를 새로고침해서 결과가 지워진 것이었고, Execute를 다시 누르자 정상으로 나왔다.
- 중복 계좌 정리: 계좌도 여러 번 만들어져 있어서 SQL로 지웠다. 거래가 붙은 계좌는 외래키 때문에 지울 수 없으므로, 남은 거래가 모두 1번 계좌 것인지 먼저 확인했다.
- 배포: Render에서는 `.env`가 없으므로 환경변수 `DATABASE_URL`을 넣었고, Start Command는 파일 구조에 맞게 `main:app`으로 적었다. Swagger의 Example Value는 예시일 뿐이고, Try it out → Execute를 해야 실제 응답이 나온다는 것도 확인했다. 3주차와 달리 서버가 잠들었다 깨어나도 데이터가 남는다. 앱과 데이터가 분리돼 있기 때문이다.
- 아직 남은 것: 삭제 API가 없어서 데이터를 SQL로 직접 지웠다. `DELETE` 경로를 추가하면 좋겠다. 또 거래를 만들어도 계좌 잔액이 바뀌지 않는데, 이 부분은 확장 단계 7의 이체(트랜잭션)에서 다룬다.
- AI 활용과 검증: 막힌 화면과 응답을 Claude에게 보여 주고 원인 진단을 요청했다. 제안받은 SQL은 바로 실행하지 않고 먼저 `SELECT`로 대상 행을 확인한 뒤 실행했다. 결과는 Swagger의 실제 응답과 Supabase Table Editor 두 곳에서 교차 확인했다.