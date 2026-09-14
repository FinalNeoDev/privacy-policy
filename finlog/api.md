# FinLog 커스텀 동기화 API 규격 및 연동 가이드

FinLog는 사용자의 금융 데이터에 대한 완전한 주권을 보장하기 위해 개발사 중앙 수집 서버를 운영하지 않는 100% 온디바이스 가계부 앱입니다.

사용자는 본인이 소유한 개인 인프라(Google 스프레드시트, NAS, 홈서버, 클라우드 VPS)로 거래 내역을 실시간 웹훅(Webhook) 또는 REST API로 자동 동기화할 수 있습니다. 본 문서는 FinLog 앱과 연동할 수 있는 서버 규격과 트랙별 구현 예제를 제공합니다.

---

## 📌 0. FinLog 앱 송신 데이터 규격 (JSON Payload)

FinLog 앱의 **[온라인 백업 / 자동 동기화]** 기능 활성화 시, 설정된 엔드포인트(`POST`)로 다음과 같은 JSON 페이로드가 전송됩니다.

### 요청 헤더 (Headers)
```http
POST /api/v1/finlog/logs HTTP/1.1
Host: your-server.com:8080
Content-Type: application/json
Authorization: Bearer <사용자가 앱에 설정한 인증 토큰>
```

### JSON Body 예시
```json
{
  "app": "FinLog",
  "version": "0.0.7",
  "syncedAt": "2026-09-14T15:00:00.000Z",
  "sheetName": "FinLog_Sheet",
  "transactions": [
    {
      "id": "tx_1726300000_abc",
      "sourceType": "NOTIFICATION",
      "sourceApp": "com.kakaobank.channel",
      "bankName": "카카오뱅크",
      "amount": 15000,
      "type": "EXPENSE",
      "merchant": "스타벅스 강남점",
      "category": "카페/간식",
      "memo": "아메리카노 결제",
      "balance": 245000,
      "rawText": "[카카오뱅크] 체크카드 승인 15,000원 스타벅스 강남점",
      "timestamp": "2026-09-14T15:00:00.000Z"
    }
  ],
  "detectionLogs": [
    {
      "id": "audit_1726300010_xyz",
      "transactionId": "tx_1726300000_abc",
      "timestamp": "2026-09-14T15:05:00.000Z",
      "fieldName": "category",
      "oldValue": "기타",
      "newValue": "카페/간식",
      "reason": "사용자 카테고리 수동 수정"
    }
  ]
}
```

### 필드 상세 설명

#### 1) 루트 객체 (Root Payload)
| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `app` | String | 필수 | 애플리케이션 이름 (`"FinLog"`) |
| `version` | String | 필수 | 앱 버전 (예: `"0.0.7"`) |
| `syncedAt` | String (ISO 8601) | 필수 | 동기화 실행 시각 UTC (예: `"2026-09-14T15:00:00.000Z"`) |
| `sheetName` | String | 선택 | 시트 연동 시 대상 시트 탭 이름 (기본: `"FinLog_Sheet"`) |
| `transactions` | Array<Object> | 필수 | 전송 대상 거래 내역 배열 (배치 전송) |
| `detectionLogs` | Array<Object> | 선택 | 수기 변경 및 감지 이력 감사 로그 배열 |

#### 2) 거래 객체 (`transactions[]`)
| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `id` | String | 필수 | 고유 거래 ID (예: `"tx_1726300000_abc"`) |
| `sourceType` | String | 필수 | 발생 경로 (`"NOTIFICATION"`: 푸시 알림, `"SMS"`: 문자, `"MANUAL"`: 수기) |
| `sourceApp` | String | 선택 | 알림 발생 패키지명 (예: `"com.kakaobank.channel"`) |
| `bankName` | String | 필수 | 금융기관/카드사명 (예: `"카카오뱅크"`, `"국민카드"`) |
| `amount` | Number | 필수 | 결제/거래 금액 (양수 정수 또는 실수) |
| `type` | String | 필수 | 거래 구분 (`"EXPENSE"`: 지출, `"INCOME"`: 수입, `"TRANSFER"`: 이체) |
| `merchant` | String | 필수 | 가맹점/상호명 (예: `"스타벅스 강남점"`) |
| `category` | String | 필수 | 카테고리 분류 (예: `"식비"`, `"카페/간식"`, `"교통"`) |
| `memo` | String | 선택 | 사용자 메모 또는 세부 내역 |
| `balance` | Number | 선택 | 계좌 잔액 (알림 텍스트에 잔액이 포함된 경우) |
| `rawText` | String | 필수 | 원본 푸시 알림 또는 SMS 원문 텍스트 |
| `timestamp` | String (ISO 8601) | 필수 | 실제 결제 발생 시각 UTC |

