---
title: "순수 LLM 코드 리뷰가 놓치는 것: SAST와 결합해 false positive를 91% 줄이는 하이브리드 파이프라인"
date: "2026-08-12"
keywords: ["AI 코드 리뷰", "SAST", "CodeRabbit", "Semgrep", "false positive", "GitHub Copilot code review", "정적 분석", "CodeQL"]
lang: "ko"
description: "LLM 단독 리뷰는 환각이 있고 SAST 단독은 false positive가 높다. 두 도구를 계층화해 결합하면 false positive를 91% 줄이면서 시맨틱 리뷰 품질을 유지할 수 있다. 실제 GitHub Actions 파이프라인 구현과 CodeRabbit·Copilot 비교까지."
---

# 순수 LLM 코드 리뷰가 놓치는 것: SAST와 결합해 false positive를 91% 줄이는 하이브리드 파이프라인

AI 코드 리뷰 도구가 2026년에 일상이 됐다. GitHub Copilot Code Review가 모든 유료 Copilot 플랜에 기본 탑재되고, CodeRabbit은 퍼블릭 리포지토리에 한해 무료 리뷰를 제공한다. PR이 열리면 몇 초 만에 봇이 코멘트를 다는 시대다.

하지만 실제 프로덕션에 적용해보면 두 가지 극단이 반복된다. **LLM 단독 리뷰**는 문맥을 잘 이해하지만 자신감 높은 헛소리(confident hallucination)를 남기고, **전통적 SAST 도구**는 결정론적 규칙으로 확실한 취약점을 잡아내지만 문맥을 몰라 false positive가 폭주한다. Semgrep의 자체 분석에 따르면, 전통적 SAST는 모든 finding 중 상당수가 실제로는 악용 불가능한 false positive로, 보안 팀의 트리아지(triage) 부담의 핵심 원인이다.

이 글에서는 두 접근을 **계층화된 하이브리드 파이프라인**으로 결합하는 설계를 다룬다. SAST로 결정론적 검사를 먼저 수행하고, LLM은 그 결과를 문맥과 함께 재해석해 false positive를 걸러내는 구조다. 실제 학술 검증(SAST-Genius, 91% FP 감소)과 프로덕션 도구(Semgrep Assistant Memories)의 데이터를 기반으로, 직접 구축할 수 있는 GitHub Actions 예시까지 제공한다.

## 왜 LLM만으로는 부족한가

LLM 기반 코드 리뷰의 강점은 분명하다. 비즈니스 로직 오류, 아키텍처 위반, 크로스 파일 영향 분석 등 전통적 정적 분석이 잡기 어려운 **시맨틱 수준의 문제**를 발견한다. 하지만 결정론적이지 않다는 근본적 한계가 있다.

첫째, **확신의 비대칭**이다. LLM은 70% 확신하는 문제와 99% 확신하는 문제를 동일한 톤으로 보고한다. 개발자가 "틀렸다"고 dismiss한 코멘트가 다음 PR에서 동일한 톤으로 다시 나타난다. CodeRabbit이나 GitHub Copilot Code Review 모두 confidence threshold를 내부적으로 사용하지만, 외부에서는 이 임계값을 세밀하게 조정하기 어렵다.

둘째, **규칙 기반 도구와의 중복·충돌**이다. ESLint나 Semgrep이 이미 잡는 문제(사용하지 않는 변수, 잘못된 import 순서)를 LLM이 다시 지적하면 노이즈만 늘어난다. QubitTool의 분석에 따르면, 순수 AI 리뷰의 두 가지 문제는 높은 비용과 결정론적 규칙에 대한 정밀도 부족이다.

셋째, **보안 취약점 탐지의 한계**다. LLM은 SQL 인젝션이나 XSS 패턴을 "인식"할 수 있지만, OWASP Top 10 전체를 일관되게 커버하거나 새로운 CVE 패턴을 추적하는 데는 전용 보안 데이터베이스에 기반한 SAST 도구가 훨씬 신뢰할 수 있다.

## 반대로, SAST만으로는 왜 부족한가

