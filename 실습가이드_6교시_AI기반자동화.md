# 📘 6교시 실습 가이드 — AI 기반 교육서비스 자동화 실습

> **Day 2 | 6교시 (1시간)**
> 학습 목표: Q&A 자동 응답 봇 & 피드백 AI 요약 리포트를 구현한다

---

## 실습 11: 수강생 Q&A 자동 응답 봇

### 목표
수강생이 질문을 보내면 FAQ를 참고하여 ChatGPT가 자동으로 답변을 생성합니다.

### 전체 구조
```
Webhook (질문 수신) → Google Sheets (FAQ 읽기) → Code (FAQ 결합)
    → OpenAI (답변 생성) → Google Sheets (로그 저장)
    → Respond to Webhook (답변 반환)
```

---

### Step 1: FAQ 데이터 시트 만들기

1. Google Sheets에서 새 스프레드시트: **"FAQ_데이터"**

2. 데이터 입력:

   | 질문 | 답변 |
   |------|------|
   | 수업 시간은 어떻게 되나요? | 오전반은 09:00~12:00, 오후반은 14:00~17:00 입니다. |
   | 수강료는 얼마인가요? | AI입문 과정은 50만원, 데이터분석 과정은 60만원입니다. |
   | 주차는 가능한가요? | 건물 지하 1층에 무료 주차가 가능합니다. 주차 등록은 안내데스크에서 해주세요. |
   | 노트북을 꼭 가져와야 하나요? | 네, 실습 수업이므로 개인 노트북이 필수입니다. 대여가 필요하시면 사전에 연락주세요. |
   | 수업을 못 갔을 경우 보강이 되나요? | 녹화 영상을 제공합니다. 수업 다음 날 이메일로 링크를 보내드립니다. |
   | 수료증이 발급되나요? | 출석 80% 이상 시 수료증이 이메일로 자동 발급됩니다. |
   | 환불은 어떻게 하나요? | 수업 시작 7일 전까지 100% 환불, 3일 전까지 50% 환불 가능합니다. |
   | Wi-Fi가 제공되나요? | 네, 강의실에서 무료 Wi-Fi를 사용하실 수 있습니다. 접속 정보는 수업 당일 안내됩니다. |

3. 별도 시트 탭 추가: **"QA로그"**
   - 헤더: 질문, AI답변, 응답시간, 성공여부

---

### Step 2: Webhook 설정

1. 새 워크플로우: **"실습11_QA자동응답봇"**

2. **Webhook** 노드 추가:
   - **Method**: `POST`
   - **Path**: `qa-bot`
   - **Response Mode**: `Last Node`

---

### Step 3: FAQ 데이터 불러오기

1. Webhook 뒤에 **Google Sheets** (Read Rows) 추가:
   - Document: "FAQ_데이터"
   - Sheet: "Sheet1" (FAQ 시트)

---

### Step 4: Code 노드 — FAQ를 프롬프트용 텍스트로 결합

1. Google Sheets 뒤에 **Code** 노드 추가

2. 코드 입력:

   ```javascript
   // FAQ 데이터를 문자열로 변환
   const faqItems = $input.all();
   
   const faqText = faqItems.map(item => 
     `Q: ${item.json.질문}\nA: ${item.json.답변}`
   ).join('\n\n');
   
   // Webhook으로 받은 질문 가져오기
   // $('Webhook') 으로 Webhook 노드 데이터에 접근
   const webhookData = $('Webhook').first().json;
   const userQuestion = webhookData.body.question;
   
   return [{
     json: {
       faqContext: faqText,
       userQuestion: userQuestion
     }
   }];
   ```

   > ⚠️ `$('Webhook')` 은 워크플로우 내 "Webhook" 이라는 이름의 노드 데이터를 참조합니다.
   > 노드 이름이 다르면 해당 이름으로 변경하세요.

---

### Step 5: OpenAI 노드 — AI 답변 생성

1. Code 노드 뒤에 **OpenAI** 노드 추가

