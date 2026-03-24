# 📘 7교시 실습 가이드 — 멀티미디어 AI 활용 실습

> **Day 2 | 7교시 (1시간)**
> 학습 목표: 이미지·문서·오디오 등 멀티미디어를 AI로 분석·생성하는 워크플로우를 구축한다
>
> **n8n 버전: 2.12.3 기준**

---

## 7교시 전체 구성

| 실습 | 시간 | 내용 | 사용 AI | 멀티미디어 |
|------|------|------|---------|-----------|
| 실습 15 | 15분 | 이미지 OCR → 수강 신청 자동 접수 | Gemini | 이미지 분석 |
| 실습 16 | 10분 | AI 이미지 생성 → 교육 홍보물 제작 | Gemini | 이미지 생성 |
| 실습 17 | 15분 | 교육 자료 PDF AI 분석·요약 | Anthropic / Gemini | 문서 분석 |
| 실습 18 | 10분 | 수업 녹음 자동 텍스트 변환 & 요약 | Gemini + OpenAI | 오디오 분석 |
| 실습 19 | 10분 | 멀티모달 종합 파이프라인 | 전체 조합 | 복합 |

---

---

## 실습 15: 이미지 OCR → 수강 신청 자동 접수

### 목표
수강 신청서 이미지를 Gemini로 분석하여 텍스트를 추출하고, Google Sheets에 자동 저장합니다.

### 시나리오
> 수강생이 손으로 작성한 신청서를 사진으로 찍어 보내면 → AI가 텍스트를 추출하고 → 자동으로 DB에 저장

### 완성 워크플로우 구조
```
Manual Trigger → Google Gemini (Analyze Image) → Code (JSON 파싱) → Google Sheets (저장) → Gmail (확인 메일)
```

---

### Step 1: 테스트용 이미지 URL 준비

> 실습에서는 인터넷에 공개된 이미지 URL을 사용합니다.
> 실제 운영 시에는 Webhook으로 이미지를 수신하거나 Google Drive에서 가져올 수 있습니다.

테스트용으로 아무 텍스트가 포함된 이미지 URL을 준비하세요.
또는 아래처럼 직접 Google Slides/Docs로 신청서 이미지를 만들어 공유 링크를 사용할 수 있습니다.

---

### Step 2: 워크플로우 생성

1. 새 워크플로우: **"실습15_이미지OCR신청접수"**

2. **Manual Trigger** 추가

3. Manual Trigger 우측 **[+]** → **Edit Fields (Set)** 추가

4. 필드 추가:

   | Name | Type | Value (Fixed) |
   |------|------|---------------|
   | `imageUrl` | String | `(테스트 이미지 URL 붙여넣기)` |

---

### Step 3: Google Gemini — 이미지 분석

1. Edit Fields 우측 **[+]** → **"Google Gemini"** 검색 → 선택

2. 설정:
   - **Credential**: "My Gemini"
   - **Resource**: `Image`
   - **Operation**: `Analyze Image`
   - **Input Type**: `URL`
   - **URL** (Expression 모드): `{{ $json.imageUrl }}`
   - **Prompt**:
     ```
     이 이미지에서 수강 신청 정보를 추출해주세요.
     
     반드시 아래 JSON 형식으로만 응답하세요. 다른 텍스트는 포함하지 마세요.
     이미지에서 해당 정보를 찾을 수 없으면 "미확인"으로 표시하세요.
     
     {
       "이름": "",
       "연락처": "",
       "이메일": "",
       "희망과정": "",
       "비고": ""
     }
     ```

3. **"Back to canvas"** 클릭

---

### Step 4: Code 노드 — JSON 파싱

1. Google Gemini 우측 **[+]** → **Code** 노드 추가

