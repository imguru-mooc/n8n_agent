# 📘 6교시 실습 가이드 — AI 기반 교육서비스 자동화 실습

> **Day 2 | 6교시 (1시간)**
> 학습 목표: 다양한 AI 활용 워크플로우를 직접 구축한다
>
> **n8n 버전: 2.12.3 기준**

---

## 6교시 전체 구성

| 실습 | 시간 | 내용 | 사용 AI |
|------|------|------|---------|
| 실습 11 | 25분 | 수강생 Q&A 자동 응답 봇 | OpenAI |
| 실습 12 | 20분 | 교육 피드백 AI 요약 리포트 | OpenAI 또는 Claude |
| 실습 13 | 10분 | 교육 콘텐츠 초안 자동 생성 | Gemini |
| 실습 14 | 5분 | 수강생 이메일 AI 자동 작성기 | 선택 |

---

---

## 실습 11: 수강생 Q&A 자동 응답 봇

### 목표
수강생이 질문을 보내면 FAQ를 참고하여 ChatGPT가 자동으로 답변을 생성합니다.

### 완성 워크플로우 구조
```
Webhook (질문 수신)
    → Google Sheets (FAQ 읽기)
    → Code (FAQ + 질문 결합)
    → OpenAI (답변 생성)
    → Edit Fields (응답 정리)
    → Google Sheets (로그 저장)
    → Respond to Webhook (답변 반환)
```

---

### Step 1: FAQ 데이터 시트 만들기

1. 브라우저에서 `sheets.google.com` 접속

2. **"빈 스프레드시트"** 클릭 → 파일 이름: **"FAQ_데이터"**

3. **Sheet1**에 아래 데이터 입력 (1행이 헤더):

   | A열(질문) | B열(답변) |
   |-----------|-----------|
   | 질문 | 답변 |
   | 수업 시간은 어떻게 되나요? | 오전반은 09:00~12:00, 오후반은 14:00~17:00 입니다. |
   | 수강료는 얼마인가요? | AI입문 과정은 50만원, 데이터분석 과정은 60만원입니다. |
   | 주차는 가능한가요? | 건물 지하 1층에 무료 주차가 가능합니다. 주차 등록은 안내데스크에서 해주세요. |
   | 노트북을 꼭 가져와야 하나요? | 네, 실습 수업이므로 개인 노트북이 필수입니다. 대여가 필요하시면 사전에 연락주세요. |
   | 수업을 못 갔을 경우 보강이 되나요? | 녹화 영상을 제공합니다. 수업 다음 날 이메일로 링크를 보내드립니다. |
   | 수료증이 발급되나요? | 출석 80% 이상 시 수료증이 이메일로 자동 발급됩니다. |
   | 환불은 어떻게 하나요? | 수업 시작 7일 전까지 100% 환불, 3일 전까지 50% 환불 가능합니다. |
   | Wi-Fi가 제공되나요? | 네, 강의실에서 무료 Wi-Fi를 사용하실 수 있습니다. 접속 정보는 수업 당일 안내됩니다. |

4. 하단의 **+** 버튼 클릭하여 새 시트 탭 추가 → 이름을 **"QA로그"** 로 변경

5. QA로그 시트의 1행에 헤더 입력:

   | A | B | C | D |
   |---|---|---|---|
   | 질문 | AI답변 | 응답시간 | 성공여부 |

> ✅ **확인**: Sheet1에 8개 FAQ, QA로그 시트에 헤더 4개가 입력되면 준비 완료!

---

### Step 2: n8n 워크플로우 생성 & Webhook 추가

1. n8n 좌측 사이드바 **"Workflows"** → **"Add workflow"** 클릭

2. 워크플로우 이름: **"실습11_QA자동응답봇"**

3. 캔버스 **[+]** 클릭 → **"Webhook"** 검색 → 선택

