---
description: |
  This workflow creates daily repo status reports. It gathers recent repository
  activity (issues, PRs, discussions, releases, code changes) and generates
  engaging GitHub issues with productivity insights, community highlights,
  and project recommendations.

on:
  schedule: daily
  workflow_dispatch:

permissions:
  contents: read
  issues: read
  pull-requests: read

network: defaults

tools:
  github:
    
---
description: |
  This workflow creates daily repo status reports using standard GitHub Actions.
  It gathers recent repository activity (issues, PRs, discussions, releases, code changes)
  and creates a GitHub issue with a summary. No AI/Copilot required - works with free tier.

on:
  schedule:
    - cron: "39 16 * * *"  # Daily at 4:39 PM UTC
  workflow_dispatch:

permissions:
  contents: read
  issues: write
  pull-requests: read

jobs:
  repo-status:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Get repo stats
        id: stats
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            // Get recent activity
            const issues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'all',
              per_page: 10,
              sort: 'updated',
              direction: 'desc'
            });

            const pullRequests = await github.rest.pulls.list({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'all',
              per_page: 10,
              sort: 'updated',
              direction: 'desc'
            });

            const openIssues = issues.data.filter(i => i.state === 'open').length;
            const openPRs = pullRequests.data.filter(pr => pr.state === 'open').length;
            const recentIssues = issues.data.filter(i => {
              const created = new Date(i.created_at);
              const oneWeekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
              return created > oneWeekAgo;
            }).length;

            core.setOutput('open_issues', openIssues);
            core.setOutput('open_prs', openPRs);
            core.setOutput('recent_issues', recentIssues);

      - name: Create repo status issue
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const date = new Date().toLocaleDateString('en-US', {
              weekday: 'long',
              year: 'numeric',
              month: 'long',
              day: 'numeric'
            });

            const body = `## 📊 Daily Repository Status - ${date}

### Repository Metrics
- **Open Issues**: ${{ steps.stats.outputs.open_issues }}
- **Open Pull Requests**: ${{ steps.stats.outputs.open_prs }}
- **Issues Created This Week**: ${{ steps.stats.outputs.recent_issues }}

### Quick Links
- [Issues](https://github.com/${{ github.repository }}/issues)
- [Pull Requests](https://github.com/${{ github.repository }}/pulls)
- [Repository](https://github.com/${{ github.repository }})

---
*Generated automatically by GitHub Actions - Free Tier Edition*`;

            // Close older status issues
            const issues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'open',
              labels: 'daily-status'
            });

            for (const issue of issues.data) {
              if (issue.title.includes('[repo-status]')) {
                await github.rest.issues.update({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  issue_number: issue.number,
                  state: 'closed'
                });
              }
            }

            // Create new status issue
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `[repo-status] Daily Report - ${date}`,
              body: body,
              labels: ['report', 'daily-status']
            });

            console.log('✅ Repository status report created successfully!');
