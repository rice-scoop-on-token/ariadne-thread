# 연구 근거 (research-references.md)

본 문서는 설계 판단의 근거가 된 연구 결과만 기록한다. 결과를 어떤 설계 결정으로 옮겼는지는 `docs/ux-flow.md`, `docs/architecture.md`에 기술하며, 본 문서는 설계 결정을 포함하지 않는다.

설계가 변경되어도 본 문서는 변경되지 않는다. 연구 결과 자체는 설계와 독립적이기 때문이다.

---

## 1. 발산 기법과 시각화의 관계

**브레인스토밍과 브레인스케칭을 비교한 실험에서, 브레인스토밍은 아이디어 개수가 유의하게 많았고 브레인스케칭은 이전 아이디어와의 연결이 유의하게 많았다는 결과가 있다.** 글을 스케치로 대체하면 아이디어 생성 효율이 떨어졌고, 기존 브레인스토밍에 스케치를 추가한 조건은 개수를 유지하면서 아이디어 간 연결을 늘렸다.

> Van der Lugt, R. (2002). Brainsketching and How it Differs from Brainstorming. *Creativity and Innovation Management*, 11(1), 43–54.

- Wiley (페이월): https://onlinelibrary.wiley.com/doi/abs/10.1111/1467-8691.00235
- 무료 버전: https://www.academia.edu/66383809/Including_Sketching_in_Design_Idea_Generation_Meetings

---

## 2. 스케치의 기능 중 실제로 지지된 것

**스케치의 세 가지 기능 가설 중, 개인 사고 과정에서의 재해석과 이전 아이디어에 대한 접근성은 지지되었으나 타인의 아이디어를 재해석하는 기능은 지지되지 않았다는 결과가 있다.** 참가자들은 스케치하는 동안 그룹 논의에서 소외되는 느낌을 보고했다.

> Van der Lugt, R. (2005). How sketching can affect the idea generation process in design group meetings. *Design Studies*, 26(2), 101–126.

- ScienceDirect (페이월): https://www.sciencedirect.com/science/article/abs/pii/S0142694X04000778
- ResearchGate: https://www.researchgate.net/publication/222577049_How_sketching_can_affect_the_idea_generation_process_in_design_group_meetings

---

## 3. 설계 고착 (Design Fixation)

**설계 문제와 함께 예시 해법을 제시하면, 결함을 명시적으로 지적해도 새 설계가 그 예시의 특징을 물려받는다는 결과가 있다.** 시각장애인용 조리 계량 도구와 자전거 거치대 과제를 사용한 실험에서, 제시된 결함 있는 예시는 생성된 설계 수 자체에는 영향을 주지 않았으나 전문가와 초보 모두가 그 결함 있는 설계를 자신의 개념 모델로 사용했다.

후속 연구에서, 고착의 부정적 효과를 보고한 선행 연구들이 대체로 시각적·회화적 예시를 사용했으며, 동일 내용을 언어보다 시각 형태로 제시하면 대상의 전형적 기능이 더 두드러져 관습적인 아이디어가 나왔다는 결과가 보고되었다.

> Jansson, D. G., & Smith, S. M. (1991). Design fixation. *Design Studies*, 12(1), 3–11.

- DOI: `10.1016/0142-694X(91)90003-F` (출판사 직링크 미확인, DOI resolve 권장)
- 서지 레코드: https://scholarhub.vn/publication/Design-fixation/7e0675c7-19b8-42c9-96c3-7bcb6e636812
- 실험 패러다임 재서술 무료 PDF: https://dalyresearch.engin.umich.edu/wp-content/uploads/sites/237/2020/12/Leahy_Daly_McKilligan_Seifert-Design-Fixation-From-Initial-Examples-Provided-Versus-Self-Generated-Ideas.pdf

---

## 4. 생성형 AI가 집단 다양성에 미치는 영향

**LLM이 생성한 아이디어를 제공받은 조건은 개별 결과물이 더 창의적이고 잘 쓰였다는 평가를 받았으나, 결과물들이 서로 더 유사해졌다는 결과가 있다.** 효과는 원래 창의성이 낮은 작가에게 특히 컸다. 저자들은 이를 개인은 이득이나 집단은 손실인 사회적 딜레마 구조로 규정했다.

> Doshi, A. R., & Hauser, O. P. (2024). Generative AI enhances individual creativity but reduces the collective diversity of novel content. *Science Advances*, 10(28), eadn5290.

- 본문 (오픈액세스): https://www.science.org/doi/10.1126/sciadv.adn5290
- PDF: https://www.science.org/doi/pdf/10.1126/sciadv.adn5290
- PubMed: https://pubmed.ncbi.nlm.nih.gov/38996021/
- 데이터셋: https://datadryad.org/dataset/doi:10.5061/dryad.qfttdz0pm

---

## 5. 유사도 맵 기반 예시 선별과 아이디어 다양성

**아이디어를 유사도 기반 공간 맵에 배치하고 그 맵으로 서로 다양한 예시를 선별해 제시한 조건이, 무작위 예시를 제시한 조건보다 참가자가 생성한 아이디어의 다양성을 높였다는 결과가 있다.**