4. Webhook 노드 설정:
   - **HTTP Method**: `POST`
   - **Path**: `qa-bot`
   - **Response Mode**: `Using 'Respond to Webhook' Node`
     > n8n 2.x에서는 이 옵션으로 마지막에 Respond to Webhook 노드로 직접 응답을 제어합니다

5. 설정을 확인하고 **"Back to canvas"** 클릭

---

### Step 3: Google Sheets에서 FAQ 데이터 읽기

1. Webhook 노드 우측 **[+]** 클릭 → **"Google Sheets"** 검색 → 선택

2. Google Sheets 노드 설정:
   - **Credential**: "My Google" (2교시에서 등록한 것)
   - **Resource**: Sheet
   - **Operation**: **Read Rows**
   - **Document**: 드롭다운에서 **"FAQ_데이터"** 선택
   - **Sheet**: **"Sheet1"** 선택

3. **"Back to canvas"** 클릭

---

### Step 4: Code 노드 — FAQ를 프롬프트용 텍스트로 결합

1. Google Sheets 노드 우측 **[+]** → **"Code"** 검색 → 선택

2. Code 노드 설정:
   - **Language**: JavaScript
   - **Mode**: Run Once for All Items

3. 코드 에디터에 아래 코드를 **그대로** 붙여넣기:

   ```javascript
   // ① FAQ 데이터를 하나의 텍스트로 결합
   const faqItems = $input.all();
   
   const faqText = faqItems.map(item => 
     `Q: ${item.json.질문}\nA: ${item.json.답변}`
   ).join('\n\n');
   
   // ② Webhook으로 받은 사용자 질문 가져오기
   const webhookData = $('Webhook').first().json;
   const userQuestion = webhookData.body.question;
   
   // ③ 두 데이터를 합쳐서 다음 노드로 전달
   return [{
     json: {
       faqContext: faqText,
       userQuestion: userQuestion
     }
   }];
   ```

   > ⚠️ **코드 설명**:
   > - `$input.all()` : 바로 앞 노드(Google Sheets)에서 들어온 모든 데이터
   > - `$('Webhook').first().json` : "Webhook"이라는 이름의 노드에서 데이터 참조
   > - 노드 이름이 다르면(예: "Webhook1") 해당 이름으로 변경해야 합니다!

4. **"Back to canvas"** 클릭

---

### Step 5: OpenAI 노드 — AI 답변 생성

1. Code 노드 우측 **[+]** → **"OpenAI"** 검색 → 선택

2. OpenAI 노드 설정:
   - **Credential**: "My OpenAI"
   - **Resource**: Message
   - **Operation**: Send
   - **Model**: `gpt-4o-mini`

3. **Messages** 섹션에서 **"Add Message"** 클릭 (System 메시지 먼저):
   - **Role**: `System`
   - **Content** — 아래를 그대로 복사·붙여넣기:
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
   - `{{ $json.faqContext }}` 부분은 **Expression** 모드로 전환해야 합니다
     > 입력 필드 우측의 토글을 클릭하여 Expression 모드로 전환

4. **"Add Message"** 한 번 더 클릭 (User 메시지):
   - **Role**: `User`
   - **Content**: Expression 모드로 전환 후 입력
     ```
     {{ $json.userQuestion }}
     ```

5. **"Back to canvas"** 클릭

---

### Step 6: Edit Fields — 응답 데이터 정리

1. OpenAI 노드 우측 **[+]** → **"Edit Fields"** 검색 → **"Edit Fields (Set)"** 선택

2. **"Add Field"** 를 4번 클릭하여 아래 필드들을 추가:

   | # | Name | Type | Value (Expression 모드) |
   |---|------|------|------------------------|
   | 1 | `질문` | String | `{{ $('Code').first().json.userQuestion }}` |
   | 2 | `AI답변` | String | `{{ $json.message.content }}` |
   | 3 | `응답시간` | String | `{{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}` |
   | 4 | `성공여부` | String | `성공` (Fixed 모드 — 고정값) |

   > ⚠️ 각 Value 입력 시 **Expression 모드**로 전환하는 것을 잊지 마세요! (4번 "성공여부"는 Fixed 모드로 직접 입력)