전통적 SAST(Semgrep, CodeQL, SonarQube 등)의 핵심 약점은 **문맥 무지(context blindness)**다. Semgrep 공식 블로그(2025년 6월)는 이를 정확히 짚는다: "모든 SAST 도구가 false positive를 생성하는 이유는, 악용 가능성(exploitability)이 조직 고유의 sanitizer, 사용자 노출 범위, 프레임워크 특유의 보호 메커니즘 등 **문맥에 의존**하기 때문이다."

예를 들어, 어떤 조직은 자체 입력 검증 라이브러리를 사용해 모든 외부 입력을 sanitize한다. Semgrep의 기본 규칙은 이 라이브러리를 모르기 때문에, 이미 안전하게 처리된 코드에서도 SQL 인젝션 경고를 띄운다. 결과는 보안 팀의 트리아지 큐가 finding으로 넘쳐나고, 개발자는 "또 틀린 경보"라며 무감각해진다.

Semgrep은 자체 데이터에서 **1,000개 이상의 배포, 45개 이상의 엔터프라이즈, 100만 개 이상의 finding 분석**을 기반으로, AI 트리아지 Assistant가 전체 트리아지 작업의 약 20%를 자동으로 처리한다고 밝혔다. 사용자는 Assistant의 결정에 95% 이상 동의했고, 보안 연구원은 96% 이상 동의했다. 한 Fortune 500 고객은 memory 2개를 추가했더니 기존 베이스라인 대비 false positive 감지율이 2.8배 개선됐다.

## 하이브리드 파이프라인: 결정론적 검사가 먼저, LLM은 재해석한다

핵심 설계 원칙은 단순하다. **결정론적 문제는 결정론적 도구로, 불확실한 시맨틱 문제는 LLM으로.** 둘을 순차가 아니라 계층(layer)으로 구성한다.

| 계층 | 도구 | 역할 | 특성 |
|------|------|------|------|
| L1 | ESLint, Prettier, Ruff | 스타일·포맷 | 제로 비용, 밀리초 단위 |
| L2 | Semgrep, SonarQube | 복잡도, 중복, 알려진 패턴 | 규칙 결정론성 높음 |
| L3 | Snyk, Trivy, CodeQL | 의존성 CVE, SAST | 전용 보안 KB 기반 |
| L4 | LLM (GPT-4o / Claude) | 비즈니스 로직, 아키텍처, 크로스 파일 영향 | 문맥 이해 |

L1-L3이 먼저 실행되고, 그 결과(diff + SAST findings)를 L4 LLM의 입력으로 전달한다. LLM은 빈 화면에서 코드를 리뷰하는 게 아니라, **이미 식별된 finding들의 문맥을 분석**한다. "이 Semgrep 경고는 조직의 자체 sanitizer 때문에 false positive다"라는 판단을 내릴 수 있는 것이다.

이 방식이 효과적이라는 학술 근거가 있다. 2025년 9월 arXiv에 발표된 **SAST-Genius** 연구(arXiv:2509.15433, Vaibhav Agrawal & Kiarash Ahi)는 LLM과 SAST를 결합한 하이브리드 프레임워크를 제안했다. 핵심 결과: Semgrep 단독으로 보고한 225개 finding 중 실제 유효한 것은 소수에 불과했으나, LLM 재해석을 거치자 false positive가 약 **91% 감소**해 약 20개 수준으로 떨어졌다. 연구진은 "LLM은 코드 분석과 패턴 인식에 뛰어나지만 불일치와 환각이 있고, SAST는 높은 false positive와 문맥 부족으로 고통받는다. 둘을 통합하면 서로의 약점을 상쇄한다"고 요약한다.

## 실전 구현: GitHub Actions 4계층 파이프라인

실제로 동작하는 파이프라인을 구성해보자. 핵심은 L1-L3의 결과를 L4 LLM에 컨텍스트로 전달하는 것이다.

