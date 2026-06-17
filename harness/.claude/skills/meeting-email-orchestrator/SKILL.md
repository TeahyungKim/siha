---
name: meeting-email-orchestrator
description: "회의록을 입력받아 결정사항·Action Item을 정리하고 인사정보 등 민감 정보를 제외한 비즈니스 이메일을 생성하는 오케스트레이터. '회의록 요약해줘', '회의 결과 이메일 작성해줘', '미팅 내용 정리해서 메일로', '회의록 이메일 초안', '회의 내용 공유 메일' 요청 시 반드시 이 스킬을 사용. 후속 작업: 이메일 수정, 다시 작성, 톤 조정, Action Item 보완, 회의록 업데이트 후 재실행 요청에도 사용."
---

# Meeting Email Orchestrator

회의록을 분석하여 결정사항·Action Item을 꼼꼼히 정리하고, 인사정보 등 민감 데이터를 제거한 비즈니스 이메일을 생성하는 통합 워크플로우.

## 실행 모드: 하이브리드

| Phase | 모드 | 이유 |
|-------|------|------|
| Phase 2 (회의록 분석) | 서브 에이전트 | 단일 작업, 팀 통신 불필요 |
| Phase 3 (추출 + 필터링) | 에이전트 팀 | 추출자↔필터 상호 검토로 누락·오류 방지 |
| Phase 4 (작성 + 검토) | 에이전트 팀 | 작성자↔검토자 실시간 피드백으로 품질 보장 |

## 에이전트 구성

| 에이전트 | 타입 | 역할 | 스킬 | 출력 |
|---------|------|------|------|------|
| meeting-analyzer | general-purpose | 회의록 구조 분석 | meeting-analysis | `_workspace/01_analysis.md` |
| content-extractor | general-purpose | 결정사항/Action Item 추출 | content-extraction | `_workspace/02_extracted_content.md` |
| privacy-filter | general-purpose | 민감 정보 필터링 | privacy-filtering | `_workspace/02_filtered_content.md`, `_workspace/02_privacy_log.md` |
| email-writer | general-purpose | 이메일 초안 작성 | email-composition | `_workspace/03_email_draft.md` |
| email-reviewer | general-purpose | 이메일 품질 검토 | — | `_workspace/03_review_notes.md` |

## 워크플로우

### Phase 0: 컨텍스트 확인

기존 산출물 존재 여부를 확인하여 실행 모드를 결정한다:

1. `_workspace/` 디렉토리 존재 여부 확인
2. 실행 모드 결정:
   - `_workspace/` 미존재 → **초기 실행**, Phase 1로 진행
   - `_workspace/` 존재 + 부분 수정 요청 (예: "Action Item만 다시 정리해줘") → **부분 재실행**, 해당 Phase부터 재시작
   - `_workspace/` 존재 + 새 회의록 입력 → **새 실행**, 기존 `_workspace/`를 `_workspace_prev/`로 이동 후 Phase 1 진행

### Phase 1: 준비

1. 사용자 입력 확인:
   - 회의록 파일 경로 또는 텍스트 형태 수신
   - 이메일 수신자 범위 확인 (내부 전체 / 특정 팀 / 경영진 등)
2. `_workspace/` 디렉토리 생성
3. 회의록 원문을 `_workspace/00_input_minutes.md`에 저장

### Phase 2: 회의록 분석 (서브 에이전트 모드)

**실행 모드:** 서브 에이전트

meeting-analyzer를 서브 에이전트로 호출:

```
Agent(
  subagent_type: "meeting-analyzer",
  model: "opus",
  prompt: "meeting-analysis 스킬(.claude/skills/meeting-analysis/SKILL.md)을 먼저 읽어라.
           그 다음 _workspace/00_input_minutes.md를 읽고 회의록을 분석하여
           _workspace/01_analysis.md에 결과를 저장하라.",
  run_in_background: false
)
```

완료 후 `_workspace/01_analysis.md` 존재 여부 확인.

### Phase 3: 추출 + 필터링 (에이전트 팀 모드)

**실행 모드:** 에이전트 팀

1. 팀 구성:

```
TeamCreate(
  team_name: "extraction-team",
  members: [
    {
      name: "content-extractor",
      agent_type: "content-extractor",
      model: "opus",
      prompt: "content-extraction 스킬(.claude/skills/content-extraction/SKILL.md)을 먼저 읽어라.
               그 다음 _workspace/01_analysis.md를 입력으로 결정사항과 Action Item을 추출하여
               _workspace/02_extracted_content.md에 저장하라.
               완료 후 privacy-filter에게 SendMessage로 '추출 완료, 필터링 시작 가능' 알림."
    },
    {
      name: "privacy-filter",
      agent_type: "privacy-filter",
      model: "opus",
      prompt: "privacy-filtering 스킬(.claude/skills/privacy-filtering/SKILL.md)을 먼저 읽어라.
               content-extractor로부터 완료 알림을 받은 뒤
               _workspace/02_extracted_content.md를 필터링하여
               _workspace/02_filtered_content.md와 _workspace/02_privacy_log.md를 생성하라.
               완료 후 리더에게 주요 제거 항목 요약 보고."
    }
  ]
)
```

2. 작업 등록:

