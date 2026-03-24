# 📘 5교시 실습 가이드 — AI API 연동 & 프롬프트 엔지니어링

> **Day 2 | 5교시 (1시간)**
> 학습 목표: ChatGPT / Claude / Gemini API를 n8n에 연동하고 프롬프트를 설계한다
>
> **n8n 버전: 2.12.3 기준**

---

## n8n v2.x 참고 사항

> n8n 2.0부터 워크플로우 관리 방식이 변경되었습니다.
> - **Save** = 초안(Draft) 저장 (실행되지 않음)
> - **Publish** = 운영 배포 (트리거 활성화, 실제 실행)
> - 워크플로우를 실제로 작동시키려면 반드시 **Publish** 해야 합니다.
> - 편집 중에는 "Test workflow" 버튼으로 수동 테스트가 가능합니다.

---

## 실습 전 준비사항

- [ ] Day 1 실습 환경 정상 동작 확인 (n8n 2.12.3)
- [ ] OpenAI API 키 (강사 제공 또는 개인 발급)
- [ ] Anthropic API 키 (강사 제공 또는 개인 발급)
- [ ] Google Gemini API 키 (Google AI Studio에서 무료 발급)
- [ ] Day 1 과제 준비 완료

---

## 실습 9-A: n8n에서 ChatGPT API 호출하기 (OpenAI 노드)

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

1. n8n 좌측 사이드바에서 **"Credentials"** 클릭

2. 우측 상단 **"+ Add Credential"** 클릭

3. 검색창에 **"OpenAI"** 입력 → **"OpenAI API"** 선택

4. **API Key** 필드에 복사한 키를 붙여넣기:
   ```
   sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

5. Credential 이름: **"My OpenAI"**

6. **"Save"** 클릭

> ✅ **확인**: "Connection tested successfully" 메시지 확인!

---

### Step 3: 첫 번째 ChatGPT 호출

1. 좌측 사이드바 **"Workflows"** → **"Add workflow"** 클릭

2. 워크플로우 이름을 클릭하여 **"실습9A_ChatGPT호출"** 로 변경

3. 캔버스의 **[+]** 버튼 클릭 → **"Manual Trigger"** 검색 → 추가

4. Manual Trigger 우측 **[+]** → **"OpenAI"** 검색 → 선택

5. OpenAI 노드 설정:
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

6. 상단 **"Test workflow"** (▶) 클릭

7. OpenAI 노드를 클릭 → **"Output"** 탭에서 결과 확인:

   ```json
   {
     "message": {
       "role": "assistant",
       "content": "1. 수강 신청, 알림 발송 등 반복 업무를 자동화하여 운영 효율을 높입니다.\n2. ..."
     }
   }
   ```

8. AI 응답 참조 표현식: `{{ $json.message.content }}`

> ✅ **확인**: AI가 의미 있는 한국어 답변을 생성하면 성공!

---

### Step 4: System Prompt로 역할 지정하기

1. OpenAI 노드를 더블클릭하여 설정을 엽니다

2. Messages 섹션에서 **기존 User 메시지 위에** 새 메시지 추가:
   - **"Add Message"** 클릭
   - **Role**: `System`
   - **Content**:
     ```
     당신은 교육서비스업 전문 컨설턴트입니다.
     한국어로 답변하세요.
     답변은 구체적이고 실무적이어야 합니다.
     ```

3. 메시지 순서 확인 (System이 위, User가 아래):
   ```
   Message 1: System → "당신은 교육서비스업 전문 컨설턴트입니다..."
   Message 2: User   → "교육서비스업에서 n8n 자동화가 유용한 이유를..."
   ```

4. 다시 **"Test workflow"** → 더 전문적인 답변이 생성되는지 확인

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

---

## 실습 9-B: n8n에서 Claude API 호출하기 (Anthropic 노드)

### 목표
n8n 내장 **Anthropic 노드**를 사용하여 Claude API를 호출합니다.

> n8n 2.x에는 Anthropic 앱 노드가 기본 내장되어 있어 별도 설치 없이 사용 가능합니다.

---

### Step 1: Anthropic API 키 발급

1. 브라우저에서 접속:
   ```
   https://console.anthropic.com
   ```

2. Google 또는 이메일로 로그인 (계정이 없으면 가입)

3. 좌측 메뉴 **"Settings"** → **"API Keys"** 클릭

4. **"+ Create Key"** 버튼 클릭

5. 키 이름 입력: `n8n-education`

6. **"Create Key"** 클릭

7. 생성된 키를 **즉시 복사**합니다:
   ```
   sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
   > ⚠️ **중요**: 이 화면에서만 키를 볼 수 있습니다! 반드시 저장하세요.