```yaml
# .github/workflows/hybrid-code-review.yml
name: Hybrid Code Review Pipeline
on:
  pull_request:
    types: [opened, synchronize, reopened]
permissions:
  contents: read
  pull-requests: write

jobs:
  # L1-L3: 결정론적 검사 (빠름, 초 단위)
  static-and-security:
    runs-on: ubuntu-latest
    outputs:
      semgrep_findings: ${{ steps.semgrep.outputs.results }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # L1: 린트 (스타일은 LLM이 다루지 않음)
      - name: L1 - Lint
        run: npx eslint --format json -o lint.json . || true

      # L2: 정적 분석
      - name: L2 - Semgrep
        id: semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/default
          generate-sarif: true

      # L3: 보안 스캔 (SAST + 의존성)
      - name: L3 - CodeQL
        uses: github/codeql-action/analyze@v3

      - name: L3 - Trivy (dependency scan)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          severity: CRITICAL,HIGH

      # SAST 결과를 아티팩트로 저장 → L4 LLM 컨텍스트로 전달
      - name: Upload findings
        uses: actions/upload-artifact@v4
        with:
          name: security-findings
          path: |
            lint.json
            *.sarif

  # L4: AI 시맨틱 리뷰 (SAST 결과를 컨텍스트로 받음)
  ai-review:
    runs-on: ubuntu-latest
    needs: static-and-security
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Download SAST findings
        uses: actions/download-artifact@v4
        with:
          name: security-findings

      - name: Get PR diff
        id: diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD \
            --diff-filter=ACMR \
            > pr_diff.patch
          echo "size=$(wc -c < pr_diff.patch)" >> $GITHUB_OUTPUT

      - name: L4 - AI Semantic Review with SAST context
        if: steps.diff.outputs.size > 100
        uses: actions/github-script@v7
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        with:
          script: |
            const fs = require('fs');
            const diff = fs.readFileSync('pr_diff.patch', 'utf8');
            const sarif = fs.readFileSync('*.sarif', 'utf8');  // L2-L3 결과

            // LLM에게 diff와 SAST findings를 함께 전달
            const prompt = `
            ## PR Diff
            ${diff.substring(0, 30000)}

            ## SAST Findings (Semgrep + CodeQL)
            ${sarif.substring(0, 10000)}

            ## 작업
            1. 각 SAST finding의 실제 악용 가능성을 문맥과 함께 판단하라
            2. SAST가 잡지 못한 시맨틱 문제(로직 오류, 경쟁 조건, 아키텍처 위반)를 식별하라
            3. confidence >= 0.8 인 이슈만 보고하라
            4. 스타일/포맷 문제는 이미 L1에서 처리했으므로 보고하지 마라
            `;
            // ... LLM 호출 및 PR 코멘트 게시
```

이 구조의 핵심은 **L4 LLM이 빈 화면에서 리뷰하지 않는다**는 점이다. SAST가 이미 식별한 finding들을 받아, 각각의 실제 악용 가능성을 문맥과 함께 재판단한다. 이것이 SAST-Genius가 보여준 91% FP 감소의 메커니즘이다.

## 도구 선택: 직접 구축 vs CodeRabbit vs Copilot

하이브리드 파이프라인을 매번 직접 구축할 필요는 없다. 기성 도구들도 이 계층 구조를 점점 더 내장하고 있다. 하지만 어디까지 자동화되고 어디서 직접 제어해야 하는지 알아야 한다.

**GitHub Copilot Code Review**는 모든 유료 Copilot 플랜(Copilot Business, Enterprise 등)에서 사용 가능하다. GitHub 공식 문서에 따르면, Lite(기본)와 Balanced 두 가지 리뷰 강도 수준을 제공하며, Balanced는 더 높은 추론 능력의 모델로 라우팅한다. 이 effort levels 기능은 2026년 8월 7일자로 정식 출반(generally available)되었다고 GitHub 블로그 챈지로그가 확인한다. 흥미로운 점은 **에이전트 기능(agentic capabilities)**이다: 전체 리포지토리 컨텍스트를 분석하고, 제안된 수정사항을 새 PR으로 자동 생성하는 클라우드 에이전트 기능이 퍼블릭 프리뷰로 제공된다. 다만 이 기능은 GitHub Actions 러너를 사용하며, 조직이 Actions를 비활성화하면 제한된 리뷰로 폴백한다.

