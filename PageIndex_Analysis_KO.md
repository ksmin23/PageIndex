# PageIndex 코드 종합 분석 보고서

---

## 1. 아키텍처 개요

PageIndex는 **벡터 DB 없이(Vectorless)**, **LLM 추론(Reasoning) 기반**으로 문서를 검색하는 RAG 프레임워크입니다. 핵심 아이디어는 문서를 **계층적 트리 구조(목차와 유사)**로 변환하고, 이 트리를 LLM이 탐색하여 관련 정보를 찾는 것입니다.

### 프로젝트 구조

```
PageIndex/
├── run_pageindex.py              # CLI 진입점
├── requirements.txt              # 의존성 (litellm, pymupdf, PyPDF2 등)
├── pageindex/
│   ├── __init__.py               # page_index, page_index_md 모듈 export
│   ├── config.yaml               # 기본 설정값 (모델, 파라미터)
│   ├── page_index.py             # 핵심: PDF → 트리 구조 생성 파이프라인
│   ├── page_index_md.py          # Markdown → 트리 구조 생성
│   └── utils.py                  # LLM 호출, PDF 파싱, 유틸리티 함수
└── cookbook/
    ├── pageindex_RAG_simple.ipynb # 기본 RAG 예제
    ├── agentic_retrieval.ipynb    # 에이전틱 검색 예제
    └── vision_RAG_pageindex.ipynb # 비전 기반 RAG 예제
```

### 모듈 간 관계

```
run_pageindex.py
    ├── pageindex.page_index.page_index_main()   ← PDF 처리
    └── pageindex.page_index_md.md_to_tree()     ← Markdown 처리
            │
            └── pageindex.utils
                    ├── llm_completion() / llm_acompletion()  ← LiteLLM 기반 LLM 호출
                    ├── get_page_tokens()                     ← PDF 텍스트 추출
                    ├── ConfigLoader                          ← 설정 관리
                    └── 각종 트리 조작/JSON 유틸리티
```

---

## 2. Documents 구조화 방법

PageIndex는 PDF/Markdown 문서를 **계층적 트리(Tree) 구조**로 변환합니다. 핵심은 **벡터 임베딩이나 청킹(Chunking)을 사용하지 않고**, 문서의 자연스러운 섹션 구조를 그대로 활용한다는 점입니다.

### 2.1 PDF 구조화 파이프라인 (`pageindex/page_index.py`)

전체 흐름은 `page_index_main()` → `tree_parser()` → `meta_processor()`로 진행됩니다:

**Step 1: PDF 텍스트 추출** (`utils.py:382-405`)
```
PDF → PyPDF2/PyMuPDF로 페이지별 텍스트 + 토큰 수 추출
```

**Step 2: 목차(TOC) 탐지** (`page_index.py:341-366`)
- 처음 N페이지(기본 20)를 LLM에게 보내서 "이 페이지에 목차가 있는가?" 판단
- 3가지 경우로 분기:
  - **목차 O + 페이지 번호 O** → `process_toc_with_page_numbers`
  - **목차 O + 페이지 번호 X** → `process_toc_no_page_numbers`
  - **목차 X** → `process_no_toc` (LLM이 직접 구조 생성)

**Step 3: 트리 구조 생성** (`page_index.py:958-997`)
- 목차가 있으면: LLM이 목차를 JSON 형식으로 변환하고, 각 섹션의 물리적 페이지 인덱스를 매칭
- 목차가 없으면: 문서를 토큰 기준 그룹으로 나누고, LLM이 직접 계층 구조를 생성

**Step 4: 검증 및 수정** (`page_index.py:900-952`)
- 생성된 트리의 각 노드에 대해 "해당 제목이 해당 페이지에 실제로 존재하는가?" 를 LLM으로 비동기 검증
- 정확도가 60% 이상이면 틀린 항목만 수정, 60% 미만이면 폴백 모드로 재처리

**Step 5: 대형 노드 재귀 분할** (`page_index.py:1000-1027`)
- 페이지 수 > `max_page_num_each_node`(기본 10) AND 토큰 수 > `max_token_num_each_node`(기본 20000)인 노드는 재귀적으로 하위 분할

**최종 트리 출력 구조:**
```json
{
  "doc_name": "document.pdf",
  "structure": [
    {
      "title": "섹션 제목",
      "node_id": "0001",
      "start_index": 1,
      "end_index": 5,
      "summary": "섹션 요약",
      "nodes": [...]
    }
  ]
}
```

### 2.2 Markdown 구조화 파이프라인 (`pageindex/page_index_md.py`)