3. **"Include Other Input Fields"**: **Off**

4. **"Back to canvas"** 클릭

---

### Step 7: Google Sheets — Q&A 로그 저장

1. Edit Fields 노드 우측 **[+]** → **"Google Sheets"** 선택

2. 설정:
   - **Operation**: **Append Row**
   - **Document**: "FAQ_데이터"
   - **Sheet**: **"QA로그"** 선택
   - 필드 매핑: 질문, AI답변, 응답시간, 성공여부 → 시트 헤더와 자동 매핑

3. **"Back to canvas"** 클릭

---

### Step 8: Respond to Webhook — 답변 반환

1. Google Sheets(로그) 노드 우측 **[+]** → **"Respond to Webhook"** 검색 → 선택

2. 설정:
   - **Response Code**: `200`
   - **Response Body**: **JSON**
   - **Response Data** (JSON 입력 — Expression 모드):
     ```json
     {
       "answer": "{{ $json.AI답변 }}",
       "source": "AI Assistant",
       "timestamp": "{{ $json.응답시간 }}"
     }
     ```

3. **"Back to canvas"** 클릭

---

### Step 9: 전체 연결 확인 & 테스트

최종 워크플로우가 아래와 같이 연결되어 있는지 확인합니다:

```
Webhook → Google Sheets(FAQ) → Code → OpenAI → Edit Fields → Google Sheets(로그) → Respond to Webhook
```

**테스트 방법:**

1. Webhook 노드를 클릭 → **"Test URL"** 을 복사합니다

2. 상단 **"Listen for test event"** 버튼을 클릭합니다 (대기 상태)

3. **새 브라우저 탭**에서 n8n으로 돌아가 → 새 워크플로우 생성: **"테스트_HTTP요청"**

4. Manual Trigger → **HTTP Request** 노드 추가:
   - **Method**: `POST`
   - **URL**: 위에서 복사한 Test URL
   - **Body Content Type**: `JSON`
   - **JSON Body**:
     ```json
     {
       "question": "수업 시간이 어떻게 되나요?"
     }
     ```

5. 테스트 워크플로우에서 **"Test workflow"** 클릭

6. 원래 워크플로우 탭으로 돌아가 결과 확인:
   - 각 노드에 ✓ 표시
   - Respond to Webhook 노드 Output:
     ```json
     {
       "answer": "안녕하세요! 오전반은 09:00~12:00, 오후반은 14:00~17:00에 수업이 진행됩니다. 원하시는 시간대를 선택하여 신청해주세요!",
       "source": "AI Assistant",
       "timestamp": "2026-03-23 10:30:00"
     }
     ```

7. **FAQ에 없는 질문도 테스트** — JSON Body를 변경:
   ```json
   {
     "question": "기숙사가 있나요?"
   }
   ```
   → "담당자에게 연결해드리겠습니다" 응답이 나오는지 확인

8. Google Sheets **"QA로그"** 시트를 열어 로그가 기록되었는지 확인

> ✅ **산출물**: AI 기반 Q&A 자동 응답 봇 워크플로우 완성!

---

---

## 실습 12: 교육 피드백 AI 요약 리포트

### 목표
축적된 피드백 데이터를 AI가 분석·요약하여 관리자에게 리포트를 자동 발송합니다.

### 완성 워크플로우 구조
```
Manual Trigger
    → Google Sheets (피드백 전체 읽기)
    → Code (데이터 결합 + 통계)
    → OpenAI 또는 Anthropic (분석 요청)
    → Edit Fields (리포트 정리)
    → Google Sheets (리포트 저장)
    → Gmail (관리자에게 발송)
```

---

### Step 1: 워크플로우 생성 & 데이터 읽기

1. 새 워크플로우: **"실습12_피드백AI요약"**

2. **Manual Trigger** 추가

