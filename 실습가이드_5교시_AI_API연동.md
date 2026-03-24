# 📘 5교시 실습 가이드 — AI API 연동 & 프롬프트 엔지니어링

> **Day 2 | 5교시 (1시간)**
> 학습 목표: ChatGPT/Claude API를 n8n에 연동하고 프롬프트를 설계한다

---

## 실습 전 준비사항

- [ ] Day 1 실습 환경 정상 동작 확인
- [ ] OpenAI API 키 (강사 제공 또는 개인 발급)
- [ ] Day 1 과제 준비 완료

---

## 실습 9: n8n에서 ChatGPT API 호출하기

### 목표
OpenAI 노드를 추가하고 첫 번째 AI 응답을 받아봅니다.

---

### Step 1: OpenAI API 키 발급

> 💡 강사가 교육용 API 키를 제공하는 경우 이 단계를 건너뛰세요.

1. 브라우저에서 접속:
   ```
   https://platform.openai.com
   ```

2. Google 또는 이메일로 로그인

3. 좌측 메뉴에서 **"API keys"** 클릭

4. **"+ Create new secret key"** 버튼 클릭

5. 키 설정:
   - **Name**: `n8n-education`
   - **Permissions**: All (기본값)

6. **"Create secret key"** 클릭

7. 생성된 키를 **즉시 복사**합니다:
   ```
   sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
   > ⚠️ **중요**: 이 화면을 벗어나면 키를 다시 볼 수 없습니다! 반드시 메모장에 저장하세요.

8. **"Done"** 클릭

---

### Step 2: n8n에 OpenAI Credential 등록

1. n8n 좌측 메뉴 **"Credentials"** → **"+ Add Credential"**

2. **"OpenAI"** 검색 → 선택

3. **API Key** 필드에 복사한 키를 붙여넣기:
   ```
   sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

4. Credential 이름: **"My OpenAI"**

5. **"Save"** 클릭

> ✅ **확인**: "Connection tested successfully" 메시지 확인!

---

### Step 3: 첫 번째 ChatGPT 호출

1. 새 워크플로우: **"실습9_ChatGPT첫호출"**

2. **Manual Trigger** 추가

3. Manual Trigger 우측 **[+]** → **"OpenAI"** 검색 → 선택

4. OpenAI 노드 설정:
   - **Credential**: "My OpenAI"
   - **Resource**: `Message`
   - **Operation**: `Send`
   - **Model**: `gpt-4o-mini`
     > 비용 절감을 위해 mini 모델 사용 (응답 품질도 충분합니다)
   - **Messages** 섹션:
     - **"Add Message"** 클릭
     - **Role**: `User`
     - **Content**:
       ```
       교육서비스업에서 n8n 자동화가 유용한 이유를 3가지 알려주세요.
       각 이유를 한 줄로 간결하게 설명해주세요.
       ```

5. **"Test workflow"** (▶) 클릭

6. OpenAI 노드의 **Output** 확인:

   ```json
   {
     "message": {
       "role": "assistant",
       "content": "1. 수강 신청, 알림 발송 등 반복 업무를 자동화하여 운영 효율을 높입니다.\n2. ..."
     }
   }
   ```

7. AI 응답 확인: `$json.message.content` 에 텍스트 응답이 담겨 있습니다

> ✅ **확인**: AI가 의미 있는 한국어 답변을 생성하면 성공!

---

### Step 4: System Prompt로 역할 지정하기

1. OpenAI 노드를 다시 열어 설정을 수정합니다

2. Messages 섹션에서 **기존 User 메시지 위에** 새 메시지 추가:
   - **"Add Message"** 클릭
   - **Role**: `System`
   - **Content**:
     ```
     당신은 교육서비스업 전문 컨설턴트입니다.
     한국어로 답변하세요.
     답변은 구체적이고 실무적이어야 합니다.
     ```

3. 메시지 순서 확인:
   ```
   Message 1: System → "당신은 교육서비스업 전문 컨설턴트입니다..."
   Message 2: User   → "교육서비스업에서 n8n 자동화가 유용한 이유를..."
   ```

4. 다시 테스트 → 더 전문적인 답변이 생성되는지 확인

---

### Step 5: 출력 형식 지정 (JSON 응답 받기)

1. User 메시지를 아래로 변경합니다:
   ```
   아래 형식의 JSON으로만 응답해주세요. 다른 텍스트는 포함하지 마세요.
   
   {
     "reasons": [
       {"title": "이유 제목", "description": "한 줄 설명"},
       {"title": "이유 제목", "description": "한 줄 설명"},
       {"title": "이유 제목", "description": "한 줄 설명"}
     ]
   }
   
   질문: 교육서비스업에서 n8n 자동화가 유용한 이유 3가지
   ```

2. 테스트 실행 → Output에서 JSON 형식 응답 확인

> 💡 **TIP**: AI에게 JSON 형식을 요청하면 후속 노드에서 데이터를 가공하기 훨씬 편합니다.

---

