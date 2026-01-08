---
layout: post
title: 주요 데이터베이스의 MCP 지원 현황과 활용
author: 'Juho'
date: 2026-01-08 00:00:00 +0900
categories: [MCP]
tags: [MCP, Database, LLM, PostgreSQL, AI]
pin: True
toc: True
---

<style>
  th{
    font-weight: bold;
    text-align: center;
    background-color: white;
  }
  td{
    background-color: white;
  }

</style>

## 목차
1. [MCP와 데이터베이스 통합](#mcp와-데이터베이스-통합)
2. [MCP를 지원하는 주요 데이터베이스](#mcp를-지원하는-주요-데이터베이스)
3. [MCP의 핵심 기능](#mcp의-핵심-기능)
4. [실제 활용 사례: Neon](#실제-활용-사례-neon)
5. [보안 고려사항](#보안-고려사항)
6. [구현 예시](#구현-예시)
7. [제한사항과 미래 방향](#제한사항과-미래-방향)

## MCP와 데이터베이스 통합
2025년은 데이터베이스 업계에서 중요한 전환점이 되었다.  
Anthropic의 MCP(Model Context Protocol) 표준을 주요 DBMS들이 공식적으로 채택하면서, LLM이 데이터베이스와 직접 상호작용할 수 있는 시대가 열렸다.  

### MCP란?
MCP는 **JSON-RPC 기반 인터페이스**로, LLM과 다양한 데이터 소스 간의 표준화된 통신을 가능하게 한다.  
데이터베이스 영역에서 MCP는 다음과 같은 역할을 한다:  

- 자연어 쿼리를 SQL로 변환
- 스키마 정보 자동 탐색
- 쿼리 결과를 LLM이 이해할 수 있는 형태로 변환
- 데이터베이스 작업의 컨텍스트 유지

### 도입 배경
MCP의 대규모 채택은 다음과 같은 타임라인으로 진행되었다:

- **2024년 11월**: Anthropic이 MCP 프로토콜 최초 발표  
- **2025년 3월**: OpenAI가 공식적으로 MCP 지원 발표  
- **2025년 중반**: 거의 모든 주요 데이터베이스 벤더가 MCP 서버 출시  

2025년은 모든 주요 DBMS가 Anthropic의 MCP 표준을 채택한 해로 기록된다.  
각 데이터베이스 벤더가 자체적으로 AI 통합 방식을 개발하는 대신, **통일된 표준을 통해 효율적으로 LLM과 연동**할 수 있게 되었다.

## MCP를 지원하는 주요 데이터베이스
현재 MCP를 공식 지원하는 주요 데이터베이스는 다음과 같다:

### 1. 엔터프라이즈 DBMS
- **Oracle**: 엔터프라이즈 환경에서의 MCP 통합
- **Snowflake**: 클라우드 데이터 웨어하우스와의 자연어 쿼리

### 2. 오픈소스 및 NoSQL
- **MongoDB**: 문서 기반 쿼리의 자연어 변환
- **ClickHouse**: 실시간 분석 쿼리 최적화

### 3. PostgreSQL 생태계
PostgreSQL 기반 서비스들은 각자 고유한 MCP 서버를 제공한다:

#### Supabase
```json
{
  "mcpServers": {
    "supabase": {
      "command": "npx",
      "args": ["-y", "@supabase/mcp-server"],
      "env": {
        "SUPABASE_URL": "your-project-url",
        "SUPABASE_KEY": "your-anon-key"
      }
    }
  }
}
```

#### Timescale
시계열 데이터에 최적화된 쿼리 지원
```json
{
  "mcpServers": {
    "timescale": {
      "command": "npx",
      "args": ["-y", "@timescale/mcp-server"],
      "env": {
        "DATABASE_URL": "postgresql://user:password@host:port/db"
      }
    }
  }
}
```

#### Xata
서버리스 PostgreSQL과 검색 기능 통합
```json
{
  "mcpServers": {
    "xata": {
      "command": "npx",
      "args": ["-y", "@xata/mcp-server"],
      "env": {
        "XATA_API_KEY": "your-api-key",
        "XATA_DATABASE_URL": "your-database-url"
      }
    }
  }
}
```

## MCP의 핵심 기능
데이터베이스 MCP 서버가 제공하는 주요 기능은 다음과 같다:

### 1. 단일 요청 단위 접근
```
사용자: "지난 주 매출이 가장 높은 상위 5개 제품을 보여줘"

LLM + MCP:
1. 자연어 이해
2. SQL 쿼리 생성
   SELECT product_name, SUM(sales_amount) as total
   FROM sales
   WHERE sale_date >= CURRENT_DATE - INTERVAL '7 days'
   GROUP BY product_name
   ORDER BY total DESC
   LIMIT 5
3. 실행 및 결과 반환
4. 자연어로 결과 설명
```

### 2. 스키마 탐색
MCP 서버는 데이터베이스 스키마를 자동으로 LLM에 제공하여, 테이블 구조, 관계, 데이터 타입을 이해하고 적절한 쿼리를 생성할 수 있게 한다.

### 3. 읽기 전용 작업
대부분의 MCP 구현은 안전을 위해 **읽기 전용(Read-Only)** 모드로 동작한다:
- `SELECT` 쿼리만 허용
- `INSERT`, `UPDATE`, `DELETE` 작업 차단
- 스키마 변경 불가

### 4. 쿼리 보호 메커니즘
```python
# 예시: 쿼리 제한 설정
MCP_CONFIG = {
    "timeout": 30,              # 30초 쿼리 타임아웃
    "max_rows": 1000,           # 최대 1000행 반환
    "read_only": True,          # 읽기 전용 모드
    "allowed_tables": [         # 접근 가능한 테이블 제한
        "products",
        "sales",
        "customers"
    ]
}
```

## 실제 활용 사례: Neon
Neon은 서버리스 PostgreSQL 플랫폼으로, MCP 지원과 함께 혁신적인 데이터 브랜칭 기능을 제공한다.

### 데이터 브랜칭의 힘
Neon의 **데이터 브랜칭(Data Branching)** 기능은 Git처럼 데이터베이스 상태를 분기하고 관리할 수 있게 한다:

```bash
# 프로덕션 데이터베이스에서 개발용 브랜치 생성
neon branches create --name dev-feature-x --parent main

# AI 에이전트가 안전하게 테스트 수행
ai-agent: "dev-feature-x 브랜치에서 새로운 인덱스 성능을 테스트해줘"
```

### 통계적 성과
**2025년 7월 기준**, Neon 플랫폼에서 **AI 에이전트가 생성한 데이터베이스가 전체의 80%**에 달한다는 놀라운 통계가 보고되었다.  
이는 다음을 의미한다:  

1. **개발 속도 향상**: 개발자가 수동으로 스키마를 설계하는 대신, AI가 요구사항을 이해하고 자동 생성
2. **실험의 용이성**: 브랜칭을 통해 안전하게 다양한 스키마 시도
3. **프로토타이핑 가속화**: 아이디어에서 구현까지의 시간 단축

이 수치는 데이터베이스 브랜칭이 AI 에이전트에게 특히 가치 있는 이유를 보여준다.  
Git 브랜치처럼 데이터베이스를 분기하고, 실험하고, 병합할 수 있는 능력은 AI가 안전하게 반복 작업을 수행할 수 있게 한다.  

### Neon MCP 설정 예시
```json
{
  "mcpServers": {
    "neon": {
      "command": "npx",
      "args": ["-y", "@neondatabase/mcp-server"],
      "env": {
        "NEON_API_KEY": "your-api-key",
        "NEON_PROJECT_ID": "your-project-id"
      }
    }
  }
}
```

## 보안 고려사항
데이터베이스와 LLM의 통합은 새로운 보안 과제를 제시한다.  

### 핵심 보안 원칙
MCP를 통한 데이터베이스 접근 시 다음 원칙을 반드시 준수해야 한다:  

- **최소 권한 계정 사용**: 제한된 권한을 가진 전용 계정만 사용
- **프록시 레이어 보호**: 자동화된 보호 메커니즘으로 비정상적인 쿼리 차단
- **감사 로그**: 모든 MCP 쿼리를 로깅하고 모니터링

MCP는 편리함을 제공하지만, 그만큼 잠재적 보안 위협도 증가한다.  
무제한 데이터베이스 접근 권한을 부여해서는 안 되며, MCP든 표준 API든 동일한 보안 원칙이 적용되어야 한다.  

### 1. 읽기 전용 제약
```sql
-- MCP 사용자 권한 설정 예시 (PostgreSQL)
CREATE USER mcp_user WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE mydb TO mcp_user;
GRANT USAGE ON SCHEMA public TO mcp_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO mcp_user;

-- 쓰기 권한은 명시적으로 제외
REVOKE INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public FROM mcp_user;
```

### 2. 쿼리 타임아웃
LLM이 생성한 비효율적 쿼리로 인한 데이터베이스 부하를 방지한다:
```python
# 타임아웃 설정
connection.execute("SET statement_timeout TO 30000")  # 30초
```

### 3. 결과 크기 제한
대량의 데이터 반환을 방지하여 네트워크 및 메모리 문제를 예방한다:
```python
def execute_query(query: str, max_rows: int = 1000):
    # LIMIT 절이 없으면 자동 추가
    if "LIMIT" not in query.upper():
        query = f"{query} LIMIT {max_rows}"
    return connection.execute(query)
```

### 4. 최소 권한 원칙
MCP 사용자는 필요한 최소한의 테이블과 컬럼에만 접근할 수 있어야 한다:
```sql
-- 특정 테이블만 접근 허용
GRANT SELECT (id, name, email) ON users TO mcp_user;
GRANT SELECT ON products TO mcp_user;

-- 민감한 테이블은 차단
REVOKE ALL ON admin_users FROM mcp_user;
```

### 5. SQL 인젝션 유사 위험
기존 SQL 인젝션과 유사하지만, 악의적 공격자 대신 **LLM의 환각(hallucination)**이 위협 요소가 된다:

```
잘못된 예:
사용자: "모든 사용자 정보를 보여줘"
LLM(환각): DROP TABLE users; SELECT * FROM users;
```

**대응 방법**:
- 쿼리 파싱 및 검증
- 화이트리스트 기반 테이블/컬럼 접근
- 자동화된 쿼리 검토 시스템

### 6. 스키마 노출 위험
MCP는 데이터베이스 스키마를 LLM에 노출하므로, 민감한 구조 정보가 유출될 수 있다:

**완화 방법**:
- 뷰(View)를 통한 추상화
- 스키마 접근 제한
- 메타데이터 필터링

## 구현 예시
Python으로 간단한 PostgreSQL MCP 서버를 구현하는 예시이다.

### 기본 구조
```python
from fastapi import FastAPI
from fastapi_mcp import FastApiMCP
import psycopg2
from psycopg2.extras import RealDictCursor

app = FastAPI()

# 데이터베이스 연결
def get_db_connection():
    return psycopg2.connect(
        host="localhost",
        database="mydb",
        user="mcp_user",
        password="secure_password",
        cursor_factory=RealDictCursor
    )

# 읽기 전용 쿼리 검증
def validate_readonly_query(query: str) -> bool:
    query_upper = query.upper().strip()

    # SELECT만 허용
    if not query_upper.startswith("SELECT"):
        return False

    # 위험한 키워드 차단
    dangerous_keywords = [
        "INSERT", "UPDATE", "DELETE", "DROP",
        "CREATE", "ALTER", "TRUNCATE", "GRANT", "REVOKE"
    ]

    for keyword in dangerous_keywords:
        if keyword in query_upper:
            return False

    return True

# MCP 도구: 쿼리 실행
@app.post("/execute-query", operation_id="execute_database_query")
async def execute_query(query: str, max_rows: int = 1000):
    """
    데이터베이스에서 읽기 전용 쿼리를 실행합니다.

    Args:
        query: 실행할 SQL SELECT 쿼리
        max_rows: 반환할 최대 행 수 (기본값: 1000)

    Returns:
        쿼리 실행 결과
    """
    # 읽기 전용 검증
    if not validate_readonly_query(query):
        return {
            "error": "Only SELECT queries are allowed",
            "query": query
        }

    # LIMIT 추가
    if "LIMIT" not in query.upper():
        query = f"{query} LIMIT {max_rows}"

    try:
        conn = get_db_connection()
        cursor = conn.cursor()

        # 타임아웃 설정
        cursor.execute("SET statement_timeout TO 30000")

        # 쿼리 실행
        cursor.execute(query)
        results = cursor.fetchall()

        cursor.close()
        conn.close()

        return {
            "success": True,
            "rows": len(results),
            "data": results
        }

    except Exception as e:
        return {
            "success": False,
            "error": str(e)
        }

# MCP 도구: 스키마 정보 조회
@app.get("/schema", operation_id="get_database_schema")
async def get_schema():
    """
    데이터베이스의 테이블과 컬럼 정보를 조회합니다.
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor()

        # 테이블 목록 조회
        cursor.execute("""
            SELECT
                table_name,
                column_name,
                data_type,
                is_nullable
            FROM information_schema.columns
            WHERE table_schema = 'public'
            ORDER BY table_name, ordinal_position
        """)

        results = cursor.fetchall()
        cursor.close()
        conn.close()

        # 테이블별로 그룹화
        schema = {}
        for row in results:
            table = row['table_name']
            if table not in schema:
                schema[table] = []

            schema[table].append({
                "column": row['column_name'],
                "type": row['data_type'],
                "nullable": row['is_nullable']
            })

        return {
            "success": True,
            "schema": schema
        }

    except Exception as e:
        return {
            "success": False,
            "error": str(e)
        }

# MCP 서버 마운트
mcp = FastApiMCP(
    app,
    name="PostgreSQL MCP Server",
    description="Read-only PostgreSQL database access via MCP",
    base_url="http://localhost:8000"
)

mcp.mount()
```

### 사용 방법
```bash
# 서버 실행
uvicorn main:app --host 0.0.0.0 --port 8000

# Claude Desktop 설정 (claude_desktop_config.json)
{
  "mcpServers": {
    "postgres-db": {
      "command": "mcp-proxy",
      "args": ["http://localhost:8000/mcp"]
    }
  }
}
```

## 제한사항과 미래 방향

### 현재 제한사항
1. **크로스 데이터베이스 조인 미지원**
   - 현재 MCP는 단일 데이터베이스 내에서만 작동
   - 이기종 데이터베이스 간 조인은 불가능
   ```
   ❌ 지원 안 됨:
   SELECT *
   FROM postgres_db.users u
   JOIN mongodb.orders o ON u.id = o.user_id
   ```

2. **분산형 구현**
   - PostgreSQL 기반 서비스들이 각자의 MCP 서버 제공
   - 표준화된 단일 구현체 부재
   - 서비스 간 이동 시 재설정 필요

3. **쓰기 작업 제한**
   - 대부분 읽기 전용으로 구현
   - 데이터 수정, 삽입, 삭제는 제한적

### 미래 발전 방향

#### 1. 통합 쿼리 레이어
```
미래 비전:
LLM이 여러 데이터 소스를 자동으로 통합하여 쿼리

사용자: "PostgreSQL의 고객 정보와 MongoDB의 주문 이력을 매칭해서 분석해줘"
LLM: 각 데이터베이스에서 데이터를 가져와 메모리에서 조인 수행
```

#### 2. 안전한 쓰기 작업
```python
# 승인 기반 쓰기 작업
@app.post("/safe-update", requires_approval=True)
async def update_data(query: str):
    """
    사용자 승인 후 UPDATE 쿼리 실행
    """
    # 1. 영향받을 행 수 미리 계산
    # 2. 사용자에게 승인 요청
    # 3. 승인 후 트랜잭션 실행
    # 4. 롤백 가능한 체크포인트 생성
```

#### 3. 지능형 쿼리 최적화
LLM이 쿼리 플랜을 분석하고 자동으로 최적화:
```sql
-- LLM이 생성한 초기 쿼리
SELECT * FROM users WHERE email LIKE '%@gmail.com'

-- 최적화된 쿼리 (MCP 서버가 자동 개선)
SELECT id, name, email
FROM users
WHERE email LIKE '%@gmail.com'
AND created_at > '2025-01-01'
LIMIT 1000
```

#### 4. 멀티 테넌트 지원
```json
{
  "mcpServers": {
    "multi-tenant-db": {
      "command": "mcp-server",
      "tenant_id": "customer_123",
      "row_level_security": true
    }
  }
}
```

## 마치며
2025년, 주요 데이터베이스들의 MCP 채택은 AI와 데이터베이스 통합의 새로운 장을 열었다.  
자연어로 데이터를 조회하고, AI 에이전트가 자동으로 데이터베이스를 생성하는 시대가 현실이 되었다.  

### 핵심 요약
1. **광범위한 채택**: ClickHouse, Snowflake, Oracle, MongoDB, PostgreSQL 생태계가 MCP 지원
2. **표준화된 인터페이스**: JSON-RPC 기반 통일된 프로토콜
3. **보안 중요성**: 최소 권한 원칙, 읽기 전용 제약, 자동화된 보호 메커니즘 필수
4. **실용적 성과**: Neon에서 AI 에이전트가 전체 데이터베이스의 80% 생성 (2025년 7월)
5. **미래 가능성**: 크로스 데이터베이스 조인, 안전한 쓰기 작업으로 확장


### 데이터베이스의 새로운 패러다임
데이터베이스 MCP 지원은 단순한 기술 트렌드가 아니라, 개발자가 데이터와 상호작용하는 방식의 **근본적 변화**를 의미한다.  

2025년은 다음이 입증된 해였다:  
- 자연어가 데이터베이스 쿼리의 유효한 인터페이스가 됨  
- AI 에이전트가 실질적으로 데이터베이스를 생성하고 관리할 수 있음  
- 데이터 브랜칭이 AI 개발 워크플로우의 핵심 도구가 됨  

자연어 인터페이스와 자동화된 데이터베이스 관리는 이제 선택이 아닌 필수가 되어가고 있다.  
하지만 동시에 보안과 거버넌스의 중요성도 그 어느 때보다 높아졌다.  
**무제한 접근 권한을 부여하지 않고, 최소 권한 원칙을 준수하며, 모든 작업을 모니터링하는 것**이 MCP 시대의 핵심 원칙이다.  