#### 3) 수정 감사 로그 (`detectionLogs[]`)
| 필드명 | 타입 | 필수 여부 | 설명 |
| :--- | :--- | :---: | :--- |
| `id` | String | 필수 | 고유 감사 로그 ID (예: `"audit_1726300010_xyz"`) |
| `transactionId` | String | 필수 | 대상 거래 ID |
| `timestamp` | String (ISO 8601) | 필수 | 수정 시각 UTC |
| `fieldName` | String | 필수 | 변경된 필드명 (예: `"category"`, `"amount"`) |
| `oldValue` | String | 선택 | 변경 전 값 |
| `newValue` | String | 선택 | 변경 후 값 |
| `reason` | String | 선택 | 변경 사유 (예: `"사용자 카테고리 수동 수정"`) |

---

## 🟢 종류 1. 트랙 1 : 구글 스프레드시트 (Google Apps Script Webhook)

별도의 백엔드 서버나 데이터베이스 구축 없이 구글 드라이브 엑셀 시트에 금융 내역을 실시간 행(Row)으로 자동 누적할 때 사용합니다.

- **연결 방식**: 구글 시트 > [확장 프로그램] > [Apps Script] 배포 URL
- **장점**: 100% 무료, 서버 유지보수 불필요, PC 구글 시트에서 엑셀 수식(`SUM`, `QUERY`, 피벗테이블) 즉시 활용 가능

### Google Apps Script 코드
```javascript
// [구글 Apps Script 코드]
function doPost(e) {
  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var sheet = ss.getActiveSheet();
    
    // 1. 첫 행이 비어있으면 엑셀 헤더 자동 생성
    if (sheet.getLastRow() === 0) {
      sheet.appendRow(['기록일시', '금융기관', '구분', '금액', '가맹점/상호', '카테고리', '잔액', '메모', '원본메시지']);
      sheet.getRange(1, 1, 1, 9)
        .setBackground('#1e293b')
        .setFontColor('#ffffff')
        .setFontWeight('bold');
      sheet.setFrozenRows(1);
    }
    
    // 2. FinLog 앱에서 보낸 데이터 파싱
    var postData = JSON.parse(e.postData.contents);
    var transactions = postData.transactions || [];
    
    // 단일 전송 및 다건 전송 모두 수용
    if (postData.transaction) transactions = [postData.transaction];
    
    // 3. 시트에 한 줄씩 추가
    transactions.forEach(function(tx) {
      var typeText = tx.type === 'EXPENSE' ? '지출' : (tx.type === 'INCOME' ? '수입' : '이체');
      sheet.appendRow([
        tx.timestamp || new Date().toLocaleString(),
        tx.bankName || '기타금융',
        typeText,
        tx.amount || 0,
        tx.merchant || '',
        tx.category || '기타',
        tx.balance || '',
        tx.memo || '',
        tx.rawText || ''
      ]);
    });
    
    return ContentService.createTextOutput(JSON.stringify({ status: "success", count: transactions.length }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ status: "error", message: err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// 연결 테스트용 (Ping)
function doGet(e) {
  return ContentService.createTextOutput(JSON.stringify({ status: "ok", service: "FinLog Google Sheets Webhook" }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### 구글 스프레드시트 배포 방법
1. 구글 스프레드시트를 신규 생성합니다.
2. 상단 메뉴에서 **확장 프로그램** > **Apps Script**를 클릭합니다.
3. 기본 코드를 지우고 위 자바스크립트 코드를 붙여넣은 뒤 저장(`Ctrl+S`)합니다.
4. 우측 상단 **[배포]** > **[새 배포]** 버튼을 누릅니다.
5. 유형 선택: **웹 앱(Web app)** 선택
   - 설명: `FinLog Webhook`
   - 다음 사용자로 실행: `나(본인 계정)`
   - 액세스 권한이 있는 사용자: **모든 사용자(Anyone)** 로 설정 (중요)
6. **배포**를 누르고 생성된 **웹 앱 URL**을 복사하여 FinLog 앱의 Webhook URL에 입력합니다.

---

## 🔵 종류 2. 트랙 2 : Node.js (Express) 예제 서버

홈서버, Synology NAS(Docker), 클라우드 VPS(AWS, Oracle Cloud 등)에서 Node.js 환경으로 수신할 때 사용하는 표준 예제입니다.

- **동작**:
  - `GET /api/v1/finlog/health` : FinLog 앱의 [연결 테스트 (Ping)] 버튼 확인용
  - `POST /api/v1/finlog/logs` : 실제 거래 내역 및 감지로그 수신 + `finlog_records.jsonl` 파일에 안전하게 누적
  - `Bearer Token` 인증 검증 지원

### `server.js`
```javascript
// server.js
const express = require('express');
const cors = require('cors');
const fs = require('fs');
const app = express();
const PORT = 8080;
const AUTH_TOKEN = "my-secret-token"; // 앱의 [인증 토큰] 칸과 일치시킴