2. 설정:
   - **Model**: `gpt-4o-mini`
   - **Messages**:
   
   **Message 1 (System)**:
   ```
   당신은 교육 센터의 친절한 상담 어시스턴트입니다.
   
   아래 FAQ 정보를 참고하여 수강생의 질문에 답변해주세요.
   
   규칙:
   1. FAQ에 있는 정보를 기반으로 정확하게 답변하세요.
   2. FAQ에 없는 질문이면 "죄송합니다. 해당 문의는 담당자에게 연결해드리겠습니다. 전화: 02-1234-5678, 이메일: help@education.com" 이라고 답변하세요.
   3. 항상 친절하고 공손한 톤을 유지하세요.
   4. 답변은 2~3문장으로 간결하게 해주세요.
   
   === FAQ 정보 ===
   {{ $json.faqContext }}
   ```

   **Message 2 (User)**:
   ```
   {{ $json.userQuestion }}
   ```

---

### Step 6: Q&A 로그 저장

1. OpenAI 뒤에 **Edit Fields** 노드 추가:

   | Name | Value |
   |------|-------|
   | `질문` | `{{ $('Code').first().json.userQuestion }}` |
   | `AI답변` | `{{ $json.message.content }}` |
   | `응답시간` | `{{ $now.format('YYYY-MM-DD HH:mm:ss') }}` |
   | `성공여부` | `성공` |

2. Edit Fields 뒤에 **Google Sheets** (Append Row) 추가:
   - Document: "FAQ_데이터"
   - Sheet: "QA로그"

---

### Step 7: 응답 반환

1. Google Sheets 뒤에 **Respond to Webhook** 노드 추가:
   - **Response Code**: 200
   - **Response Body** (JSON):
     ```json
     {
       "answer": "{{ $json.AI답변 }}",
       "source": "AI Assistant",
       "timestamp": "{{ $json.응답시간 }}"
     }
     ```

---

### Step 8: 테스트

1. Webhook 노드 → **"Listen for test event"** 클릭

2. 별도 **테스트 워크플로우** 또는 HTTP Request로 질문 전송:
   ```json
   {
     "question": "수업 시간이 어떻게 되나요?"
   }
   ```

3. 응답 확인:
   ```json
   {
     "answer": "오전반은 09:00~12:00, 오후반은 14:00~17:00에 진행됩니다. 원하시는 시간대를 선택하여 신청해주세요!",
     "source": "AI Assistant",
     "timestamp": "2026-03-23 10:30:00"
   }
   ```

4. FAQ에 없는 질문도 테스트:
   ```json
   {
     "question": "기숙사가 있나요?"
   }
   ```
   → "담당자에게 연결" 응답이 나오는지 확인

5. Google Sheets "QA로그" 시트에 로그가 저장되는지 확인

> ✅ **산출물**: AI 기반 Q&A 자동 응답 봇 워크플로우 완성!

---

## 실습 12: 교육 피드백 AI 요약 리포트

### 목표
축적된 피드백 데이터를 ChatGPT가 분석·요약하여 관리자에게 리포트를 발송합니다.

---

### Step 1: 워크플로우 구성

1. 새 워크플로우: **"실습12_피드백AI요약"**

2. **Manual Trigger** 추가

3. **Google Sheets** (Read Rows):
   - Document: "교육피드백_DB" (4교시에서 생성)
   - Sheet: "피드백"

---

### Step 2: 피드백 데이터 결합

1. **Code** 노드 추가:

   ```javascript
   const items = $input.all();
   
   // 피드백 텍스트를 하나로 결합
   const feedbackList = items.map((item, index) => 
     `[수강생 ${index + 1}] 만족도: ${item.json.만족도}/5\n피드백: ${item.json.피드백}`
   ).join('\n\n---\n\n');
   
   // 기본 통계
   const scores = items.map(i => Number(i.json.만족도));
   const avg = (scores.reduce((a, b) => a + b, 0) / scores.length).toFixed(1);
   const courseName = items[0].json.과정명;
   
   return [{
     json: {
       과정명: courseName,
       응답수: items.length,
       평균만족도: avg,
       전체피드백: feedbackList
     }
   }];
   ```

---

### Step 3: OpenAI로 분석 요청

1. Code 노드 뒤에 **OpenAI** 노드 추가

2. 설정:

   **System Prompt**:
   ```
   당신은 교육 품질 분석 전문가입니다.
   수강생 피드백을 분석하여 구조화된 리포트를 작성해주세요.
   한국어로 답변하세요.
   ```

   **User Message**:
   ```
   아래는 "{{ $json.과정명 }}" 과정의 수강생 피드백입니다.
   총 {{ $json.응답수 }}명이 응답했으며, 평균 만족도는 {{ $json.평균만족도 }}/5.0 입니다.
   
   === 수강생 피드백 ===
   {{ $json.전체피드백 }}
   === 끝 ===
   
   위 피드백을 분석하여 아래 항목을 정리해주세요:
   
   1. **전체 만족도 요약** (2~3문장)
   2. **긍정적 피드백 Top 3** (수강생들이 좋아한 점)
   3. **개선 요청 사항 Top 3** (수강생들이 불만족한 점)
   4. **다음 교육을 위한 제언** (구체적 개선 방안 3가지)
   5. **한 줄 총평**
   ```