또한 비전문가의 단순 유사도 비교 판단을 다차원 척도법과 능동적 유사도 학습으로 집계해 만든 맵에 대해, **사람 평가자들이 맵에서 도출된 비유사도에 동의하는 정도가 평가자들끼리 서로 동의하는 정도와 같거나 그 이상이었다는 검증 결과가 있다.**

> Siangliulue, P., Arnold, K. C., Gajos, K. Z., & Dow, S. P. (2015). Toward Collaborative Ideation at Scale. *CSCW*.

- 무료: https://www.eecs.harvard.edu/~kgajos/papers/2015/siangliulue15toward.shtml

---

## 6. 배치 행동의 부산물로 의미 정보를 수집하는 방식

**아이디에이션 과정에 자연스럽게 통합된 활동(아이디어를 끌어다 놓는 배치)의 부산물로 커뮤니티 구성원이 직접 의미 정보를 제공하는 유기적 인간 계산 방식이 구현되었다.** 별도 태깅 작업이나 외부 크라우드 작업자에 의존하지 않으며, 부차 과제에서 암묵적으로 수집한 판단이 외부 작업자의 명시적 판단으로 만든 의미 모델과 최소한 동등한 정확도를 내는지가 핵심 검증 질문이었다.

> Siangliulue, P., Chan, J., Dow, S. P., & Gajos, K. Z. (2016). IdeaHound: Improving Large-scale Collaborative Ideation with Crowd-Powered Real-time Semantic Modeling. *UIST*, 609–624.

- 저자 페이지: http://www.eecs.harvard.edu/~kgajos/papers/2016/siangliulue16ideahound.shtml
- 전문 PDF (Harvard DASH): https://dash.harvard.edu/server/api/core/bitstreams/7312037d-f448-6bd4-e053-0100007fdf3b/content

---

## 7. 라운드 분할 구조

**시각적 브레인스토밍 기법들이 공통적으로 짧은 라운드 구조를 사용한다는 정리가 있다.** 6-3-5는 참가자 6명이 5분마다 시트를 옆으로 넘기며 라운드마다 3개의 스케치를 생산하는 구조이며, Gallery Method와 Braindrawing도 수 분 단위 라운드를 반복한다.

> Visual Thinking Styles and Idea Generation Strategies (ERIC, EJ1137716)

- 무료 PDF: https://files.eric.ed.gov/fulltext/EJ1137716.pdf

---

## 8. 명목 그룹과 상호작용 그룹의 성과 격차

**상호작용 그룹이 명목 그룹(각자 따로 낸 결과를 합친 것)보다 성과가 낮은 격차가 반복적으로 관찰되었으며, 구체적 지시와 목표 제시가 그 격차를 축소했다는 결과가 있다.** 개수 목표의 효과는 명확했으나 다른 종류 목표의 근거는 상대적으로 불명확하다고 정리된다.

> Paulus, P. B., & Kenworthy, J. B. Effective Brainstorming. In *Handbook of Group Creativity and Innovation*.

- https://www.researchgate.net/publication/326127810_Effective_Brainstorming

---

## 9. 휴리스틱 카드의 고착 완화 효과

**휴리스틱을 이름과 설명, 그림이 담긴 카드 형태로 명시적으로 제공하는 방식이 아이디어 생성 향상과 고착 극복에 효과를 보였다는 결과가 있다.** 카드는 앞면에 휴리스틱 명칭·설명·그림을, 뒷면에 적용 제품 예시 두 개를 배치하는 형식이다.

> Overcoming Design Fixation in Idea Generation. Design Research Society.

- 무료 PDF: https://dl.designresearchsociety.org/cgi/viewcontent.cgi?article=1716&context=drs-conference-papers

---

## 10. 아이디에이션 효과 측정 지표

**아이디어 발상 기법의 효과를 비교 평가하기 위한 정량 지표 체계(quantity, variety, novelty, quality)가 제안되었고, 이후 ideation 연구의 표준 측정 프레임으로 사용된다.**

> Shah, J. J., Smith, S. M., & Vargas-Hernandez, N. (2003). Metrics for measuring ideation effectiveness. *Design Studies*, 24(2), 111–134.

- DOI: `10.1016/S0142-694X(02)00034-0` (페이월)

**주의** — 본 항목은 초록 원문을 확보하지 못한 상태에서 통용되는 내용으로 기술했다. 외부 문서에 인용할 경우 원문 확인이 필요하다.

---

## 11. 문제 재표현을 통한 유추 탐색

**설계 문제를 언어적으로 재표현해 예상 밖의 유추와 유사 영역으로 유도하는 WordTree 기법을 사용한 설계자가 더 많은 유추를 식별했다는 통제 실험 결과가 있다.** 상위에 일반 개념, 하위에 구체 개념을 배치하는 계층 구조를 사용한다.

> Linsey, J. S., Markman, A. B., & Wood, K. L. (2012). Design by Analogy: A Study of the WordTree Method for Problem Re-Representation. *Journal of Mechanical Design*, 134(4), 041009.

- DOI: `10.1115/1.4006145` (페이월)
- 무료 선행 버전 (2008 ASEE): https://peer.asee.org/3974