app.use(cors());
app.use(express.json());

// 1. Health Check (앱의 "연결 테스트 (Ping)" 버튼이 호출)
app.get('/api/v1/finlog/health', (req, res) => {
  const auth = req.headers['authorization'];
  if (AUTH_TOKEN && auth !== `Bearer ${AUTH_TOKEN}`) {
    return res.status(401).json({ error: '인증 실패' });
  }
  res.json({ status: 'ok', serverTime: new Date().toISOString() });
});

// 2. 로그 수신 엔드포인트 (앱의 "지금 전체 로그 전송" 및 자동 동기화가 호출)
app.post('/api/v1/finlog/logs', (req, res) => {
  const auth = req.headers['authorization'];
  if (AUTH_TOKEN && auth !== `Bearer ${AUTH_TOKEN}`) {
    return res.status(401).json({ error: '인증 실패' });
  }
  const { app: appName, version, transactions, detectionLogs, syncedAt } = req.body;
  console.log(`[${syncedAt}] 거래 ${transactions?.length || 0}건, 수정로그 ${detectionLogs?.length || 0}건 수신`);
  
  // 로컬 파일 (JSONL) 또는 MariaDB/PostgreSQL 등에 저장
  const logLine = JSON.stringify(req.body) + '\n';
  fs.appendFileSync('finlog_records.jsonl', logLine, 'utf8');
  
  res.json({
    success: true,
    receivedTransactions: transactions?.length || 0,
    receivedAuditLogs: detectionLogs?.length || 0
  });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`FinLog Express 서버가 포트 ${PORT}에서 실행 중입니다.`);
});
```

### 실행 방법
```bash
npm init -y
npm install express cors
node server.js
```

---

## 🟢 종류 3. 트랙 2 : Python (FastAPI) 예제 서버

파이썬 환경에서 데이터 분석, 판다스(Pandas) 연계, 비동기 고성능 REST API를 운영하고자 할 때 가장 추천하는 방식입니다.

- **특징**: Pydantic을 이용한 자동 스키마 유효성 검증, Swagger 문서(`http://서버IP:8080/docs`) 자동 생성

### `main.py`
```python
# main.py
from fastapi import FastAPI, Header, HTTPException, status
from pydantic import BaseModel
from typing import List, Optional
import json

app = FastAPI(title="FinLog Sync Server", version="0.0.7")
AUTH_TOKEN = "my-secret-token"  # 앱의 인증 토큰과 일치

# 데이터 모델 정의
class FinTransaction(BaseModel):
    id: str
    sourceType: str
    bankName: str
    amount: float
    type: str
    merchant: str
    category: str
    memo: Optional[str] = None
    balance: Optional[float] = None
    rawText: str
    timestamp: str

class SyncPayload(BaseModel):
    app: str
    version: str
    syncedAt: str
    transactions: List[dict]
    detectionLogs: Optional[List[dict]] = []

# 1. Health Check
@app.get("/api/v1/finlog/health")
def health_check(authorization: Optional[str] = Header(None)):
    if AUTH_TOKEN and authorization != f"Bearer {AUTH_TOKEN}":
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="인증 실패")
    return {"status": "ok", "version": "0.0.7"}

# 2. 로그 수신 엔드포인트
@app.post("/api/v1/finlog/logs")
def receive_logs(payload: SyncPayload, authorization: Optional[str] = Header(None)):
    if AUTH_TOKEN and authorization != f"Bearer {AUTH_TOKEN}":
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="인증 실패")
    
    print(f"[{payload.syncedAt}] 거래 {len(payload.transactions)}건 수신 완료")
    
    # JSONL 파일에 한 줄씩 누적 저장 (향후 Pandas나 SQLite로 분석 가능)
    with open("finlog_records.jsonl", "a", encoding="utf-8") as f:
        f.write(payload.model_dump_json() + "\n")
        
    return {
        "success": True,
        "count": len(payload.transactions)
    }
```

