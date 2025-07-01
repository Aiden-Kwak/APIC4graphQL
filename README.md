### 데모 목적
APIC와 APIC for GraphQL의 GraphQL API 관리 기능을 시연한다.

### Process
- Stepzen 접속 및 로그인 정보 획득 (https://us-east-a.dashboard.ibm.stepzen.com/?instance=20250625-0707-4923-4038-f78c0da81040&environment=ramzanipurwa)

- CLI Access Information Form: 
```bash
stepzen login [your_domain] -a [your_environment]
```

- REST API로부터 GraphQL API를 빌드
```bash
mkdir product-demo
cd product-demo

# Stepzen 작업공간초기화
stepzen init --endpoint=api/product-demo 

# REST 엔드포인트 자동분석 -> GraphQL 스키마 생성(쿼리네임: customers)
# CUSTOMERS API
stepzen import curl "https://introspection.apis.stepzen.com/customers" --query-name "customers" 

# 새 REST API 추가
# ORDERS API
stepzen import curl "https://introspection.apis.stepzen.com/orders" --query-name "orders" --query-type "Order"

```

- curl/에 Customers api, curl-01/에 orders api 생성된거 확인 가능
- product-demo/index.graphql에선 두 스키마가 참조되고 있음

```bash
# 엔드포인트 시작
stepzen start

# 엔트포인트 테스트
-> Stepzen에서 쿼리 테스트 및 결과 데이터 확인

# PostfreSQL 을 graphQL로 변환
stepzen import postgresql \
  --db-host='postgresqlaws.introspection.stepzen.net' \
  --db-database='introspection' \
  --db-user='testUserIntrospection' \
  --db-password='HurricaneStartingSample1934' \
  --name=postgresql
```
- 새로운 쿼리 작성 필요없이 기존 PostgreSQL DB를 GraphQL로 자동 노출 가능하다.

### 커스텀 쿼리 정의
type Query 블록 아래로 getAddressById 함수 작성해 삽입
```bash
getAddressById(id: Int!): [Address]
  @dbquery(
    type: "postgresql"
    schema: "public"
    query: """
      SELECT * FROM "address" WHERE "id" = $1
    """
    configuration: "postgresql_config"
  )
```
- getAddressById(id: Int!): [Address]
    - id를 입력받아 Address 객체 리스트를 반환하는 GraphQL 쿼리

- @dbquery
    - Stepzen 디렉티브. 이 필드를 DB 쿼리 결과와 연결한다.

- type, schema, query
    - PostgreSQL query를 직접 지정한다.

- $1
    - 파라미터 id 바인딩 자리
- configuration
    - DB 연결정보: config.yaml에서 지정.

### Stepzen을 이용한 REST API, PostgreSQL 데이터 Federation.
> REST API 기반 GraphQL 스키마에 포함된 address 데이터를 PostgreSQL DB에서 조회하도로 리디렉션해보자!
> 즉 데이터 소스를 REST에서 PostgreSQL로 변경.

- 1. 현재 REST와 PostgreSQL 모두 Address 정의해서 충돌발생함. PostgreSQL 쪽으로 일원화하자.
- 2. RootEntry 타입의 address 필드를 materializer로 변경. 
```bash
address: [Address] #원래 정의된건 REST에서 가져온 것
@materializer(query: "getAddressById") # materializer로 PostgreSQL에서 정의한 getAddressById 쿼리를 사용하도로 바꿈.
```
    - 그럼 이제 RootEntry에 들어있는 address 필드는 PostgreSQL에서 직접조회된다.
==> 원래 Customer는 REST, Address는 Postgre로 제공되는 분리된 시스탬. 이걸 @materializer를 이용해 GraphQL에서 하나의 API로 보이게함. (stitching)