3. Manual Trigger 우측 **[+]** → **Google Sheets** (Read Rows):
   - **Document**: "교육피드백_DB" (4교시에서 생성한 시트)
   - **Sheet**: "피드백"

> 💡 4교시 시트가 없다면 새로 만드세요. 헤더: 이름, 과정명, 만족도, 피드백, 수업날짜, 응답일시

---

### Step 2: Code 노드 — 피드백 결합 & 통계 계산

1. Google Sheets 우측 **[+]** → **Code** 노드 추가

2. Language: JavaScript, Mode: Run Once for All Items

3. 코드를 그대로 붙여넣기:

   ```javascript
   const items = $input.all();
   
   // ① 피드백 텍스트를 하나로 결합
   const feedbackList = items.map((item, index) => 
     `[수강생 ${index + 1}] 만족도: ${item.json.만족도}/5점\n내용: ${item.json.피드백}`
   ).join('\n\n---\n\n');
   
   // ② 기본 통계 계산
   const scores = items.map(i => Number(i.json.만족도));
   const sum = scores.reduce((a, b) => a + b, 0);
   const avg = (sum / scores.length).toFixed(1);
   const maxScore = Math.max(...scores);
   const minScore = Math.min(...scores);
   const courseName = items[0].json.과정명;
   
   // ③ 결과 반환
   return [{
     json: {
       과정명: courseName,
       응답수: items.length,
       평균만족도: avg,
       최고점: maxScore,
       최저점: minScore,
       전체피드백: feedbackList
     }
   }];
   ```

4. **"Back to canvas"** 클릭

---

### Step 3: AI 분석 요청 (OpenAI 또는 Anthropic 선택)

#### 방법 A: OpenAI 노드 사용

1. Code 노드 우측 **[+]** → **OpenAI** 선택

2. 설정:
   - **Model**: `gpt-4o-mini`
   - **Message 1 (System)**:
     ```
     당신은 교육 품질 분석 전문가입니다. 수강생 피드백을 분석하여 구조화된 리포트를 작성해주세요. 한국어로 답변하세요.
     ```
   - **Message 2 (User)** — Expression 모드로 전환 후:
     ```
     아래는 "{{ $json.과정명 }}" 과정의 수강생 피드백입니다.
     총 {{ $json.응답수 }}명 응답, 평균 만족도 {{ $json.평균만족도 }}/5.0 (최고 {{ $json.최고점 }}점, 최저 {{ $json.최저점 }}점)
     
     === 수강생 피드백 ===
     {{ $json.전체피드백 }}
     === 끝 ===
     
     위 피드백을 분석하여 아래 5가지를 정리해주세요:
     1. 전체 만족도 요약 (2~3문장)
     2. 긍정적 피드백 Top 3 (수강생들이 좋아한 점)
     3. 개선 요청 사항 Top 3 (수강생들이 불만족한 점)
     4. 다음 교육을 위한 구체적 제언 3가지
     5. 한 줄 총평
     ```

#### 방법 B: Anthropic 노드 사용 (Claude)

1. Code 노드 우측 **[+]** → **Anthropic** 선택

2. 설정:
   - **Credential**: "My Anthropic"
   - **Resource**: Message
   - **Operation**: Message a Model
   - **Model**: `claude-sonnet-4-20250514`
   - **Add Option** → **System Prompt**:
     ```
     당신은 교육 품질 분석 전문가입니다. 수강생 피드백을 분석하여 구조화된 리포트를 작성해주세요. 한국어로 답변하세요.
     ```
   - **Message (User)**: 위 방법 A와 동일한 프롬프트 사용

> 💡 **어떤 모델을 선택할까?**
> - 빠른 응답 & 저렴한 비용 → OpenAI `gpt-4o-mini`
> - 더 정교한 분석 & 긴 피드백 처리 → Anthropic `claude-sonnet-4`

---

### Step 4: Edit Fields — 리포트 데이터 정리