Markdown의 경우 LLM 없이도 기본 구조화가 가능합니다:

1. `#` 헤딩 레벨로 계층 구조를 **직접 파싱** (LLM 불필요)
2. 각 노드에 해당하는 텍스트 내용 추출
3. (선택) **Tree Thinning**: 토큰 수가 임계값 미만인 노드를 부모에 병합
4. (선택) 요약 생성 시에만 LLM 호출

---

## 3. 사용자 프롬프트에 대한 답변 방법

**중요: 이 오픈소스 저장소 자체는 "트리 구조 생성"까지만 담당합니다. 질의응답(Q&A)은 별도의 워크플로우로 구현해야 합니다.**

`cookbook/pageindex_RAG_simple.ipynb`에서 보여주는 전체 RAG 파이프라인:

### Step 1: 트리 구조 생성 (이 저장소의 역할)
```
PDF → PageIndex → 계층적 트리 (각 노드에 제목, 페이지 범위, 요약 포함)
```

### Step 2: 트리 검색 (Reasoning-based Retrieval)
LLM에게 트리 구조 + 질문을 주고, **관련 노드를 추론으로 선택**하게 합니다:
```python
search_prompt = f"""
질문과 문서의 트리 구조가 주어집니다.
각 노드에는 node_id, 제목, 요약이 포함됩니다.
질문에 대한 답을 포함할 가능성이 높은 노드를 모두 찾으세요.
...
"""
# LLM이 관련 node_id 목록을 반환
```

### Step 3: 컨텍스트 추출
선택된 노드의 `text` (실제 페이지 텍스트)를 추출합니다.

### Step 4: 답변 생성
추출된 컨텍스트 + 질문을 LLM에게 주어 최종 답변을 생성합니다.

> 핵심 차별점: 벡터 유사도(similarity)가 아닌 LLM의 **추론(reasoning)**으로 관련 섹션을 찾습니다. 사람이 목차를 보고 필요한 부분을 찾아가는 방식과 동일합니다.

---

## 4. PageIndex API Key 없이 사용하는 방법

이 저장소는 **완전한 셀프 호스팅을 지원**합니다. PageIndex API Key는 필요 없습니다.

### 필요한 것: LLM API Key만 있으면 됨

```bash
# 1. 의존성 설치
pip3 install --upgrade -r requirements.txt

# 2. .env 파일에 LLM API 키 설정
echo "CHATGPT_API_KEY=sk-your-openai-key" > .env
# 또는 다른 프로바이더 키 설정 (아래 섹션 6 참조)

# 3. 실행
python3 run_pageindex.py --pdf_path /path/to/document.pdf
```

### Python API로 직접 호출:
```python
from pageindex import page_index

result = page_index(
    doc="path/to/document.pdf",
    model="gpt-4o-2024-11-20",
    if_add_node_summary="yes"
)
```

> **PageIndex API Key**(`PAGEINDEX_API_KEY`)는 Vectify의 **클라우드 서비스** (Chat Platform, MCP, 호스팅 API)를 사용할 때만 필요합니다. 이 GitHub 저장소의 코드를 직접 실행할 때는 LLM 프로바이더의 API Key만 있으면 됩니다.

---

## 5. OpenAI 모델이 필요한 부분과 그렇지 않은 부분

### LLM이 **반드시 필요한** 부분 (PDF 처리)

| 기능 | 위치 | 설명 |
|------|------|------|
| 목차 탐지 | `page_index.py:104-122` | 각 페이지에 목차가 있는지 LLM이 판단 |
| 목차 변환 | `page_index.py:273-336` | 원시 목차를 JSON 구조로 변환 |
| 트리 구조 생성 | `page_index.py:542-574` | 목차 없는 문서의 구조를 LLM이 직접 생성 |
| 페이지 인덱스 매핑 | `page_index.py:243-269` | 섹션 제목을 물리적 페이지에 매칭 |
| 검증/수정 | `page_index.py:13-45` | 생성된 매핑이 맞는지 검증 |
| 노드 요약 생성 | `utils.py:573-581` | 각 노드의 텍스트 요약 (선택적) |
| 문서 설명 생성 | `utils.py:617-626` | 문서 전체 설명 (선택적) |

### LLM이 **필요 없는** 부분