> 💡 신규 계정에는 **$5 무료 크레딧**이 제공됩니다. 교육 실습에 충분한 양입니다.

---

### Step 2: n8n에 Anthropic Credential 등록

1. 좌측 사이드바 **"Credentials"** → **"+ Add Credential"**

2. 검색창에 **"Anthropic"** 입력 → **"Anthropic API"** 선택

3. **API Key** 필드에 복사한 키를 붙여넣기:
   ```
   sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

4. Credential 이름: **"My Anthropic"**

5. **"Save"** 클릭

> ✅ **확인**: "Connection tested successfully" 메시지 확인!
>
> ⚠️ **"resource not found" 에러가 발생하는 경우:**
> n8n 구버전에서 Credential 테스트 시 사용하는 모델이 계정에 없을 수 있습니다.
> n8n 2.12.3에서는 이 이슈가 해결되어 있지만, 혹시 발생하면 그래도 **Save를 진행**하세요 — 실제 워크플로우에서는 정상 동작합니다.

---

### Step 3: Anthropic 노드로 Claude 호출

1. 새 워크플로우 생성: **"실습9B_Claude호출"**

2. **Manual Trigger** 추가

3. Manual Trigger 우측 **[+]** → **"Anthropic"** 검색 → 선택

   > n8n 2.12.3에는 Anthropic 관련 노드가 여러 개 있습니다:
   >
   > | 노드 이름 | 유형 | 용도 |
   > |-----------|------|------|
   > | **Anthropic** | 앱 노드 | 직접 메시지 전송, 문서/이미지 분석, 프롬프트 생성 |
   > | **Anthropic Chat Model** | 서브 노드 | AI Agent 또는 LLM Chain에 연결하여 사용 |
   >
   > 여기서는 **Anthropic** 앱 노드를 선택합니다.

4. Anthropic 노드 설정:
   - **Credential**: "My Anthropic"
   - **Resource**: `Message`
   - **Operation**: `Message a Model`
   - **Model**: 드롭다운에서 `claude-sonnet-4-20250514` 선택
     > Sonnet 4은 속도·비용·성능의 균형이 가장 좋은 모델입니다
     > 빠른 응답이 필요하면 `claude-haiku-4-5-20251001` 선택
   - **Messages** 섹션:
     - **"Add Message"** 클릭
     - **Role**: `User`
     - **Content**:
       ```
       교육서비스업에서 업무 자동화의 장점을 3가지 알려주세요.
       한국어로 답변하세요.
       ```

5. **"Test workflow"** (▶) 클릭

6. Anthropic 노드의 **Output** 확인:

   ```json
   {
     "id": "msg_xxxxxxxxxxxx",
     "type": "message",
     "role": "assistant",
     "content": [
       {
         "type": "text",
         "text": "교육서비스업에서 업무 자동화의 장점은 다음과 같습니다:\n\n1. ..."
       }
     ],
     "model": "claude-sonnet-4-20250514",
     "usage": {
       "input_tokens": 35,
       "output_tokens": 250
     }
   }
   ```

7. Claude 응답 참조 표현식: `{{ $json.content[0].text }}`

> ✅ **확인**: Claude가 한국어 답변을 정상적으로 생성하면 성공!

---

### Step 4: Claude에서 System Prompt 사용하기

1. Anthropic 노드 설정 하단의 **"Add Option"** 클릭

2. 옵션 목록에서 **"System Prompt"** 선택

3. System Prompt 필드에 입력:
   ```
   당신은 교육서비스업 전문 컨설턴트입니다.
   항상 한국어로 답변하세요.
   답변은 구체적이고 실무적이어야 합니다.
   구조화된 형식으로 정리해주세요.
   ```

4. 다시 **"Test workflow"** → 더 체계적인 답변이 생성되는지 확인

> 💡 **Claude 특징**: Claude는 긴 문서 분석(최대 200K 토큰 입력)과 구조화된 지시사항 처리에 특히 강합니다. 또한 Anthropic 노드에서는 문서 분석(Analyze Document), 이미지 분석(Analyze Image) 등 멀티모달 기능도 지원합니다.

---

### Step 5: Anthropic 노드의 추가 기능 (v2.12.3)

n8n 2.12.3의 Anthropic 노드는 메시지 전송 외에도 다양한 기능을 지원합니다:

| Resource | Operation | 설명 |
|----------|-----------|------|
| Message | Message a Model | Claude에 메시지를 보내고 응답 받기 |
| Document | Analyze Document | PDF 등 문서를 업로드하여 질문하기 |
| Image | Analyze Image | 이미지를 업로드하여 분석·설명 받기 |
| File | Upload File | 파일을 API에 업로드 |
| File | List Files / Get Metadata / Delete | 파일 관리 |
| Prompt | Generate / Improve / Templatize | 프롬프트 자동 생성·개선 |

> 💡 예: "Analyze Document"로 수강 교재 PDF를 업로드하면 Claude가 내용을 요약하거나 퀴즈를 자동 생성할 수 있습니다.

---

---

## 실습 9-C: n8n에서 Gemini API 호출하기 (Google Gemini 노드)

### 목표
Google Gemini 노드를 사용하여 Gemini API를 호출합니다.

---

### Step 1: Google Gemini API 키 발급

1. 브라우저에서 접속:
   ```
   https://aistudio.google.com/apikey
   ```

2. Google 계정으로 로그인

3. **"Create API Key"** 버튼 클릭

4. 프로젝트 선택:
   - **"Create API key in new project"** 선택 (처음인 경우)
   - 또는 기존 Google Cloud 프로젝트 선택

5. 생성된 API 키를 복사합니다:
   ```
   AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
   ```

> 💡 **무료 사용**: Google AI Studio의 Gemini API는 **무료 티어**가 있어 교육 실습에 비용 부담 없이 사용 가능합니다!

---

### Step 2: n8n에 Google Gemini Credential 등록

1. 좌측 사이드바 **"Credentials"** → **"+ Add Credential"**

2. 검색창에 **"Gemini"** 입력 → **"Google Gemini(PaLM) API"** 선택

3. 설정:
   - **Host**: `https://generativelanguage.googleapis.com` (기본값 그대로 유지)
   - **API Key**: 위에서 복사한 키 붙여넣기
     ```
     AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
     ```