1. AI 노드 우측 **[+]** → **Edit Fields (Set)** 추가

2. 필드 추가 (5개):

   | # | Name | Value (Expression) |
   |---|------|--------------------|
   | 1 | `과정명` | `{{ $('Code').first().json.과정명 }}` |
   | 2 | `평균만족도` | `{{ $('Code').first().json.평균만족도 }}` |
   | 3 | `응답수` | `{{ $('Code').first().json.응답수 }}` |
   | 4 | `집계일시` | `{{ $now.toFormat('yyyy-MM-dd HH:mm') }}` |
   | 5 | `AI요약` | OpenAI 사용 시: `{{ $json.message.content }}`  /  Anthropic 사용 시: `{{ $json.content[0].text }}` |

   > ⚠️ AI 노드에 따라 5번 `AI요약`의 Expression이 다릅니다!

---

### Step 5: 리포트 저장 & 이메일 발송

1. Edit Fields 우측 **[+]** → **Google Sheets** (Append Row):
   - Document: "교육피드백_DB"
   - Sheet: **"리포트"**

2. Google Sheets 우측 **[+]** → **Gmail** 노드:
   - **Credential**: "My Gmail"
   - **To**: 본인 이메일
   - **Subject** (Expression 모드):
     ```
     [{{ $json.과정명 }}] 교육 피드백 AI 분석 리포트 — {{ $json.집계일시 }}
     ```
   - **Email Type**: HTML
   - **Message** (Expression 모드):
     ```html
     <h1>교육 피드백 분석 리포트</h1>
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
     <div style="background:#f5f5f5;padding:16px;border-radius:8px;white-space:pre-wrap;">{{ $json.AI요약 }}</div>
     <hr>
     <p><small>이 리포트는 n8n + AI에 의해 자동 생성되었습니다.</small></p>
     ```

---

### Step 6: 테스트

1. **"Test workflow"** (▶) 클릭

2. 확인:
   - AI 노드 Output → 분석 리포트 내용 정상 생성
   - Google Sheets "리포트" 시트 → 새 행 추가됨
   - Gmail → 리포트 이메일 수신됨

> ✅ **산출물**: 피드백 AI 분석 → 요약 리포트 → Sheets 저장 → 이메일 자동 발송 완성!

---

---

## 실습 13: 교육 콘텐츠 초안 자동 생성 (Gemini 활용)

### 목표
주제를 입력하면 Gemini가 교육 슬라이드 구성안을 자동 생성합니다.

### 완성 워크플로우 구조
```
Manual Trigger → Edit Fields (주제 입력) → Google Gemini (구성안 생성) → Google Sheets (저장)
```

---

### Step 1: 워크플로우 생성

1. 새 워크플로우: **"실습13_콘텐츠초안생성"**

2. **Manual Trigger** 추가

3. Manual Trigger 우측 **[+]** → **Edit Fields (Set)** 추가

4. 필드 추가:

   | Name | Type | Value (Fixed) |
   |------|------|---------------|
   | `주제` | String | `Python 데이터 분석 입문` |
   | `대상` | String | `IT 비전공 직장인` |
   | `시간` | String | `2시간` |

---

### Step 2: Google Gemini 노드 연결

1. Edit Fields 우측 **[+]** → **"Google Gemini"** 검색 → 선택

2. 설정:
   - **Credential**: "My Gemini"
   - **Resource**: Message
   - **Operation**: Message a Model
   - **Model**: `gemini-2.0-flash`

3. **Messages** → **"Add Message"** 클릭:
   - **Role**: `User`
   - **Content** (Expression 모드):
     ```
     아래 조건에 맞는 교육 슬라이드 구성안을 만들어주세요.

     조건:
     - 주제: {{ $json.주제 }}
     - 대상: {{ $json.대상 }}
     - 교육 시간: {{ $json.시간 }}
     - 슬라이드 수: 15~20개

     각 슬라이드별로 아래 형식으로 작성해주세요:
     ---
     슬라이드 N: [제목]
     핵심 내용:
     1. ...
     2. ...
     3. ...
     실습 활동: ...
     ---

     마지막에 전체 시간 배분표도 포함해주세요.
     한국어로 작성해주세요.
     ```

