# GitHub 설정

브랜치 룰셋 정의다. GitHub가 자동으로 읽지 않으며, 아래 명령으로 직접 적용한다.

## 적용

\`\`\`powershell
$REPO = "rice-scoop-on-token/ariadne-thread"

gh api --method POST -H "Accept: application/vnd.github+json" /repos/$REPO/rulesets --input infra/github/ruleset-main.json
gh api --method POST -H "Accept: application/vnd.github+json" /repos/$REPO/rulesets --input infra/github/ruleset-develop.json
gh api --method POST -H "Accept: application/vnd.github+json" /repos/$REPO/rulesets --input infra/github/ruleset-naming.json
\`\`\`

## 확인

\`\`\`powershell
gh api /repos/rice-scoop-on-token/ariadne-thread/rulesets
\`\`\`

## 수정

룰셋은 생성 후 ID로 갱신한다. 같은 이름으로 POST하면 중복 생성된다.

\`\`\`powershell
gh api --method PUT -H "Accept: application/vnd.github+json" /repos/rice-scoop-on-token/ariadne-thread/rulesets/{id} --input infra/github/ruleset-main.json
\`\`\`