4. Credential 이름: **"My Gemini"**

5. **"Save"** 클릭

> ✅ **확인**: "Connection tested successfully" 메시지 확인!

---

### Step 3: Google Gemini 노드로 호출

1. 새 워크플로우 생성: **"실습9C_Gemini호출"**

2. **Manual Trigger** 추가

3. Manual Trigger 우측 **[+]** → **"Google Gemini"** 검색 → 선택

   > n8n 2.12.3에는 Gemini 관련 노드가 여러 개 있습니다:
   >
   > | 노드 이름 | 유형 | 용도 |
   > |-----------|------|------|
   > | **Google Gemini** | 앱 노드 | 직접 메시지 전송, 이미지/문서/오디오 분석, 이미지 생성 |
   > | **Google Gemini Chat Model** | 서브 노드 | AI Agent 또는 LLM Chain에 연결하여 사용 |
   >
   > 여기서는 **Google Gemini** 앱 노드를 선택합니다.

4. Google Gemini 노드 설정:
   - **Credential**: "My Gemini"
   - **Resource**: `Message`
   - **Operation**: `Message a Model`
   - **Model**: 드롭다운에서 `gemini-2.0-flash` 선택
     > Flash는 빠르고 비용 효율적인 모델입니다
   - **Messages** 섹션:
     - **"Add Message"** 클릭
     - **Role**: `User`
     - **Content**:
       ```
       교육서비스업에서 AI를 활용한 업무 자동화 방안을 3가지 제안해주세요.
       한국어로 답변하세요.
       ```

5. **"Test workflow"** (▶) 클릭

6. Google Gemini 노드의 **Output** 확인:

   ```json
   {
     "text": "교육서비스업에서 AI를 활용한 업무 자동화 방안을 제안드리겠습니다.\n\n1. ..."
   }
   ```

7. Gemini 응답 참조 표현식: `{{ $json.text }}`

> ✅ **확인**: Gemini가 한국어 답변을 정상적으로 생성하면 성공!

---

### Step 4: Gemini 노드의 추가 기능 (v2.12.3)

n8n 2.12.3의 Google Gemini 노드가 지원하는 전체 기능:

| Resource | Operation | 설명 |
|----------|-----------|------|
| Message | Message a Model | Gemini에 메시지를 보내고 응답 받기 |
| Image | Analyze Image | 이미지를 보내고 분석·설명 받기 |
| Image | Generate an Image | 텍스트 프롬프트로 이미지 생성 |
| Document | Analyze Document | 문서를 업로드하여 질문하기 |
| Audio | Analyze Audio | 오디오 파일을 분석·질문 |
| Audio | Transcribe a Recording | 음성을 텍스트로 변환 (STT) |
| File | Upload to File Search Store | RAG용 파일 업로드 |