```
TaskCreate(tasks: [
  {
    title: "결정사항/Action Item 추출",
    description: "_workspace/01_analysis.md 기반으로 추출",
    assignee: "content-extractor"
  },
  {
    title: "민감 정보 필터링",
    description: "_workspace/02_extracted_content.md 기반으로 필터링",
    assignee: "privacy-filter",
    depends_on: ["결정사항/Action Item 추출"]
  }
])
```

3. 두 작업 완료 확인 후 팀 정리: `TeamDelete("extraction-team")`

### Phase 4: 이메일 작성 + 검토 (에이전트 팀 모드)

**실행 모드:** 에이전트 팀

1. 팀 구성:

```
TeamCreate(
  team_name: "writing-team",
  members: [
    {
      name: "email-writer",
      agent_type: "email-writer",
      model: "opus",
      prompt: "email-composition 스킬(.claude/skills/email-composition/SKILL.md)을 먼저 읽어라.
               _workspace/02_filtered_content.md(핵심 소재)와
               _workspace/01_analysis.md(맥락 참조)를 바탕으로 이메일 초안을 작성하여
               _workspace/03_email_draft.md에 저장하라.
               완료 후 email-reviewer에게 SendMessage로 검토 요청."
    },
    {
      name: "email-reviewer",
      agent_type: "email-reviewer",
      model: "opus",
      prompt: "email-writer의 검토 요청을 받은 뒤
               _workspace/03_email_draft.md와 _workspace/02_filtered_content.md를
               교차 비교하여 검토하라.
               수정 필요 사항은 email-writer에게 SendMessage로 구체적으로 전달.
               최대 2회 수정 사이클 허용.
               최종 승인 시 리더에게 완료 보고."
    }
  ]
)
```

2. 작업 등록:

```
TaskCreate(tasks: [
  {
    title: "이메일 초안 작성",
    assignee: "email-writer"
  },
  {
    title: "이메일 검토 및 최종 승인",
    assignee: "email-reviewer",
    depends_on: ["이메일 초안 작성"]
  }
])
```

3. email-reviewer의 최종 승인 확인 후 팀 정리: `TeamDelete("writing-team")`

### Phase 5: 최종 산출물 정리

1. `_workspace/03_email_draft.md` 읽어 최종 이메일 내용 확인
2. 최종 이메일을 `meeting_email_{YYYYMMDD}.md`로 저장
3. 사용자에게 보고:
   - 생성된 이메일 전문
   - 필터링된 민감 정보 요약 (`_workspace/02_privacy_log.md` 기반)
   - 확인이 필요한 `[확인 필요: ...]` 플레이스홀더 목록

## 데이터 흐름

```
[회의록 입력]
      ↓
[meeting-analyzer(서브)] → _workspace/01_analysis.md
      ↓
[content-extractor(팀)] ──→ _workspace/02_extracted_content.md
         ↕ SendMessage
[privacy-filter(팀)]    ──→ _workspace/02_filtered_content.md
                              _workspace/02_privacy_log.md
      ↓
[email-writer(팀)] ──→ _workspace/03_email_draft.md
      ↕ SendMessage
[email-reviewer(팀)] ──→ _workspace/03_review_notes.md
      ↓
[최종] meeting_email_{YYYYMMDD}.md
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| meeting-analyzer 실패 | 1회 재시도. 재실패 시 사용자에게 회의록 형식 확인 요청 |
| content-extractor 실패 | 오케스트레이터가 01_analysis.md 기반으로 직접 추출 후 진행 가능 여부 판단 |
| privacy-filter 실패 | 사용자에게 알림: "자동 필터링 실패. 02_extracted_content.md 수동 검토 후 계속 진행하시겠습니까?" |
| 작성-검토 수정 2회 초과 | 현재 초안을 사용자에게 제시하고 직접 수정 요청 |
| 민감 정보 제거 후 내용 부족 | 사용자에게 알리고 추가 정보 제공 요청 |

## 테스트 시나리오

### 정상 흐름
1. 사용자가 `회의록샘플/03_채용결정회의.md` 제공
2. Phase 2: meeting-analyzer가 채용 안건 분석, 지원자 점수·연봉·평판 조회 플래그 표시
3. Phase 3: content-extractor가 결정사항 4건·Action Item 5건 추출 → privacy-filter가 연봉 수치·평판 조회·점수 제거, 지원자명 역할로 대체
4. Phase 4: email-writer가 "1순위 지원자 오퍼 진행" 수준으로 이메일 초안 작성 → email-reviewer가 민감정보 잔류 없음 확인 후 승인
5. 최종 이메일 + "지원자 연봉 정보, 평판 조회, 점수 제거됨" 요약 보고

### 에러 흐름
1. Phase 3에서 privacy-filter 에러 발생
2. 리더가 감지 → 사용자에게 알림: "개인정보 자동 필터링에 실패했습니다. `_workspace/02_extracted_content.md`에 민감 정보가 포함될 수 있습니다. 수동 검토 후 계속 진행하시겠습니까?"
3. 사용자 승인 시 Phase 4 진행 (이메일 하단에 "※ 자동 개인정보 필터링 미적용 — 발송 전 수동 확인 필요" 주석 자동 추가)
