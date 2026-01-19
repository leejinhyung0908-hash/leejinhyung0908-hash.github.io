---
title: "ESG 중대성 평가(Materiality Assessment) AI 모델 구축 - PM 관점의 칸반보드와 기술 구현"
excerpt: "LangGraph 스타 토폴로지 구조를 활용한 ESG 중대성 평가 AI 시스템의 3주 개발 계획과 상세 구현 방안을 정리합니다."
date: 2026-01-19
categories:
  - ESG
  - AI
tags:
  - ESG
  - Materiality Assessment
  - LangGraph
  - KoElectra
  - EXAONE
  - Next.js
  - PostgreSQL
  - pgvector
  - RAG
toc: true
toc_sticky: true
---

ESG 중대성 평가(Materiality Assessment) AI 모델 구축을 위한 **PM 관점의 상세 칸반보드**와 **기술적 구현 방안**을 정리합니다.

제시된 **스타 토폴로지(Star Topology)** 구조는 중앙 제어 노드가 각 전문 모델(EXAONE, KoElectra)에 업무를 할당하고 결과를 취합하는 데 최적화된 선택입니다.

---

## 📊 3주 집중 개발 칸반보드

### Week 1: 데이터 아키텍처 및 RAG 기반 구축

#### To-Do
- **ESG 지표 DB 설계:** 5대 핵심 주제 및 산출 지표(온실가스 배출량, 윤리경영 제보 등) 테이블 설계
- **pgvector 환경 구축:** 과거 지속가능경영보고서 및 업계 벤치마크 데이터 벡터화 및 저장
- **Next.js 초기 설정:** App Router 기반 프로젝트 구조 및 API Route(LangGraph 통신용) 설정

#### Implementation
- `PostgreSQL`에 이슈별 영향도(High/Mid/Low)와 발생 가능성을 매핑하는 가중치 테이블 생성

---

### Week 2: LangGraph 기반 AI 워크플로우 개발

#### To-Do
- **KoElectra 노드 구현:** 사용자 입력(뉴스, 공시, 내부 데이터)을 ESG 5대 주제로 자동 분류하는 NLU 모델 서빙
- **EXAONE 노드 구현:** 분류된 데이터를 바탕으로 영향 측정 지표를 분석하고 정성적 근거를 생성하는 LLM 추론 로직 구현
- **스타 토폴로지 라우팅:** 중앙 노드(`Supervisor`)가 KoElectra의 분류 결과를 보고 EXAONE에게 특정 분석을 요청하는 로직 완성

#### Implementation
- LangGraph의 `State` 객체에 평가 점수와 근거 문장을 실시간 업데이트하며 노드 간 데이터 동기화

---

### Week 3: 프론트엔드 통합 및 서비스 최적화

#### To-Do
- **Next.js 대시보드:** 중대성 평가 결과를 'Materiality Matrix'(X축: 비즈니스 영향, Y축: 사회/환경 영향)로 시각화
- **검증 및 튜닝:** 실제 데이터 투입 후 평가 산식(High/Mid/Low)과 AI 판정 결과 일치 여부 테스트
- **최종 배포:** 리포트 생성 기능(PDF/JSON) 및 히스토리 관리 기능 추가

#### Implementation
- `Recharts` 또는 `Canvas`를 활용하여 ESG 이슈별 위치를 매트릭스 상에 동적으로 렌더링

---

## 🛠️ 상세 구현 방안

### 1. LangGraph 스타 토폴로지 구조

중앙의 **Router 노드**가 상태를 관리하며 두 모델의 강점을 극대화합니다.

```
         ┌─────────────┐
         │   Router    │
         │ (Supervisor)│
         └──────┬──────┘
                │
        ┌───────┴───────┐
        ▼               ▼
┌───────────────┐ ┌───────────────┐
│   KoElectra   │ │    EXAONE     │
│ Classification│ │   Reasoning   │
└───────────────┘ └───────────────┘
```