### Step 6: Claude API 연동 (HTTP Request)

1. 같은 워크플로우에 **HTTP Request** 노드 추가

2. 설정:
   - **Method**: `POST`
   - **URL**: `https://api.anthropic.com/v1/messages`
   - **Authentication**: `Generic Credential Type` → `Header Auth`
     - 또는 Header를 직접 지정 (아래 방법)

3. **Headers** 탭 — "Add Header":

   | Name | Value |
   |------|-------|
   | `x-api-key` | `sk-ant-xxxxxxxx` (Claude API 키) |
   | `anthropic-version` | `2023-06-01` |
   | `Content-Type` | `application/json` |

4. **Body** 탭:
   - **Body Content Type**: `JSON`
   - **JSON Body**:
     ```json
     {
       "model": "claude-sonnet-4-20250514",
       "max_tokens": 1024,
       "messages": [
         {
           "role": "user",
           "content": "교육서비스업에서 업무 자동화의 장점을 3가지 알려주세요. 한국어로 답변하세요."
         }
       ]
     }
     ```

5. **"Test step"** 클릭

6. Output에서 Claude 응답 확인:
   ```json
   {
     "content": [
       {
         "type": "text",
         "text": "교육서비스업에서 업무 자동화의 장점은..."
       }
     ]
   }
   ```

7. Claude 응답 참조: `$json.content[0].text`

> ✅ **확인**: ChatGPT와 Claude 둘 다 정상 응답을 받으면 성공!

---

## 실습 10: AI 응답 → Sheets 저장 → Slack 발송

### 목표
AI 응답을 가공하여 Google Sheets에 저장하고 Slack으로 공유합니다.

---

### Step 1: 전체 워크플로우 구성

```
Manual Trigger → OpenAI (질문) → Edit Fields (응답 추출)
                                       → Google Sheets (저장)
                                       → Slack 또는 Gmail (공유)
```

1. 새 워크플로우: **"실습10_AI응답파이프라인"**

2. **Manual Trigger** 추가

3. **OpenAI** 노드 추가:
   - System: `당신은 교육 분석 전문가입니다. 한국어로 답변하세요.`
   - User: `최근 교육 트렌드에서 AI 활용 방안을 5가지 제안해주세요.`
   - Model: `gpt-4o-mini`

4. **Edit Fields** 노드 추가:

   | Name | Value |
   |------|-------|
   | `질문` | `최근 교육 트렌드에서 AI 활용 방안` |
   | `응답` | `{{ $json.message.content }}` |
   | `모델` | `gpt-4o-mini` |
   | `일시` | `{{ $now.format('YYYY-MM-DD HH:mm') }}` |

5. **Google Sheets** (Append Row):
   - 새 스프레드시트 **"AI응답로그"** 생성
   - 헤더: 질문, 응답, 모델, 일시
   - 4개 필드 매핑

6. **Gmail** 노드 (또는 Slack):
   - **To**: 본인 이메일
   - **Subject**: `📌 AI 분석 결과 — {{ $json.일시 }}`
   - **Message**:
     ```html
     <h2>AI 분석 결과</h2>
     <p><strong>질문:</strong> {{ $json.질문 }}</p>
     <hr>
     <div>{{ $json.응답 }}</div>
     <hr>
     <p><small>모델: {{ $json.모델 }} | 생성일시: {{ $json.일시 }}</small></p>
     ```

---

### Step 2: 테스트

1. **"Test workflow"** 실행
2. Google Sheets "AI응답로그" 시트에 데이터 추가 확인
3. 이메일 수신 확인

> ✅ **산출물**: AI 응답 → Sheets 저장 → 이메일 발송 파이프라인

---

## AI API 비용 & 안전 가이드

### 비용 절감 팁

| 설정 | 권장값 | 효과 |
|------|--------|------|
| Model | `gpt-4o-mini` | GPT-4o 대비 약 1/15 비용 |
| max_tokens | 500~1000 | 불필요하게 긴 응답 방지 |
| Temperature | 0.3~0.5 | 일관성 높은 답변 (창의성 필요 시 0.7~1.0) |

### API 키 보안 주의사항

1. API 키를 코드에 직접 넣지 마세요 → n8n Credential에만 저장
2. API 키를 다른 사람에게 공유하지 마세요
3. OpenAI 대시보드에서 **Usage Limit** (사용 한도) 설정 권장
4. 사용하지 않는 키는 즉시 삭제

---

## 5교시 산출물 체크리스트

- [ ] OpenAI API Credential 등록 완료
- [ ] [실습 9] ChatGPT API 호출 성공 (System Prompt, JSON 형식)
- [ ] Claude API HTTP Request 호출 성공
- [ ] [실습 10] AI 응답 → Sheets → 이메일 파이프라인 완성
- [ ] 프롬프트 엔지니어링 기초 (역할 지정, 형식 지정) 이해

---

> **다음 교시 예고**: 6교시에서는 AI Q&A 자동 응답 봇과 피드백 AI 요약 리포트를 구축합니다!