3. **Model**: `gpt-4o-mini`
4. **Max Tokens**: 1500 (충분한 분석 결과를 위해)

---

### Step 4: 리포트 저장 & 이메일 발송

1. OpenAI 뒤에 **Edit Fields** 노드:

   | Name | Value |
   |------|-------|
   | `과정명` | `{{ $('Code').first().json.과정명 }}` |
   | `평균만족도` | `{{ $('Code').first().json.평균만족도 }}` |
   | `응답수` | `{{ $('Code').first().json.응답수 }}` |
   | `집계일시` | `{{ $now.format('YYYY-MM-DD HH:mm') }}` |
   | `AI요약` | `{{ $json.message.content }}` |

2. **Google Sheets** (Append Row):
   - Document: "교육피드백_DB"
   - Sheet: "리포트"

3. **Gmail** 노드:
   - **To**: 관리자 이메일 (본인 이메일로 테스트)
   - **Subject**: `📊 [{{ $json.과정명 }}] 교육 피드백 AI 분석 리포트 — {{ $json.집계일시 }}`
   - **Message**:
     ```html
     <h1>📊 교육 피드백 분석 리포트</h1>
     <table border="1" cellpadding="10" style="border-collapse:collapse;margin-bottom:20px;">
       <tr style="background:#0D7377;color:white;">
         <td><strong>과정명</strong></td>
         <td><strong>평균 만족도</strong></td>
         <td><strong>응답 수</strong></td>
         <td><strong>집계 일시</strong></td>
       </tr>
       <tr>
         <td>{{ $json.과정명 }}</td>
         <td><strong>{{ $json.평균만족도 }}</strong> / 5.0</td>
         <td>{{ $json.응답수 }}명</td>
         <td>{{ $json.집계일시 }}</td>
       </tr>
     </table>
     
     <h2>AI 분석 결과</h2>
     <div style="background:#f5f5f5;padding:16px;border-radius:8px;white-space:pre-wrap;">
     {{ $json.AI요약 }}
     </div>
     
     <hr>
     <p><small>이 리포트는 n8n + ChatGPT에 의해 자동 생성되었습니다.</small></p>
     ```

---

### Step 5: 테스트

1. **"Test workflow"** 실행

2. 확인 사항:
   - OpenAI 노드 Output에서 분석 리포트 내용 확인
   - Google Sheets "리포트" 시트에 저장 확인
   - 이메일 수신 확인

> ✅ **산출물**: 피드백 AI 분석 → 요약 리포트 → Sheets 저장 → 이메일 발송 완성!

---

## 보너스: 교육 콘텐츠 초안 자동 생성

### 빠른 실습 (5분)

1. 새 워크플로우에서 Manual Trigger → OpenAI 연결

2. System: `교육 과정 설계 전문가. 한국어로 답변.`

3. User:
   ```
   아래 주제로 2시간 교육 슬라이드 구성안을 만들어주세요.
   각 슬라이드별로 제목, 핵심 내용 3줄, 실습 활동을 포함해주세요.
   총 15~20개 슬라이드로 구성해주세요.
   
   주제: Python 데이터 분석 입문
   대상: IT 비전공 직장인
   ```

4. 테스트 실행 → AI가 생성한 슬라이드 구성안 확인

> 💡 이 워크플로우를 확장하면 주간 뉴스레터, 블로그 초안, 마케팅 문구도 자동 생성 가능!

---

## 6교시 산출물 체크리스트

- [ ] [실습 11] AI Q&A 자동 응답 봇 (FAQ 컨텍스트 주입)
- [ ] [실습 12] 피드백 AI 분석 요약 리포트 자동 생성·발송
- [ ] Code 노드에서 여러 노드 데이터 참조 방법 이해
- [ ] 프롬프트 엔지니어링 심화 (구조화된 분석 요청) 이해

---

> **다음 교시 예고**: 7교시에서는 2일간 배운 모든 기술을 종합하여 팀별 프로젝트를 설계·구현합니다!