---

### Step 3: 결과 저장

1. Google Gemini 우측 **[+]** → **Edit Fields** 추가:

   | Name | Value (Expression) |
   |------|--------------------|
   | `주제` | `{{ $('Edit Fields').first().json.주제 }}` |
   | `구성안` | `{{ $json.text }}` |
   | `생성일시` | `{{ $now.toFormat('yyyy-MM-dd HH:mm') }}` |
   | `모델` | `gemini-2.0-flash` |

2. Edit Fields 우측 **[+]** → **Google Sheets** (Append Row):
   - 새 스프레드시트 **"콘텐츠초안_로그"** 생성 (헤더: 주제, 구성안, 생성일시, 모델)

---

### Step 4: 테스트

1. **"Test workflow"** 실행

2. Google Gemini 노드 Output에서 생성된 슬라이드 구성안 확인

3. Google Sheets에 저장 확인

> ✅ **산출물**: 교육 콘텐츠 초안 자동 생성 워크플로우

> 💡 **응용**: Edit Fields의 "주제"를 바꿔가며 다양한 과정의 구성안을 빠르게 생성해보세요!
> - `n8n 워크플로우 자동화 기초`
> - `ChatGPT 비즈니스 활용법`
> - `데이터 시각화 with Google Sheets`

---

---

## 실습 14: 수강생 이메일 AI 자동 작성기

### 목표
상황과 수강생 정보를 입력하면 AI가 맞춤형 이메일 본문을 자동 생성합니다.

### 완성 워크플로우 구조
```
Manual Trigger → Edit Fields (상황 입력) → AI (이메일 생성) → Gmail (발송)
```

---

### Step 1: 워크플로우 생성

1. 새 워크플로우: **"실습14_이메일AI작성기"**

2. **Manual Trigger** → **Edit Fields (Set)** 추가

3. 필드 추가:

   | Name | Value (Fixed) |
   |------|---------------|
   | `수강생이름` | `홍길동` |
   | `수강생이메일` | `(본인 이메일)` |
   | `과정명` | `AI입문` |
   | `상황` | `수강 신청 완료 환영 이메일` |

   > 💡 "상황"을 바꾸면 다양한 이메일을 생성할 수 있습니다:
   > - `수강 신청 완료 환영 이메일`
   > - `수강료 납부 독촉 이메일`
   > - `수업 결석 안내 이메일`
   > - `수료 축하 이메일`
   > - `다음 과정 추천 이메일`

---

### Step 2: AI 이메일 생성

1. Edit Fields 우측 **[+]** → **OpenAI** (또는 Anthropic, Gemini) 추가

2. 설정:
   - **Model**: `gpt-4o-mini`
   - **Message 1 (System)**:
     ```
     당신은 교육 기관의 이메일 담당자입니다.
     주어진 상황에 맞는 전문적이고 따뜻한 이메일을 작성해주세요.
     
     규칙:
     1. 수강생 이름을 포함하여 개인화된 인사를 넣으세요.
     2. HTML 형식으로 작성하세요 (<h2>, <p>, <ul> 태그 사용).
     3. 이메일 하단에 교육 센터 연락처를 포함하세요 (전화: 02-1234-5678).
     4. 전체 길이는 200자 내외로 간결하게 작성하세요.
     5. 한국어로 작성하세요.
     ```
   - **Message 2 (User)** (Expression 모드):
     ```
     수강생 이름: {{ $json.수강생이름 }}
     과정명: {{ $json.과정명 }}
     상황: {{ $json.상황 }}
     
     위 정보에 맞는 이메일을 HTML 형식으로 작성해주세요.
     ```

---

### Step 3: Gmail 발송