**이미지 분석 빠른 실습:**

1. Google Gemini 노드에서 **Resource: Image** → **Operation: Analyze Image** 선택
2. Input Type: URL → 이미지 URL 입력
3. Prompt: `이 이미지에 있는 텍스트를 모두 추출해주세요.`
4. 테스트 실행 → OCR처럼 이미지 속 텍스트가 추출됩니다

**음성 인식 빠른 실습:**

1. **Resource: Audio** → **Operation: Transcribe a Recording** 선택
2. 오디오 파일(mp3, wav 등) 입력
3. 테스트 실행 → 음성이 텍스트로 변환됩니다

> 💡 **활용 시나리오**: 수업 녹음 파일 → Gemini 음성 인식 → 텍스트 변환 → ChatGPT 요약 → 수강생에게 이메일 발송

---

---

## 3대 AI 모델 비교 요약 (n8n 2.12.3 기준)

| 항목 | OpenAI (ChatGPT) | Anthropic (Claude) | Google (Gemini) |
|------|-----------------|-------------------|----------------|
| **n8n 노드 이름** | OpenAI | Anthropic | Google Gemini |
| **Credential 타입** | OpenAI API | Anthropic API | Google Gemini(PaLM) API |
| **메시지 Operation** | Message > Send | Message > Message a Model | Message > Message a Model |
| **추천 모델** | `gpt-4o-mini` | `claude-sonnet-4-20250514` | `gemini-2.0-flash` |
| **응답 참조 표현식** | `$json.message.content` | `$json.content[0].text` | `$json.text` |
| **System Prompt 위치** | Messages에 Role: System 추가 | Add Option → System Prompt | Messages에 Role: System 추가 |
| **API 키 발급** | platform.openai.com | console.anthropic.com | aistudio.google.com/apikey |
| **무료 티어** | 없음 (유료) | $5 크레딧 | 무료 티어 제공 |
| **멀티모달** | 이미지 분석 | 문서·이미지 분석 | 이미지·문서·오디오·이미지생성 |
| **강점** | 범용 성능, 넓은 생태계 | 긴 문서 분석, 정확한 지시 수행 | 멀티모달, 비용 효율, 음성 인식 |

> 💡 **실무 팁**: 용도에 따라 모델을 나눠 사용하면 비용 절감 + 성능 최적화가 가능합니다.
> - 간단한 분류/요약 → **Gemini Flash** (무료/저렴)
> - 일반적인 텍스트 생성 → **GPT-4o-mini** (가성비)
> - 복잡한 분석/긴 문서 → **Claude Sonnet** (정확성)

---

---

## 실습 10: AI 응답 → Sheets 저장 → 이메일 발송

### 목표
AI 응답을 가공하여 Google Sheets에 저장하고 이메일로 공유합니다.

---

### Step 1: 전체 워크플로우 구성

```
Manual Trigger → OpenAI (질문) → Edit Fields (응답 추출)
                                       → Google Sheets (저장)
                                       → Gmail (공유)
```

1. 새 워크플로우: **"실습10_AI응답파이프라인"**

2. **Manual Trigger** 추가

3. **OpenAI** 노드 추가:
   - System: `당신은 교육 분석 전문가입니다. 한국어로 답변하세요.`
   - User: `최근 교육 트렌드에서 AI 활용 방안을 5가지 제안해주세요.`
   - Model: `gpt-4o-mini`

4. **Edit Fields (Set)** 노드 추가:

   | Name | Value |
   |------|-------|
   | `질문` | `최근 교육 트렌드에서 AI 활용 방안` |
   | `응답` | `{{ $json.message.content }}` |
   | `모델` | `gpt-4o-mini` |
   | `일시` | `{{ $now.toFormat('yyyy-MM-dd HH:mm') }}` |

   > ⚠️ **n8n 날짜 함수 주의**: n8n은 Luxon 라이브러리를 사용합니다.
   > - ✅ 올바른 문법: `$now.toFormat('yyyy-MM-dd')`
   > - ❌ 잘못된 문법: `$now.format('yyyy-MM-dd')` ← 에러 발생!
   > - ✅ 내일 날짜: `$now.plus({ days: 1 }).toFormat('yyyy-MM-dd')`

