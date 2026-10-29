# Studio Agent 스타터 코드

대구·울산·경북 대학 연합 AI Workflow 해커톤 2기에서 Upstage Studio를 미리 경험해 보기 위한 스타터 코드입니다. **미니 미션 1을 위한 자료**입니다.

- Playbook: https://daeul-hackathon-playbook.vercel.app
- Upstage Studio 사용 설명서: https://console.upstage.ai/ko/docs/studio

이 코드에는 Studio에서 만든 에이전트를 API로 실행해 결과 JSON을 받는 부분이 들어 있습니다.

- 에이전트는 Studio 화면에서 만듭니다. 코드 없이도 해커톤을 진행할 수 있습니다.
- 코드는 같은 에이전트를 여러 문서에 한 번에 돌려 결과를 모아 볼 때 씁니다. 미션 2의 테스트 데이터셋 10건을 돌릴 때 유용합니다.

---

## 예시: HR 이력서 검토 에이전트 3개

예시 에이전트는 **가장 기초적인 노드(Parse · Classify · Extract)만** 써서 Studio를 경험해 보는 구성입니다. Studio의 전체 노드는 Playbook 1-1에서 소개합니다. 팀 워크플로우에서는 이 노드들까지 조합할 수 있습니다.

| 에이전트 | 입력 문서 | 노드 구성 | 하는 일 |
|---|---|---|---|
| Resume Agent | `data/Resume.pdf` | Parse → Extract | 이력서에서 이름·기술·경력·학력 추출 |
| JD Agent | `data/JobDescription.pdf` | Parse → Extract | 채용공고에서 필수·우대 기술, 자격요건 추출 |
| Matching Agent | `data/Matching_Input.pdf` | Parse → Classify → Extract 3개 | 이력서와 공고를 비교해 High / Medium / Low Match 분류 후 평가 |

`Matching_Input.pdf` 는 이력서와 채용공고 추출 결과를 한 문서로 합친 샘플입니다.

각 에이전트의 Studio 설정은 `agent_configs/` 에, 실행 결과 예시는 `examples/` 에 있습니다.

---

## 폴더 구조

```text
studio-agent-starter/
├── run_agent.py        # 실행 진입점: 문서 1개 또는 폴더 전체를 에이전트에 실행
├── agent_client.py     # Studio Agent 호출 (Job 생성 → 상태 조회 → JSON 파싱)
├── upload.py           # Upstage Files API로 문서 업로드
├── config.py           # API Key, Agent ID, 폴더 경로
├── requirements.txt
├── .env.example
├── agent_configs/      # 예시 에이전트 3개의 Studio 설정 (워크플로우 설정 내보내기 파일)
├── data/               # 샘플 입력 문서
├── examples/           # 샘플 실행 결과 JSON
└── result/             # 실행하면 생기는 결과 폴더
```

---

## 실행 흐름

```text
문서(PDF 등) → upload.py (File ID 발급) → agent_client.py (Studio Agent 실행) → result/<문서이름>.json
```

---

## 시작하기

### 1. 설치

Python 3.11 이상이 필요합니다.

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. `.env` 만들기

`.env.example` 을 복사해 `.env` 를 만들고 값을 넣습니다.

```text
UPSTAGE_API_KEY=up_xxxxxxxxxxxxxxxx
AGENT_ID=agt_xxxxxxxxxxxxxxxx
CONFIG_ID=1
```

- `UPSTAGE_API_KEY` : Upstage Console에서 발급
- `AGENT_ID`, `CONFIG_ID` : Studio 에이전트 화면 오른쪽 위 메뉴 → `API로 이용하기` 에서 복사

`.env` 는 GitHub에 올리지 않습니다.

### 3. 실행

```bash
# 문서 1개
python run_agent.py data/Resume.pdf

# 폴더 안 문서 전부 (예: 팀 테스트 데이터셋)
python run_agent.py my_dataset/

# .env 와 다른 에이전트로 실행
python run_agent.py data/JobDescription.pdf --agent-id agt_xxxx --config-id 1
```

결과는 `result/<문서이름>.json` 으로 저장됩니다. 실패한 문서는 마지막에 목록으로 나옵니다.

---

## 팀 워크플로우에 적용하기

1. Studio에서 팀 워크플로우를 만들고 샘플 문서로 실행해 결과를 확인합니다.
2. `API로 이용하기` 에서 Agent ID와 Config ID를 `.env` 에 넣습니다.
3. 테스트 데이터셋 문서를 한 폴더에 모아 `python run_agent.py 폴더/` 로 실행합니다.
4. `result/` 의 JSON을 정답지와 비교합니다.

코드는 바꾸지 않아도 됩니다. 중요한 것은 Studio 워크플로우의 설계와 출력 키입니다.

- **마지막 노드의 출력 키 = 정답지 키** 로 맞춥니다. 키 이름이 다르면 심사에서 그 항목은 0점입니다.
- 확실하지 않은 값은 추측하지 않고 `검토 필요` 로 출력하도록 설계합니다.

---

## 문제 해결

| 증상 | 해결 |
|---|---|
| `다음 환경변수가 설정되지 않았습니다.` | `.env` 가 `config.py` 와 같은 폴더에 있는지, 값이 비어 있지 않은지 확인 |
| `401 Unauthorized` | `UPSTAGE_API_KEY` 확인. Console에서 재발급 |
| `Agent Not Found`, 404 | Agent ID를 다시 복사. 앞뒤 공백 확인 |
| status `failed` | Studio에서 같은 파일로 직접 실행해 어느 노드에서 실패하는지 확인 |
| `output_text가 JSON 형식이 아닙니다.` | 마지막 노드가 정해진 스키마(JSON)로 출력하는지 확인 |
| `in_progress` 가 오래 계속됨 | 페이지 수가 많은 문서는 오래 걸립니다. 테스트는 2~3쪽 문서로 먼저 진행 |