### 실행 방법
```bash
pip install fastapi uvicorn
uvicorn main:app --host 0.0.0.0 --port 8080
```

---

## 🟡 종류 4. 트랙 2 : Python (Flask) 초간단 예제 서버

FastAPI보다 더 가볍게, 단 몇 줄의 파이썬 코드로 라즈베리파이나 NAS에서 띄우고 싶을 때 적합합니다.

### `app.py`
```python
# app.py
from flask import Flask, request, jsonify
import json

app = Flask(__name__)
AUTH_TOKEN = "my-secret-token"

@app.route('/api/v1/finlog/health', methods=['GET'])
def health():
    auth = request.headers.get('Authorization')
    if AUTH_TOKEN and auth != f"Bearer {AUTH_TOKEN}":
        return jsonify({"error": "Unauthorized"}), 401
    return jsonify({"status": "ok"}), 200

@app.route('/api/v1/finlog/logs', methods=['POST'])
def logs():
    auth = request.headers.get('Authorization')
    if AUTH_TOKEN and auth != f"Bearer {AUTH_TOKEN}":
        return jsonify({"error": "Unauthorized"}), 401
    data = request.get_json()
    with open('finlog_records.jsonl', 'a', encoding='utf-8') as f:
        f.write(json.dumps(data, ensure_ascii=False) + '\n')
    return jsonify({"success": True, "count": len(data.get('transactions', []))}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### 실행 방법
```bash
pip install flask
python app.py
```

---

## 💻 종류 5. cURL 테스트 명령 (개발자 로컬 테스트용)

개인 서버를 띄운 후, FinLog 앱 없이 터미널(CMD/PowerShell/Bash)에서 서버가 정상 동작하는지 테스트하는 명령입니다.

### 1) Health Check (Ping) 테스트
```bash
curl -X GET http://localhost:8080/api/v1/finlog/health \
     -H "Authorization: Bearer my-secret-token"
```

### 2) 가상 거래 데이터 전송 테스트
```bash
curl -X POST http://localhost:8080/api/v1/finlog/logs \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer my-secret-token" \
     -d '{
       "app": "FinLog",
       "version": "0.0.7",
       "syncedAt": "2026-09-14T15:00:00Z",
       "sheetName": "FinLog_Sheet",
       "transactions": [
         {
           "id": "test_01",
           "sourceType": "NOTIFICATION",
           "bankName": "테스트은행",
           "type": "EXPENSE",
           "amount": 10000,
           "merchant": "테스트가맹점",
           "category": "식비",
           "timestamp": "2026-09-14T15:00:00Z",
           "rawText": "테스트 결제 10,000원"
         }
       ]
     }'
```

---

## 💡 요약 및 권장 사항

1. **가장 간편한 방법**:
   - 별도 서버나 고정 IP가 없다면 **[종류 1: 구글 스프레드시트(Apps Script)]** 로 시작하시는 것이 가장 편리합니다.
2. **개인 홈서버 / NAS / VPS 구축 시**:
   - Node.js 환경: **[종류 2: Express]**
   - Python 환경: 데이터 분석 및 안정성을 위해 **[종류 3: FastAPI]** 를 추천합니다.
3. **FinLog 앱 내 [온라인 백업 / 자동 동기화] 설정값 매핑**:
   - **통신 프로토콜**: `http` 또는 `https`
   - **서버 주소**: 서버 IP 또는 도메인 (예: `192.168.0.50` 또는 `mynas.me`)
   - **포트**: `8080` (또는 설정한 포트)
   - **엔드포인트**: `/api/v1/finlog/logs`
   - **인증 토큰**: 서버 코드의 `AUTH_TOKEN` (예: `my-secret-token`)과 동일하게 입력