2. 코드:

   ```javascript
   const rawText = $json.text;
   
   // AI 응답에서 JSON 부분만 추출
   let jsonStr = rawText;
   
   // 코드 블록으로 감싸져 있을 경우 제거
   if (jsonStr.includes('```')) {
     jsonStr = jsonStr.replace(/```json\n?/g, '').replace(/```\n?/g, '');
   }
   
   try {
     const parsed = JSON.parse(jsonStr.trim());
     return [{
       json: {
         이름: parsed.이름 || '미확인',
         연락처: parsed.연락처 || '미확인',
         이메일: parsed.이메일 || '미확인',
         희망과정: parsed.희망과정 || '미확인',
         비고: parsed.비고 || '',
         접수일시: new Date().toISOString().slice(0, 19).replace('T', ' '),
         접수방법: '이미지OCR'
       }
     }];
   } catch (e) {
     // JSON 파싱 실패 시 원본 텍스트 반환
     return [{
       json: {
         이름: '파싱실패',
         연락처: '',
         이메일: '',
         희망과정: '',
         비고: rawText,
         접수일시: new Date().toISOString().slice(0, 19).replace('T', ' '),
         접수방법: '이미지OCR(파싱실패)'
       }
     }];
   }
   ```

---

### Step 5: Google Sheets 저장 & 확인 메일

1. Code 노드 우측 **[+]** → **Google Sheets** (Append Row):
   - 새 스프레드시트 **"이미지신청접수_DB"** 생성
   - 헤더: 이름, 연락처, 이메일, 희망과정, 비고, 접수일시, 접수방법

2. Google Sheets 우측 **[+]** → **Gmail**:
   - **To** (Expression): `{{ $json.이메일 }}`
   - **Subject**: `[수강 신청 접수 완료] {{ $json.희망과정 }}`
   - **Message**:
     ```html
     <h2>{{ $json.이름 }}님, 수강 신청이 접수되었습니다.</h2>
     <p>희망 과정: <strong>{{ $json.희망과정 }}</strong></p>
     <p>접수 일시: {{ $json.접수일시 }}</p>
     <p>접수 방법: 이미지 자동 인식 (AI OCR)</p>
     <hr>
     <p>정확한 정보 확인 후 최종 등록 안내를 드리겠습니다.</p>
     ```

---

### Step 6: 테스트

1. **"Test workflow"** 실행

2. 확인:
   - Gemini 노드 Output → 이미지에서 추출한 텍스트
   - Code 노드 Output → 정리된 JSON 데이터
   - Google Sheets → 새 행 추가
   - Gmail → 확인 이메일 발송

> ✅ **산출물**: 이미지 OCR → JSON 파싱 → Sheets 저장 → 이메일 발송 워크플로우

---

---

## 실습 16: AI 이미지 생성 → 교육 홍보물 제작

### 목표
텍스트 프롬프트로 교육 과정 홍보용 이미지를 AI로 생성합니다.

### 완성 워크플로우 구조
```
Manual Trigger → Edit Fields (과정 정보) → Google Gemini (Generate Image) → Gmail (홍보 이메일)
```

> ⚠️ **사전 조건**: Google Gemini 이미지 생성은 Google Cloud 결제 계정이 연결되어 있어야 합니다.
> 무료 티어로는 이미지 생성이 제한될 수 있습니다. 결제 계정이 없는 경우 강사 데모로 진행합니다.

---

### Step 1: 워크플로우 생성

1. 새 워크플로우: **"실습16_AI홍보이미지생성"**

2. **Manual Trigger** → **Edit Fields (Set)** 추가

3. 필드:

   | Name | Value (Fixed) |
   |------|---------------|
   | `과정명` | `AI 입문 - 비전공자를 위한 첫 걸음` |
   | `대상` | `직장인, 대학생` |
   | `키워드` | `인공지능, 자동화, 미래` |

---

### Step 2: Google Gemini — 이미지 생성

1. Edit Fields 우측 **[+]** → **Google Gemini** 선택

2. 설정:
   - **Resource**: `Image`
   - **Operation**: `Generate an Image`
   - **Model**: `imagen-3.0-generate-002` (또는 사용 가능한 모델 선택)
   - **Prompt** (Expression 모드):
     ```
     교육 과정 홍보 포스터를 만들어주세요.
     
     과정명: {{ $json.과정명 }}
     대상: {{ $json.대상 }}
     분위기: 밝고 현대적, 테크놀로지 느낌
     스타일: 미니멀 일러스트레이션, 파란색과 초록색 계열
     텍스트 포함하지 않기 (이미지만)
     ```
   - **Number of Images**: `1`

3. **"Back to canvas"** 클릭

---

### Step 3: 이미지 확인 & 활용

1. **"Test workflow"** 실행

2. Google Gemini 노드 Output에서 생성된 이미지 확인
   - 이미지는 **바이너리 데이터**로 Output에 포함됩니다
   - 노드 Output 탭에서 이미지 미리보기 가능

3. 활용 옵션:
   - **Gmail로 첨부 발송**: Gmail 노드에서 바이너리 데이터 첨부
   - **Google Drive에 저장**: Google Drive 노드의 Upload 기능 사용
   - **Slack에 공유**: Slack 노드의 파일 업로드 기능 사용

> ✅ **산출물**: AI 이미지 생성 워크플로우
>
> 💡 **프롬프트 팁**: "텍스트 포함하지 않기"를 명시하면 이미지 안의 텍스트 오류를 방지할 수 있습니다. 텍스트는 나중에 디자인 도구에서 별도로 추가하는 것이 품질이 좋습니다.

---

---

## 실습 17: 교육 자료 PDF AI 분석·요약

### 목표
PDF 교육 자료를 AI에 업로드하여 자동으로 요약하고, 학습 가이드를 생성합니다.

### 완성 워크플로우 구조
```
Manual Trigger → Google Drive (PDF 다운로드) → Anthropic 또는 Gemini (Analyze Document) → Edit Fields → Google Sheets (저장)
```

---

### Step 1: 워크플로우 생성

1. 새 워크플로우: **"실습17_PDF분석요약"**

2. **Manual Trigger** 추가

---

### Step 2: PDF 파일 가져오기

**방법 A: Google Drive에서 가져오기**

1. Manual Trigger 우측 **[+]** → **Google Drive** 선택
   - **Operation**: Download File
   - **File**: 4교시에서 업로드한 교육 자료 PDF 선택 (또는 더미 PDF)

**방법 B: HTTP Request로 URL에서 가져오기**

1. Manual Trigger 우측 **[+]** → **HTTP Request** 선택
   - **Method**: GET
   - **URL**: PDF 파일의 공개 URL
   - **Response Format**: File

---

### Step 3-A: Anthropic 노드로 PDF 분석 (Claude)

1. 파일 가져오기 노드 우측 **[+]** → **Anthropic** 선택

2. 설정:
   - **Credential**: "My Anthropic"
   - **Resource**: `Document`
   - **Operation**: `Analyze Document`
   - **Input Type**: `Binary`
     > 이전 노드에서 다운로드한 PDF 바이너리가 자동으로 전달됩니다
   - **Prompt**:
     ```
     이 교육 자료를 분석하여 아래 항목을 정리해주세요. 한국어로 작성하세요.
     
     1. 문서 요약 (3~5문장)
     2. 주요 학습 목표 (3개)
     3. 핵심 키워드 (5~10개, 쉼표로 구분)
     4. 난이도 평가 (초급/중급/고급)
     5. 예상 학습 시간
     6. 수강 전 필요한 사전 지식
     7. 이 자료를 기반으로 한 복습 퀴즈 3문항 (객관식, 각 4지선다)
     ```

3. **"Back to canvas"** 클릭

---

### Step 3-B: Google Gemini 노드로 PDF 분석 (대안)

1. 파일 가져오기 노드 우측 **[+]** → **Google Gemini** 선택

2. 설정:
   - **Credential**: "My Gemini"
   - **Resource**: `Document`
   - **Operation**: `Analyze Document`
   - **Input Type**: `Binary`
   - **Prompt**: 위 Step 3-A와 동일

---

### Step 4: 결과 저장

1. AI 노드 우측 **[+]** → **Edit Fields** 추가:

   | Name | Value (Expression) |
   |------|--------------------|
   | `파일명` | `(PDF 파일명 또는 고정값)` |
   | `분석결과` | Anthropic: `{{ $json.content[0].text }}` / Gemini: `{{ $json.text }}` |
   | `분석일시` | `{{ $now.toFormat('yyyy-MM-dd HH:mm') }}` |
   | `사용모델` | `claude-sonnet-4` 또는 `gemini-2.0-flash` |

2. Edit Fields 우측 **[+]** → **Google Sheets** (Append Row):
   - 새 스프레드시트 **"자료분석_로그"** (헤더: 파일명, 분석결과, 분석일시, 사용모델)

---

### Step 5: 테스트

1. **"Test workflow"** 실행

2. AI 노드 Output에서 PDF 분석 결과 확인:
   - 문서 요약
   - 학습 목표
   - 키워드
   - **자동 생성된 복습 퀴즈** 3문항

> ✅ **산출물**: PDF 교육 자료 AI 분석·요약 워크플로우
>
> 💡 **실무 활용**: 이 워크플로우를 Schedule Trigger로 연결하면, Google Drive에 새 교육 자료가 업로드될 때마다 자동으로 분석 리포트를 생성할 수 있습니다!

---

---

## 실습 18: 수업 녹음 자동 텍스트 변환 & 요약

### 목표
수업 녹음 오디오 파일을 Gemini로 텍스트 변환(STT)하고, OpenAI로 요약합니다.

### 완성 워크플로우 구조
```
Manual Trigger → HTTP Request (오디오 파일) → Google Gemini (Transcribe) → OpenAI (요약) → Google Sheets (저장) → Gmail (발송)
```

---

### Step 1: 워크플로우 생성 & 오디오 파일 준비

1. 새 워크플로우: **"실습18_녹음텍스트변환"**

2. **Manual Trigger** 추가

3. 오디오 파일 가져오기:

   **방법 A: Google Drive에서 다운로드**
   - Google Drive 노드 → Download File → mp3/wav 파일 선택

   **방법 B: HTTP Request로 URL에서 가져오기**
   - HTTP Request 노드 → GET → 오디오 파일 URL → Response Format: File

   > 💡 테스트용 오디오가 없다면 스마트폰으로 30초 정도 말하는 것을 녹음하여 Google Drive에 업로드하세요.

---

### Step 2: Google Gemini — 음성을 텍스트로 변환

1. 오디오 가져오기 노드 우측 **[+]** → **Google Gemini** 선택

2. 설정:
   - **Credential**: "My Gemini"
   - **Resource**: `Audio`
   - **Operation**: `Transcribe a Recording`
   - **Input Type**: `Binary`
     > 이전 노드에서 다운로드한 오디오 바이너리가 자동 전달

3. **"Back to canvas"** 클릭

---

### Step 3: OpenAI — 텍스트 요약

1. Google Gemini 우측 **[+]** → **OpenAI** 선택

2. 설정:
   - **Model**: `gpt-4o-mini`
   - **Message 1 (System)**:
     ```
     당신은 교육 콘텐츠 정리 전문가입니다.
     수업 녹취록을 분석하여 구조화된 학습 노트를 작성해주세요.
     한국어로 작성하세요.
     ```
   - **Message 2 (User)** (Expression 모드):
     ```
     아래는 수업 녹취록입니다:
     
     {{ $json.text }}
     
     위 녹취록을 기반으로 아래 항목을 정리해주세요:
     
     1. 수업 요약 (5문장 이내)
     2. 핵심 포인트 (3~5개, 번호 매기기)
     3. 중요 용어 설명 (3개)
     4. 복습을 위한 핵심 질문 (3개)
     5. 다음 수업 전 준비할 사항
     ```

---

### Step 4: 결과 저장 & 이메일 발송

1. OpenAI 우측 **[+]** → **Edit Fields**:

   | Name | Value (Expression) |
   |------|--------------------|
   | `녹취록` | `{{ $('Google Gemini').first().json.text }}` |
   | `학습노트` | `{{ $json.message.content }}` |
   | `생성일시` | `{{ $now.toFormat('yyyy-MM-dd HH:mm') }}` |

2. Edit Fields 우측 **[+]** → **Google Sheets** (Append Row):
   - 스프레드시트 **"수업녹취_로그"** (헤더: 녹취록, 학습노트, 생성일시)

3. Google Sheets 우측 **[+]** → **Gmail**:
   - **Subject**: `[학습 노트] 수업 요약이 준비되었습니다 — {{ $json.생성일시 }}`
   - **Message** (Expression):
     ```html
     <h1>수업 학습 노트</h1>
     <p><small>생성일시: {{ $json.생성일시 }} | AI 자동 생성</small></p>
     <hr>
     <div style="white-space:pre-wrap;">{{ $json.학습노트 }}</div>
     <hr>
     <details>
       <summary>원본 녹취록 보기</summary>
       <p style="color:#888;white-space:pre-wrap;">{{ $json.녹취록 }}</p>
     </details>
     ```

---

### Step 5: 테스트

1. **"Test workflow"** 실행

2. 확인:
   - Gemini 노드 → 오디오가 텍스트로 변환됨
   - OpenAI 노드 → 텍스트가 구조화된 학습 노트로 요약됨
   - Gmail → 학습 노트 이메일 수신

> ✅ **산출물**: 오디오 → 텍스트 변환 → AI 요약 → 학습 노트 이메일 발송
>
> 💡 **실무 활용 시나리오**:
> - 매 수업 종료 후 녹음 파일을 Google Drive에 업로드
> - Google Drive Trigger → 자동으로 텍스트 변환 & 요약
> - 수강생 전원에게 학습 노트 자동 이메일 발송

---

---

## 실습 19: 멀티모달 종합 파이프라인

### 목표
하나의 워크플로우에서 여러 AI 모델을 조합하여 멀티미디어 데이터를 처리합니다.

### 시나리오
> 수업 후 강사가 판서 사진(이미지) + 수업 요약 메모(텍스트)를 입력하면:
> 1. Gemini가 판서 이미지에서 텍스트를 추출
> 2. OpenAI가 추출 텍스트 + 강사 메모를 결합하여 학습 자료를 생성
> 3. 결과를 Google Sheets에 저장 & 수강생에게 이메일 발송

### 완성 워크플로우 구조
```
Manual Trigger
    → Edit Fields (이미지 URL + 강사 메모 입력)
    → Google Gemini (Analyze Image — 판서 텍스트 추출)
    → Code (추출 텍스트 + 강사 메모 결합)
    → OpenAI (학습 자료 생성)
    → Edit Fields (결과 정리)
    → Google Sheets (저장)
    → Gmail (수강생 발송)