| 기능 | 위치 | 설명 |
|------|------|------|
| PDF 텍스트 추출 | `utils.py:216-241` | PyPDF2/PyMuPDF로 직접 처리 |
| 토큰 카운팅 | `utils.py:25-28` | LiteLLM의 `token_counter` 사용 (로컬 처리) |
| 트리 자료구조 조작 | `utils.py:127-214` | write_node_id, get_nodes, list_to_tree 등 |
| Markdown 파싱 | `page_index_md.py:32-87` | `#` 헤딩 기반 구조 추출 (정규식) |
| Tree Thinning | `page_index_md.py:135-187` | 토큰 기준 노드 병합 (로컬 처리) |
| 설정 로딩 | `utils.py:649-681` | YAML 파일 읽기 |
| 결과 저장 | `run_pageindex.py:72-80` | JSON 파일 쓰기 |

> **핵심 포인트**: "OpenAI 모델"이 반드시 필요한 것은 아닙니다. LiteLLM을 통해 **어떤 LLM이든** 사용 가능합니다 (다음 섹션 참조).

---

## 6. 다른 AI 모델 사용 방법

PageIndex는 **LiteLLM을 통합**하여 멀티 프로바이더를 지원합니다.

### LiteLLM 통합 구조 (`utils.py:1-23`)

```python
import litellm

# 모든 LLM 호출이 litellm.completion() / litellm.acompletion()을 사용
def llm_completion(model, prompt, ...):
    response = litellm.completion(model=model, messages=messages, temperature=0)
    ...

async def llm_acompletion(model, prompt):
    response = await litellm.acompletion(model=model, messages=messages, temperature=0)
    ...
```

### 다른 모델 사용법

**방법 1: CLI 파라미터로 지정**
```bash
# Anthropic Claude
export ANTHROPIC_API_KEY=sk-ant-your-key
python3 run_pageindex.py --pdf_path doc.pdf --model anthropic/claude-sonnet-4-20250514

# Google Gemini
export GEMINI_API_KEY=your-key
python3 run_pageindex.py --pdf_path doc.pdf --model gemini/gemini-2.5-pro

# Azure OpenAI
export AZURE_API_KEY=your-key
export AZURE_API_BASE=https://your-resource.openai.azure.com
python3 run_pageindex.py --pdf_path doc.pdf --model azure/gpt-4o
```

**방법 2: `config.yaml` 수정**
```yaml
# config.yaml에 이미 Claude 예시가 주석으로 포함되어 있음
# model: "gpt-4o-2024-11-20"
model: "anthropic/claude-sonnet-4-6"   # 이렇게 변경
```

**방법 3: Python API에서 직접 지정**
```python
from pageindex import page_index

result = page_index(
    doc="document.pdf",
    model="anthropic/claude-sonnet-4-20250514"
)
```

### LiteLLM이 지원하는 주요 프로바이더

| 프로바이더 | 모델 형식 | 환경변수 |
|-----------|----------|---------|
| OpenAI | `gpt-4o`, `gpt-4.1` | `OPENAI_API_KEY` |
| Anthropic | `anthropic/claude-sonnet-4-20250514` | `ANTHROPIC_API_KEY` |
| Google | `gemini/gemini-2.5-pro` | `GEMINI_API_KEY` |
| Azure | `azure/gpt-4o` | `AZURE_API_KEY` + `AZURE_API_BASE` |
| AWS Bedrock | `bedrock/anthropic.claude-v2` | AWS 자격증명 |
| Ollama (로컬) | `ollama/llama3` | 별도 키 불필요 |
| Together AI | `together_ai/...` | `TOGETHERAI_API_KEY` |

> **참고**: `utils.py:19-21`에서 `CHATGPT_API_KEY`를 `OPENAI_API_KEY`로 자동 매핑하는 하위 호환성 코드가 있습니다. 다른 프로바이더 사용 시에는 해당 프로바이더의 환경변수만 설정하면 됩니다.

---

## 요약

| 항목 | 설명 |
|------|------|
| **핵심 철학** | 벡터 유사도 대신 LLM 추론으로 검색 (Similarity ≠ Relevance) |
| **트리 생성** | LLM이 문서의 목차/구조를 파악 → 계층적 JSON 트리 |
| **질의응답** | 트리 구조를 LLM에게 제공 → 추론으로 관련 노드 선택 → 컨텍스트 기반 답변 |
| **API Key** | PageIndex API Key 불필요 (셀프 호스팅), LLM API Key만 필요 |
| **모델 의존성** | OpenAI 고정이 아닌 LiteLLM 기반 멀티 프로바이더 지원 |
| **LLM 불필요 영역** | PDF 파싱, Markdown 구조 추출, 토큰 카운팅, 트리 자료구조 조작 |