- **KoElectra (Classification Node):** 한국어 금융/기업 용어에 최적화된 모델이므로, 대량의 텍스트를 분석해 해당 이슈가 **환경(기후변화)**인지 **사회(인권/안전)**인지 빠르게 분류합니다.
- **EXAONE (Reasoning Node):** 분류된 카테고리에 따라 "사회적 비용 발생(Social Cost)"이나 "가구 소득 증감" 등 복잡한 정성적 가치를 추론하고 최종 등급을 산출합니다.

### 2. 평가 산식의 로직화

평가 체계를 코드화하여 AI가 판단 근거로 삼게 합니다.

#### 입력 데이터
기업의 활동 데이터 (예: 연간 온실가스 배출량, 안전사고 건수)

#### AI 판단 로직

```python
def evaluate_materiality(category: str, metrics: dict) -> str:
    """ESG 중대성 평가 로직"""
    
    if category == "환경":
        # 온실가스 감축량에 따른 '환경 가치 증감' 계산
        score = calculate_environmental_value(metrics)
    elif category == "사회":
        # 고용 창출 및 교육 이수자에 따른 '사회적 가치' 측정
        score = calculate_social_value(metrics)
    
    # 결과 매핑: 임계값 기반 자동 레이블링
    if score >= threshold_high:
        return "High(●●●)"
    elif score >= threshold_mid:
        return "Mid(●●○)"
    else:
        return "Low(●○○)"
```

### 3. PostgreSQL + pgvector 활용 (RAG)

단순한 AI 추론을 넘어, 과거의 평가 이력을 참고하여 일관성을 유지합니다.

#### Semantic Search
새로운 ESG 이슈가 발생했을 때, `pgvector`를 통해 과거 유사했던 이슈의 처리 결과와 등급을 검색하여 EXAONE에게 컨텍스트로 제공합니다.

```sql
-- 유사 이슈 검색 쿼리 예시
SELECT issue_title, rating, reasoning
FROM esg_assessments
ORDER BY embedding <=> $1  -- 코사인 유사도 기반
LIMIT 5;
```

#### Grounding
AI가 임의로 판단하는 것이 아니라, DB에 저장된 **실제 산출 지표**를 기준으로 답변하게 하여 할루시네이션(환각)을 방지합니다.

### 4. Next.js 화면 구성

| 구성 요소 | 설명 |
|----------|------|
| **입력 UI** | 기업의 주요 이슈나 성과 지표를 입력하는 폼 |
| **분석 과정 시각화** | LangGraph의 노드 전환 상태를 실시간으로 보여주어 신뢰도 확보 |
| **결과 대시보드** | 5대 주제별 중대성 등급을 테이블과 매트릭스로 표기 |

#### Materiality Matrix 시각화 예시

```tsx
// Recharts를 활용한 매트릭스 컴포넌트
import { ScatterChart, Scatter, XAxis, YAxis, Tooltip } from 'recharts';

const MaterialityMatrix = ({ data }) => (
  <ScatterChart width={600} height={400}>
    <XAxis 
      dataKey="businessImpact" 
      name="비즈니스 영향" 
      domain={[0, 10]} 
    />
    <YAxis 
      dataKey="socialImpact" 
      name="사회/환경 영향" 
      domain={[0, 10]} 
    />
    <Tooltip cursor={{ strokeDasharray: '3 3' }} />
    <Scatter name="ESG Issues" data={data} fill="#8884d8" />
  </ScatterChart>
);
```

---

## 💡 PM의 핵심 제언

3주라는 기간 내에 성공하기 위해서는 **데이터 정제**를 개발과 동시에 진행해야 합니다.

특히 '산출 지표'들을 AI가 바로 계산할 수 있도록 **수치화된 프롬프트 템플릿**을 미리 정의하는 것이 성공의 열쇠입니다.

### 핵심 성공 요인

1. **데이터 품질:** ESG 지표의 정확한 수치화 및 표준화
2. **모델 역할 분담:** KoElectra(분류) + EXAONE(추론)의 명확한 책임 분리
3. **RAG 활용:** 과거 평가 이력 기반의 일관된 판단
4. **실시간 피드백:** LangGraph 상태 시각화를 통한 투명성 확보

---

*다음 포스트에서는 실제 LangGraph 구현 코드와 함께 더 상세한 기술 구현 내용을 다루겠습니다.*