```

---

### Step 1: 워크플로우 생성 & 입력 설정

1. 새 워크플로우: **"실습19_멀티모달파이프라인"**

2. **Manual Trigger** → **Edit Fields (Set)** 추가

3. 필드:

   | Name | Value (Fixed) |
   |------|---------------|
   | `과정명` | `n8n 워크플로우 자동화` |
   | `수업차시` | `3차시` |
   | `이미지URL` | `(판서 이미지 URL)` |
   | `강사메모` | `오늘 수업: Webhook 개념과 수강신청 자동화 실습. 핵심 포인트: 1) Webhook은 외부 이벤트를 수신하는 URL, 2) Test URL과 Production URL 차이, 3) JSON Body 파싱 방법. 다음 시간: AI API 연동` |

---

### Step 2: Gemini — 판서 이미지 분석

1. Edit Fields 우측 **[+]** → **Google Gemini** 선택

2. 설정:
   - **Resource**: Image
   - **Operation**: Analyze Image
   - **Input Type**: URL
   - **URL** (Expression): `{{ $json.이미지URL }}`
   - **Prompt**:
     ```
     이 칠판/화이트보드 사진에서 모든 텍스트와 다이어그램 내용을 추출해주세요.
     구조를 최대한 유지하며 한국어로 정리해주세요.
     다이어그램이 있으면 텍스트로 설명해주세요.
     ```

---

### Step 3: Code — 데이터 결합

1. Google Gemini 우측 **[+]** → **Code** 노드 추가

2. 코드:

   ```javascript
   const imageText = $json.text;
   const inputData = $('Edit Fields').first().json;
   
   return [{
     json: {
       과정명: inputData.과정명,
       수업차시: inputData.수업차시,
       판서내용: imageText,
       강사메모: inputData.강사메모,
       결합텍스트: `[판서 내용]\n${imageText}\n\n[강사 메모]\n${inputData.강사메모}`
     }
   }];
   ```

---

### Step 4: OpenAI — 학습 자료 생성

1. Code 노드 우측 **[+]** → **OpenAI** 선택

2. 설정:
   - **Model**: `gpt-4o-mini`
   - **Message 1 (System)**:
     ```
     당신은 교육 콘텐츠 전문 에디터입니다.
     수업 판서 내용과 강사 메모를 기반으로 정리된 학습 자료를 만들어주세요.
     수강생이 복습하기 쉬운 형태로 작성하세요.
     한국어로 작성하세요.
     ```
   - **Message 2 (User)** (Expression 모드):
     ```
     과정명: {{ $json.과정명 }} — {{ $json.수업차시 }}
     
     {{ $json.결합텍스트 }}
     
     위 내용을 기반으로 학습 자료를 작성해주세요:
     1. 오늘의 학습 목표 (3개)
     2. 핵심 개념 정리 (항목별 2~3문장 설명)
     3. 실습 요약 (순서대로 정리)
     4. 핵심 용어 사전 (5개, 용어: 설명 형태)
     5. 복습 체크리스트 (5개 항목)
     6. 다음 수업 예습 포인트
     ```

---

### Step 5: 결과 정리 & 발송

1. OpenAI 우측 **[+]** → **Edit Fields**:

   | Name | Value (Expression) |
   |------|--------------------|
   | `과정명` | `{{ $('Code').first().json.과정명 }}` |
   | `수업차시` | `{{ $('Code').first().json.수업차시 }}` |
   | `학습자료` | `{{ $json.message.content }}` |
   | `생성일시` | `{{ $now.toFormat('yyyy-MM-dd HH:mm') }}` |

2. → **Google Sheets** (Append Row) → **Gmail**:
   - **Subject**: `[{{ $json.과정명 }}] {{ $json.수업차시 }} 학습 자료가 준비되었습니다`
   - **Message**:
     ```html
     <h1>{{ $json.과정명 }} — {{ $json.수업차시 }} 학습 자료</h1>
     <p><small>{{ $json.생성일시 }} | AI 자동 생성</small></p>
     <hr>
     <div style="white-space:pre-wrap;">{{ $json.학습자료 }}</div>
     ```

---

### Step 6: 테스트

1. **"Test workflow"** 실행

2. 단계별 Output 확인:
   - Gemini → 이미지에서 텍스트 추출 결과
   - Code → 판서 + 강사 메모 결합
   - OpenAI → 구조화된 학습 자료
   - Gmail → 최종 이메일

> ✅ **산출물**: 이미지 분석 + 텍스트 처리 + AI 생성 = 멀티모달 종합 파이프라인

---

---

## 멀티미디어 AI 노드 빠른 참조표 (n8n 2.12.3)

### Google Gemini 노드

| Resource | Operation | 입력 | 출력 참조 | 용도 |
|----------|-----------|------|-----------|------|
| Image | Analyze Image | URL 또는 Binary | `$json.text` | 이미지 속 텍스트/내용 분석 |
| Image | Generate an Image | Text Prompt | Binary (이미지) | 텍스트→이미지 생성 |
| Document | Analyze Document | Binary (PDF 등) | `$json.text` | 문서 내용 분석·요약 |
| Audio | Analyze Audio | Binary (mp3 등) | `$json.text` | 오디오 내용 분석·질문 |
| Audio | Transcribe a Recording | Binary | `$json.text` | 음성→텍스트 변환 (STT) |
| Message | Message a Model | Text | `$json.text` | 일반 텍스트 대화 |

### Anthropic 노드

| Resource | Operation | 입력 | 출력 참조 | 용도 |
|----------|-----------|------|-----------|------|
| Document | Analyze Document | Binary (PDF 등) | `$json.content[0].text` | 문서 분석 (최대 200K 토큰) |
| Image | Analyze Image | URL 또는 Binary | `$json.content[0].text` | 이미지 분석 |
| Message | Message a Model | Text | `$json.content[0].text` | 일반 텍스트 대화 |
| Prompt | Generate / Improve | Text | `$json.content[0].text` | 프롬프트 자동 생성·개선 |

### OpenAI 노드

| Resource | Operation | 입력 | 출력 참조 | 용도 |
|----------|-----------|------|-----------|------|
| Message | Send | Text | `$json.message.content` | 일반 텍스트 대화 |
| Image | Generate | Text Prompt | Binary (이미지) | DALL-E 이미지 생성 |
| Audio | Transcribe | Binary (mp3 등) | `$json.text` | Whisper 음성→텍스트 |
| Audio | Translate | Binary | `$json.text` | 음성 번역 |

---

## 모델 선택 가이드 — 멀티미디어 작업별

| 작업 | 추천 1순위 | 추천 2순위 | 이유 |
|------|-----------|-----------|------|
| **이미지 OCR** | Gemini | Claude | Gemini 무료 + 빠른 속도 |
| **이미지 생성** | Gemini (Imagen) | OpenAI (DALL-E) | Gemini 비용 효율적 |
| **PDF 문서 분석** | Claude | Gemini | Claude 긴 문서 처리 최강 |
| **음성→텍스트** | Gemini | OpenAI (Whisper) | Gemini 다국어 무료 |
| **텍스트 요약** | OpenAI | Claude | GPT-4o-mini 가성비 |
| **구조화 분석** | Claude | OpenAI | Claude 지시 수행 정확도 |

---

## n8n 2.12.3 멀티미디어 작업 시 주의사항

1. **바이너리 데이터 전달**: 파일 다운로드 → AI 분석 노드 연결 시 바이너리가 자동 전달됨. 별도 변환 불필요
2. **파일 크기 제한**: 
   - Gemini: 단일 파일 최대 20MB (인라인), 2GB (File API)
   - Claude: 단일 파일 최대 32MB
   - OpenAI Whisper: 오디오 최대 25MB
3. **지원 포맷**:
   - 이미지: PNG, JPEG, GIF, WebP
   - 문서: PDF (텍스트 기반 + 스캔 모두 가능)
   - 오디오: MP3, WAV, FLAC, M4A, OGG
4. **Publish 필수**: Webhook/Schedule 기반 멀티미디어 처리는 Publish 후에만 실제 동작
5. **바이너리 저장**: n8n 2.x에서 바이너리 데이터는 filesystem 또는 database에 저장 (인메모리 X)

---

## 7교시 산출물 체크리스트

- [ ] [실습 15] 이미지 OCR → JSON 파싱 → Sheets 저장 워크플로우
- [ ] [실습 16] AI 이미지 생성 워크플로우 (또는 데모 확인)
- [ ] [실습 17] PDF 교육 자료 AI 분석·요약 + 자동 퀴즈 생성
- [ ] [실습 18] 오디오 → 텍스트 변환 → AI 요약 → 학습 노트 이메일
- [ ] [실습 19] 멀티모달 종합 파이프라인 (이미지+텍스트 → AI 학습 자료)
- [ ] 3대 AI 모델별 멀티미디어 노드·출력 경로 차이 이해
- [ ] 바이너리 데이터 흐름 및 파일 크기 제한 숙지

---

> **다음 교시 예고**: 8교시에서는 2일간 배운 모든 기술을 종합하여 팀별 프로젝트를 설계·구현하고 발표합니다!