1. OpenAI 우측 **[+]** → **Gmail** 추가

2. 설정:
   - **To** (Expression): `{{ $('Edit Fields').first().json.수강생이메일 }}`
   - **Subject** (Expression): `[{{ $('Edit Fields').first().json.과정명 }}] {{ $('Edit Fields').first().json.상황 }}`
   - **Email Type**: HTML
   - **Message** (Expression): `{{ $json.message.content }}`
     > AI가 생성한 HTML이 그대로 이메일 본문으로 들어갑니다

---

### Step 4: 테스트

1. **"Test workflow"** 실행

2. Gmail 수신함에서 AI가 작성한 이메일 확인

3. Edit Fields의 **"상황"** 값을 바꿔가며 다양한 이메일 생성 테스트:
   - `수강료 납부 독촉 이메일` → 납부 안내가 포함된 이메일 자동 생성
   - `수료 축하 이메일` → 축하 메시지 + 수료증 안내 이메일 생성

> ✅ **산출물**: AI 맞춤형 이메일 자동 작성 & 발송 워크플로우

---

---

## AI 활용 워크플로우 설계 팁 모음

### 프롬프트 설계 원칙

| 원칙 | 설명 | 예시 |
|------|------|------|
| **역할 지정** | System Prompt에서 AI의 역할을 명확히 | "교육 품질 분석 전문가" |
| **규칙 설정** | 답변 불가 시 대응 규칙 포함 | "FAQ에 없으면 담당자 연결 안내" |
| **형식 지정** | 출력 형식을 구체적으로 요청 | "HTML로 작성", "JSON으로 응답" |
| **길이 제한** | 답변 길이를 지정 | "2~3문장으로 간결하게" |
| **언어 지정** | 응답 언어를 명시 | "한국어로 답변하세요" |

### 모델별 프롬프트 팁

| AI 모델 | 팁 |
|---------|-----|
| **OpenAI** | JSON 형식 요청 시 가장 안정적. "다른 텍스트 없이 JSON만 응답" 명시 |
| **Claude** | 구조화된 긴 지시에 강함. XML 태그로 섹션 구분하면 정확도 향상 |
| **Gemini** | 멀티모달 입력 가능. 이미지+텍스트 조합 프롬프트 활용 |

### n8n 2.12.3에서 AI 노드 사용 시 주의사항

1. **Expression 모드 전환**: AI 프롬프트에 `{{ }}` 표현식을 넣으려면 반드시 Expression 모드로 전환
2. **응답 경로 차이**: OpenAI(`$json.message.content`), Anthropic(`$json.content[0].text`), Gemini(`$json.text`)
3. **Code 노드 격리**: n8n 2.x에서 Code 노드는 Task Runner에서 격리 실행 — `process.env` 사용 불가
4. **Publish vs Save**: Schedule Trigger/Webhook을 실제 운영하려면 반드시 **Publish** 해야 함
5. **날짜 함수**: `.format()` 아닌 `.toFormat()` 사용 (Luxon 라이브러리)

---

## 6교시 산출물 체크리스트

- [ ] [실습 11] AI Q&A 자동 응답 봇 — FAQ 기반 답변 생성 & 로그 저장
- [ ] [실습 12] 피드백 AI 분석 요약 리포트 — Sheets 저장 & 이메일 발송
- [ ] [실습 13] 교육 콘텐츠 초안 자동 생성 — Gemini 활용
- [ ] [실습 14] 수강생 이메일 AI 자동 작성기 — 상황별 이메일 생성
- [ ] Code 노드에서 다른 노드 데이터 참조 방법 이해 (`$('노드이름')`)
- [ ] 3대 AI 모델 프롬프트 설계 차이 이해
- [ ] n8n 2.x 주의사항 (Expression 모드, Publish, Task Runner) 숙지

---

> **다음 교시 예고**: 7교시에서는 2일간 배운 모든 기술을 종합하여 팀별 프로젝트를 설계·구현합니다!