5. **Google Sheets** (Append Row):
   - 새 스프레드시트 **"AI응답로그"** 생성 (헤더: 질문, 응답, 모델, 일시)
   - 4개 필드 매핑

6. **Gmail** 노드:
   - **To**: 본인 이메일
   - **Subject**: `AI 분석 결과 — {{ $json.일시 }}`
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

1. **"Test workflow"** (▶) 실행
2. Google Sheets "AI응답로그" 시트에 데이터 추가 확인
3. 이메일 수신 확인

> ✅ **산출물**: AI 응답 → Sheets 저장 → 이메일 발송 파이프라인

---

### 응용: 모델을 바꿔서 비교해보기

위 워크플로우에서 OpenAI 노드를 **Anthropic** 또는 **Google Gemini** 로 교체하면, 같은 질문에 대한 각 모델의 답변을 비교할 수 있습니다.

**교체 시 Edit Fields 수정 포인트:**

| AI 모델 | 노드 변경 | `응답` 필드 값 변경 | `모델` 필드 값 변경 |
|---------|----------|---------------------|---------------------|
| OpenAI | OpenAI 노드 | `{{ $json.message.content }}` | `gpt-4o-mini` |
| Claude | Anthropic 노드 | `{{ $json.content[0].text }}` | `claude-sonnet-4` |
| Gemini | Google Gemini 노드 | `{{ $json.text }}` | `gemini-2.0-flash` |

> 💡 3개 노드를 모두 연결하고 **Merge 노드**로 결과를 합치면, 동일 질문에 대한 3개 모델의 답변을 한 번에 비교할 수도 있습니다!

---

---

## AI API 비용 & 안전 가이드

### 비용 절감 팁

| 설정 | 권장값 | 효과 |
|------|--------|------|
| Model | `gpt-4o-mini` / `gemini-2.0-flash` | 가장 저렴한 옵션 |
| max_tokens | 500~1000 | 불필요하게 긴 응답 방지 |
| Temperature | 0.3~0.5 | 일관성 높은 답변 (창의성 필요 시 0.7~1.0) |

### 모델별 비용 비교 (대략적 기준, 2026년 3월)

| 모델 | 입력 (1M 토큰) | 출력 (1M 토큰) | 비고 |
|------|----------------|----------------|------|
| GPT-4o-mini | $0.15 | $0.60 | 가성비 우수 |
| GPT-4o | $2.50 | $10.00 | 고성능 |
| Claude Sonnet 4 | $3.00 | $15.00 | 긴 문서에 강점 |
| Claude Haiku 4.5 | $0.80 | $4.00 | 빠른 응답 |
| Gemini 2.0 Flash | 무료 티어 포함 | 무료 티어 포함 | 가장 저렴 |

> 💡 교육 실습 기준으로 3가지 모델을 모두 사용해도 **하루 $1 이하**로 충분합니다.

### API 키 보안 주의사항

1. API 키를 코드에 직접 넣지 마세요 → n8n Credential에만 저장
2. API 키를 다른 사람에게 공유하지 마세요
3. 각 서비스 대시보드에서 **Usage Limit** (사용 한도) 설정 권장
4. 사용하지 않는 키는 즉시 삭제

### n8n 2.x Code 노드 참고

> n8n 2.0부터 Code 노드(JavaScript/Python)는 **Task Runner**에서 격리 실행됩니다.
> - 환경 변수 직접 접근이 제한됩니다 (`process.env` 사용 불가)
> - 필요한 값은 n8n Credential 또는 Edit Fields 노드로 전달하세요
> - 이는 보안 강화를 위한 변경입니다

---

## 5교시 산출물 체크리스트

- [ ] OpenAI API Credential 등록 완료
- [ ] [실습 9-A] ChatGPT API 호출 성공 (System Prompt, JSON 형식)
- [ ] Anthropic API Credential 등록 완료
- [ ] [실습 9-B] Claude API — Anthropic 노드로 호출 성공
- [ ] Google Gemini API Credential 등록 완료
- [ ] [실습 9-C] Gemini API — Google Gemini 노드로 호출 성공
- [ ] 3대 AI 모델 응답 구조 차이 이해 (응답 참조 표현식)
- [ ] [실습 10] AI 응답 → Sheets → 이메일 파이프라인 완성
- [ ] 프롬프트 엔지니어링 기초 (역할 지정, 형식 지정) 이해

---

> **다음 교시 예고**: 6교시에서는 AI Q&A 자동 응답 봇과 피드백 AI 요약 리포트를 구축합니다!