**CodeRabbit**은 더 독립적인 접근이다. CodeRabbit 공식 가격 페이지에 따르면, Free 플랜($0/월)은 퍼블릭·프라이빗 리포지토리 모두에서 PR 요약 기능을 제공하고, Pro($24/월/사용자, 연간 청구)는 린터 및 SAST 도구 연동, Jira/Linear 통합, 커스텀 사전 병합 검사를 포함한다. Pro Plus($48/월/사용자)는 자동 수정(autofix), 병합 충돌 해결 등을 추가한다. 오픈소스 퍼블릭 리포지토리는 영구 무료다. CodeRabbit은 내부적으로 L1-L4 계층을 통합 제공하지만, 조직 고유의 false positive 패턴을 학습시키는 기능은 제한적이다.

**직접 구축의 가치**는 정확히 이 지점에 있다. SAST-Genius와 Semgrep Assistant Memories가 보여주듯, **조직 고유의 문맥(자체 sanitizer, 내부 프레임워크, 비즈니스 도메인 규칙)을 false positive 필터로 학습**시키는 것이 가장 큰 ROI다. CodeRabbit의 프롬프트를 조직 맥락에 맞게 미세 조정하는 것은 제한적이지만, 직접 구축한 L4 계층은 Semgrep Assistant Memories처럼 과거 트리아지 결정을 memory로 저장하고 재사용할 수 있다.

## false positive 제어: 파이프라인이 살아남는 조건

하이브리드 파이프라인이 프로덕션에서 살아남느냐 죽느냐는 false positive 제어에 달려 있다. 세 가지 실전 원칙이 있다.

**원칙 1: 심각도에 따른 단계적 대응.** 모든 이슈가 병합을 막아서는 안 된다. 보안 취약점(CRITICAL/HIGH)은 hard block, 성능 회귀는 soft warning, 스타일 제안은 informational comment로 분리한다. LLM이 보고하는 이슈에도 confidence threshold(예: 0.8 이상만 PR 코멘트)를 적용해야 한다.

**원칙 2: 피드백 루프.** 개발자가 AI 코멘트를 dismiss하면 그 정보를 다음 리뷰의 컨텍스트로 사용해야 한다. Semgrep Assistant Memories의 Fortune 500 사례에서 memory 2개만으로 FP 감지율이 2.8배 개선된 것은, **피드백을 학습 메커니즘으로 설계**했기 때문이다. 직접 구축 시에는 dismiss 이벤트를 DB에 저장하고, 동일한 패턴의 코드에서 재발생을 차단하는 간단한 룰 베이스를 추가하는 것만으로도 큰 효과가 있다.

**원칙 3: diff 슬라이싱으로 비용 통제.** 전체 코드베이스가 아니라 PR의 증분 변경만 LLM에 전달한다. QubitTool의 분석에 따르면 diff 슬라이싱, 증분 리뷰, 모델 티어링(간단한 검사는 저비용 모델, 복잡한 검사만 고비용 모델)을 조합하면 PR당 토큰 비용을 효과적으로 통제할 수 있다.

## 핵심 요약

- **LLM과 SAST는 경쟁이 아니라 보완 관계다.** 결정론적 검사는 SAST로, 시맨틱 판단은 LLM으로. SAST-Genius 연구(arXiv:2509.15433)에서 이 조합은 false positive를 91% 줄였다(225건 → 20건).
- **계층 구조가 핵심이다.** L1 린트 → L2 정적 분석 → L3 보안 스캔 → L4 AI 시맨틱 리뷰. L4는 빈 화면에서 시작하지 않고 L1-L3의 결과를 컨텍스트로 받는다.
- **조직 문맥이 false positive를 결정한다.** Semgrep Assistant Memories의 데이터(100만+ finding 분석, 95%+ 사용자 동의율)가 증명하듯, 개발자 피드백을 학습 메커니즘으로 설계하는 것이 핵심이다.
- **기성 도구의 한계를 알아야 한다.** CodeRabbit(Pro $24/월)과 GitHub Copilot Code Review는 훌륭한 베이스라인이지만, 조직 고유의 false positive 패턴 학습은 직접 구축한 계층에서만 완전히 제어할 수 있다.
- **첫 단계 제안:** 이미 Semgrep이나 CodeQL을 CI에 달아놓았다면, 그 SARIF 결과를 LLM 프롬프트에 컨텍스트로 전달하는 간단한 스크립트부터 시작하라. 전체 파이프라인을 한 번에 구축할 필요 없이, L3→L4 컨텍스트 전달만으로 false positive 감소 효과를 즉시 체감할 수 있다.
