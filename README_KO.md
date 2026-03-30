# Website Quality Checker

[English](README.md) | **한국어**

[Google Search Quality Rater Guidelines(QRG)](https://static.googleusercontent.com/media/guidelines.raterhub.com/en//searchqualityevaluatorguidelines.pdf)를 기반으로 웹페이지 품질을 검사하는 Claude Code 스킬입니다. URL을 입력하거나 HTML/텍스트를 붙여넣으면, 6개 카테고리에 걸쳐 종합 등급(S/A/B/C/D)이 포함된 상세 보고서를 생성합니다.

## 주요 기능

- **QRG 기반 평가** — SEO 휴리스틱이 아닌, Google 공식 Search Quality Rater Guidelines를 체계적으로 적용
- **6개 카테고리 검사** — 페이지 목적, MC 품질, 사이트/작성자 정보, E-E-A-T, 평판, 스팸/남용
- **YMYL 자동 감지** — 건강·금융·법률·안전 등 YMYL 주제를 자동 판별하고 가중치를 조정
- **증거 기반 채점** — 모든 점수는 콘텐츠에 명시적으로 존재하는 증거에 기반; 추론은 인정하지 않음
- **낙관 편향 방지** — 상한 규칙, 검증 불가 항목의 보수적 점수, Devil's Advocate 자기 검증으로 점수 부풀림 차단
- **실행 가능한 결과물** — 등급만이 아닌, 예상 점수 상승폭이 포함된 우선순위별 개선 계획 제공
- **유연한 입력** — URL(`web_fetch`로 가져옴) 또는 직접 붙여넣은 HTML/텍스트 모두 지원
- **부분 검사** — 특정 카테고리만 단독 실행 가능 (예: "E-E-A-T만 검사해줘")

## 개요

### 작동 방식

스킬은 3단계로 실행됩니다.

**Phase 1: Discovery**

1. 입력 수신 (URL 또는 붙여넣은 콘텐츠)
2. URL인 경우 `web_fetch`로 페이지 가져오기
3. 페이지 유형 판별 (정보성, 상거래, 블로그, 포럼, 엔터테인먼트, 정부/기관)
4. YMYL 해당 여부 판별 (명확한 YMYL / 중간 / 비YMYL)
5. YMYL 여부에 따라 가중치 확정

**Phase 2: Analysis** 6. 6개 카테고리 순차 검사 (A → B → C → D → E → F), 각 0~100점 채점

**Phase 3: Synthesis** 7. 가중 합산으로 종합 점수 산출 8. 즉시 D등급 트리거 확인 (유해 콘텐츠, 스팸 확정, 사기 평판) 9. 75점 이상 카테고리에 Devil's Advocate 자기 검증 수행 10. 최종 등급 부여 및 마크다운 보고서 생성

### 검사 카테고리

| 카테고리                | 가중치 | 검사 내용                                              |
| ----------------------- | ------ | ------------------------------------------------------ |
| A. 페이지 목적 & 유해성 | 15%    | 목적이 명확하고 유익한가, 기만적이거나 유해하지 않은가 |
| B. MC(주요 콘텐츠) 품질 | 25%    | 노력, 독창성, 재능/기술, 정확성, 적절한 분량           |
| C. 웹사이트/작성자 정보 | 10%    | 운영 주체 투명성, 작성자 자격, 연락처                  |
| D. E-E-A-T              | 25%    | 경험, 전문성, 권위성, 신뢰성                           |
| E. 평판                 | 10%    | 독립적 외부 출처, 리뷰, 뉴스 보도                      |
| F. 스팸/남용 탐지       | 15%    | 복사 콘텐츠, AI 대량 생성, 기만적 디자인, 광고 방해    |

YMYL 주제에서는 가중치가 조정됩니다 — 상세 내용은 [scoring-weights.md](skills/website-quality-checker/references/scoring-weights.md)를 참조하세요.

### 등급 체계

| 종합 점수 | 등급 | Google QRG 대응 |
| --------- | ---- | --------------- |
| 90~100    | S    | Highest         |
| 75~89     | A    | High            |
| 55~74     | B    | Medium          |
| 35~54     | C    | Low             |
| 0~34      | D    | Lowest          |

아래 조건 중 하나라도 해당되면 점수와 무관하게 **즉시 D등급**이 부여됩니다:

- 카테고리 A = 0 (해롭거나 기만적 목적)
- 카테고리 C = 0 + YMYL (YMYL인데 사이트/작성자 정보 전무)
- 카테고리 E ≤ 10 (사기·범죄 관련 매우 부정적 평판)
- 카테고리 F = 0 (스팸 확정)

### 채점 원칙

1. **증거 기반** — 콘텐츠에 명시적으로 존재하는 증거에 기반하여 채점. "전문가인 것 같다"는 증거가 아님.
2. **검증 불가 = 보수적 점수** — 확인할 수 없는 항목은 50점이 아닌 35~40점. "확인 안 했으니 괜찮다"가 아니라 "확인할 수 없으니 신뢰할 수 없다."
3. **상한 규칙** — ❌가 1개면 해당 카테고리 상한 70점, 2개 이상이면 상한 50점.
4. **Devil's Advocate 자기 검증** — 6개 카테고리 채점 후, 75점 이상인 카테고리의 증거를 재검토. 증거 불충분 시 60점 이하로 하향.

## 플러그인 구조

```
website-quality-checker/
├── .claude-plugin/
│   └── plugin.json                          # 플러그인 매니페스트
├── skills/
│   └── website-quality-checker/
│       ├── SKILL.md                         # 스킬 정의 (3단계 워크플로우)
│       └── references/                      # 검사 중 필요 시 로드
│           ├── google-qrg-summary.md        # QRG 핵심: YMYL, MC 품질, 평판
│           ├── eeat-criteria.md             # E-E-A-T 평가 질문
│           ├── lowest-quality-signals.md    # 즉시 Lowest 트리거
│           └── scoring-weights.md           # 가중치 표 & 점수 산출 공식
├── README.md                                # English
├── README_KO.md                             # 한국어
├── CHANGELOG.md
└── LICENSE
```

**Progressive Disclosure**: 스킬 본체(`SKILL.md`)는 500줄 이내로 유지합니다. 세부 기준은 `references/`에 분리되어 있으며, 해당 카테고리 검사 시에만 로드되어 컨텍스트 사용을 효율적으로 관리합니다.

## 사용법

Claude Code에 자연어로 요청하세요:

```
"https://example.com 품질 검사해줘"
"이 사이트 신뢰할 수 있어?"
"이 URL 페이지 품질 분석해줘"
"이 페이지 E-E-A-T 평가해줘"
"스팸 사이트인지 확인해줘"
"이 페이지 MC 품질이랑 스팸 여부만 체크해줘"
```

HTML이나 텍스트를 직접 붙여넣을 수도 있습니다:

```
"이 콘텐츠 품질 평가해줘: [텍스트 붙여넣기]"
```

URL이 없는 경우 평판 카테고리(E)는 "확인 불가"로 처리되어 보수적 점수(35점)가 부여됩니다.

### 부분 검사

특정 카테고리만 요청할 수 있습니다:

- "E-E-A-T만 검사해줘" → 카테고리 D만 실행
- "스팸인지 확인해줘" → 카테고리 F만 실행
- "이 사이트 신뢰할 수 있어?" → 카테고리 C + D + E 실행

부분 검사 시 실행한 카테고리만 보고서에 포함되며, 종합 점수는 "부분 검사"로 표기됩니다.

## 출력

마크다운 보고서가 생성됩니다:

### 보고서 구성

1. **헤더** — 검사 대상 URL, 검사 일시, 페이지 유형, YMYL 해당 여부
2. **종합 등급** — 등급(S/A/B/C/D), 100점 만점 점수, 시각적 프로그레스 바
3. **점수 총괄 표** — 6개 카테고리별 가중치, 점수, 가중 기여분, 상태 라벨
4. **카테고리별 상세** — 항목별 ✅/⚠️/❌ 판정과 콘텐츠 내 증거 인용
5. **개선 실행 계획** — 3단계 우선순위:
   - 즉시 수정 (높은 효과, 낮은 난이도)
   - 중기 개선 (높은 효과, 중간 난이도)
   - 전략적 개선 (장기 효과)
6. **예상 개선 효과** — 모든 권고 적용 시 카테고리별 전후 점수 추정
7. **Devil's Advocate 검증 로그** — 자기 검증 중 조정된 점수 기록
8. **핵심 결론** — 강점 1개 + 약점 1~2개, 3문장 이내 압축 요약

### 등급 출력 예시

```
## 종합 등급: B (68/100)

0         20        40        60        80        100
|---------|---------|---------|---------|---------|
██████████████████████████████████░░░░░░░░░░░░░░░░░ 68/100
                                  ^
```

## 설치

```bash
git clone https://github.com/llqqssttyy/website-quality-checker.git
```

## 참조 문서

검사 중 필요 시 로드되는 참조 문서:

- [google-qrg-summary.md](skills/website-quality-checker/references/google-qrg-summary.md) — YMYL 판별, MC 품질 기준, 평판 기준
- [eeat-criteria.md](skills/website-quality-checker/references/eeat-criteria.md) — E-E-A-T 차원별 세부 판별 질문
- [lowest-quality-signals.md](skills/website-quality-checker/references/lowest-quality-signals.md) — 즉시 Lowest 판정 트리거 (유해, 비신뢰, 스팸)
- [scoring-weights.md](skills/website-quality-checker/references/scoring-weights.md) — 가중치 표, YMYL 보정, 점수 산출 공식

## 라이선스

[MIT](LICENSE)
