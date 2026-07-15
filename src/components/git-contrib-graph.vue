<script setup lang="ts">
import {
    ref, computed,
    // onMounted
} from 'vue'

interface ContributionDay {
    date: string
    count: number
    level: number
    details: ContributionDetail[]
}

interface ContributionDetail {
    type: 'commit' | 'pr' | 'issue' | 'review' | 'merge_request'
    title: string
    repo: string
    url: string
    source: 'github' | 'gitlab'
}

// interface GitHubContribution {
//     date: string
//     contributionCount: number
// }

// interface GitLabCommit {
//     committed_date: string
// }

const githubToken = ref(import.meta.env.VITE_GITHUB_TOKEN ?? '')
const gitlabToken = ref(import.meta.env.VITE_GITLAB_TOKEN ?? '')
const gitlabUsername = ref(import.meta.env.VITE_GITLAB_USERNAME ?? "Wabtey")

const githubEnabled = ref(true)
const gitlabEnabled = ref(true)

const selectedYear = ref<string>('last-year')
const contributions = ref<ContributionDay[]>([])
const loading = ref(false)
const error = ref('')
const selectedDay = ref<ContributionDay | null>(null)
const showModal = ref(false)

const availableYears = computed(() => {
    const currentYear = new Date().getFullYear()
    const years = [{ value: 'last-year', label: 'Last Year from Today' }]

    for (let year = currentYear; year >= 2020; year--) {
        years.push({ value: year.toString(), label: year.toString() })
    }

    return years
})

/* ------------------------------ Month labels ------------------------------ */

const weeks = computed(() => {
    const grouped: ContributionDay[][] = []
    let currentWeek: ContributionDay[] = []

    contributions.value.forEach((day, _) => {
        currentWeek.push(day)
        if (currentWeek.length === 7) {
            grouped.push([...currentWeek])
            currentWeek = []
        }
    })

    if (currentWeek.length > 0) {
        grouped.push(currentWeek)
    }

    return grouped
})

const getLevelColor = (level: number): string => {
    const colors = [
        '#161b22',
        '#0e4429',
        '#006d32',
        '#26a641',
        '#39d353'
        // '#ebedf0',
        // '#9be9a8',
        // '#40c463',
        // '#30a14e',
        // '#216e39'
    ]
    return colors[Math.min(level, 4)] ?? '#161b22' // '#ebedf0'
}

/* ---------------------------------- Query --------------------------------- */

const fetchGitHubContributions = async (): Promise<Map<string, ContributionDetail[]>> => {
    const query = `
    query {
      viewer {
        contributionsCollection {
          commitContributionsByRepository {
            repository {
              name
              isPrivate
              owner {
                login
              }
            }
            contributions(first: 100) {
              nodes {
                commitCount
                occurredAt
                commitMessages: url
              }
            }
          }
          pullRequestContributionsByRepository {
            repository {
              name
              isPrivate
              owner {
                login
              }
            }
            contributions(first: 100) {
              nodes {
                pullRequest {
                  title
                  url
                  createdAt
                }
              }
            }
          }
          issueContributionsByRepository {
            repository {
              name
              isPrivate
              owner {
                login
              }
            }
            contributions(first: 100) {
              nodes {
                issue {
                  title
                  url
                  createdAt
                }
              }
            }
          }
          pullRequestReviewContributionsByRepository {
            repository {
              name
              isPrivate
              owner {
                login
              }
            }
            contributions(first: 100) {
              nodes {
                pullRequest {
                  title
                  url
                }
                occurredAt
              }
            }
          }
        }
      }
    }
  `

    const response = await fetch('https://api.github.com/graphql', {
        method: 'POST',
        headers: {
            'Authorization': `bearer ${githubToken.value}`,
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({ query })
    })

    if (!response.ok) {
        throw new Error(`GitHub API error: ${response.statusText}`)
    }

    const data = await response.json()
    const detailsMap = new Map<string, ContributionDetail[]>()
    const collection = data.data.viewer.contributionsCollection

    // Process commits
    collection.commitContributionsByRepository.forEach((repo: any) => {
        const isPrivate = repo.repository.isPrivate
        const repoName = isPrivate ? 'Private repository' : `${repo.repository.owner.login}/${repo.repository.name}`
        repo.contributions.nodes.forEach((contribution: any) => {
            const date = contribution.occurredAt.split('T')[0]
            if (!detailsMap.has(date)) detailsMap.set(date, [])

            for (let i = 0; i < contribution.commitCount; i++) {
                detailsMap.get(date)!.push({
                    type: 'commit',
                    title: isPrivate ? 'Private contribution' : 'Commit',
                    repo: repoName,
                    url: isPrivate ? '' : contribution.commitMessages,
                    source: 'github'
                })
            }
        })
    })

    // Process PRs
    collection.pullRequestContributionsByRepository.forEach((repo: any) => {
        const isPrivate = repo.repository.isPrivate
        const repoName = isPrivate ? 'Private repository' : `${repo.repository.owner.login}/${repo.repository.name}`
        repo.contributions.nodes.forEach((contribution: any) => {
            const date = contribution.pullRequest.createdAt.split('T')[0]
            if (!detailsMap.has(date)) detailsMap.set(date, [])

            detailsMap.get(date)!.push({
                type: 'pr',
                title: isPrivate ? 'Private contribution' : contribution.pullRequest.title,
                repo: repoName,
                url: isPrivate ? '' : contribution.pullRequest.url,
                source: 'github'
            })
        })
    })

    // Process Issues
    collection.issueContributionsByRepository.forEach((repo: any) => {
        const isPrivate = repo.repository.isPrivate
        const repoName = isPrivate ? 'Private repository' : `${repo.repository.owner.login}/${repo.repository.name}`
        repo.contributions.nodes.forEach((contribution: any) => {
            const date = contribution.issue.createdAt.split('T')[0]
            if (!detailsMap.has(date)) detailsMap.set(date, [])

            detailsMap.get(date)!.push({
                type: 'issue',
                title: isPrivate ? 'Private contribution' : contribution.issue.title,
                repo: repoName,
                url: isPrivate ? '' : contribution.issue.url,
                source: 'github'
            })
        })
    })

    // Process Reviews
    collection.pullRequestReviewContributionsByRepository.forEach((repo: any) => {
        const isPrivate = repo.repository.isPrivate
        const repoName = isPrivate ? 'Private repository' : `${repo.repository.owner.login}/${repo.repository.name}`
        repo.contributions.nodes.forEach((contribution: any) => {
            const date = contribution.occurredAt.split('T')[0]
            if (!detailsMap.has(date)) detailsMap.set(date, [])

            detailsMap.get(date)!.push({
                type: 'review',
                title: isPrivate ? 'Private contribution' : `Review: ${contribution.pullRequest.title}`,
                repo: repoName,
                url: isPrivate ? '' : contribution.pullRequest.url,
                source: 'github'
            })
        })
    })

    return detailsMap
}

// GitLab's events API doesn't say whether a project is private inline, so
// we look each project up once and cache the result. If the lookup fails
// (e.g. no access), we fail safe and treat it as private.
const gitlabProjectPrivacyCache = new Map<number, boolean>()

const isGitlabProjectPrivate = async (projectId: number): Promise<boolean> => {
    if (gitlabProjectPrivacyCache.has(projectId)) {
        return gitlabProjectPrivacyCache.get(projectId)!
    }

    try {
        const response = await fetch(`https://gitlab.com/api/v4/projects/${projectId}`, {
            headers: { 'PRIVATE-TOKEN': gitlabToken.value }
        })

        if (!response.ok) {
            gitlabProjectPrivacyCache.set(projectId, true)
            return true
        }

        const project = await response.json()
        const isPrivate = project.visibility !== 'public'
        gitlabProjectPrivacyCache.set(projectId, isPrivate)
        return isPrivate
    } catch {
        gitlabProjectPrivacyCache.set(projectId, true)
        return true
    }
}

const fetchGitLabCommits = async (startDate: Date, endDate: Date): Promise<Map<string, ContributionDetail[]>> => {
    const detailsMap = new Map<string, ContributionDetail[]>()

    let page = 1
    let hasMore = true

    while (hasMore) {
        const response = await fetch(
            `https://gitlab.com/api/v4/users/${gitlabUsername.value}/events?per_page=100&page=${page}`,
            {
                headers: {
                    'PRIVATE-TOKEN': gitlabToken.value
                }
            }
        )

        if (!response.ok) {
            throw new Error(`GitLab API error: ${response.statusText}`)
        }

        const events = await response.json()

        if (events.length === 0) {
            hasMore = false
            break
        }

        for (const event of events) {
            const date = event.created_at.split('T')[0]
            const eventDate = new Date(date)

            if (eventDate >= startDate && eventDate <= endDate) {
                if (!detailsMap.has(date)) detailsMap.set(date, [])

                const isPrivate = event.project_id ? await isGitlabProjectPrivate(event.project_id) : true
                const repoName = isPrivate
                    ? 'Private project'
                    : (event.project_id ? `Project ${event.project_id}` : 'Unknown')
                const projectUrl = isPrivate ? '' : `https://gitlab.com/${event.project_id}`

                if (event.action_name === 'pushed to' || event.action_name === 'pushed new') {
                    const commitCount = event.push_data?.commit_count || 1
                    for (let i = 0; i < commitCount; i++) {
                        detailsMap.get(date)!.push({
                            type: 'commit',
                            title: isPrivate ? 'Private contribution' : (event.push_data?.commit_title || 'Commit'),
                            repo: repoName,
                            url: projectUrl,
                            source: 'gitlab'
                        })
                    }
                } else if (event.action_name === 'opened' && event.target_type === 'MergeRequest') {
                    detailsMap.get(date)!.push({
                        type: 'merge_request',
                        title: isPrivate ? 'Private contribution' : (event.target_title || 'Merge Request'),
                        repo: repoName,
                        url: isPrivate ? '' : (event.target_url || projectUrl),
                        source: 'gitlab'
                    })
                } else if (event.action_name === 'opened' && event.target_type === 'Issue') {
                    detailsMap.get(date)!.push({
                        type: 'issue',
                        title: isPrivate ? 'Private contribution' : (event.target_title || 'Issue'),
                        repo: repoName,
                        url: isPrivate ? '' : (event.target_url || projectUrl),
                        source: 'gitlab'
                    })
                }
            }
        }

        page++
        if (page > 10) hasMore = false // Safety limit
    }

    return detailsMap
}

const generateDateRange = (): { dates: string[], startDate: Date, endDate: Date } => {
    const dates: string[] = []
    let startDate: Date
    let endDate: Date

    if (selectedYear.value === 'last-year') {
        // Last year from today
        endDate = new Date()
        startDate = new Date()
        startDate.setFullYear(startDate.getFullYear() - 1)
    } else {
        // Specific calendar year
        const year = parseInt(selectedYear.value)
        startDate = new Date(year, 0, 1)
        endDate = new Date(year, 11, 31)
    }

    // Start from the Sunday before the start date
    const adjustedStart = new Date(startDate)
    adjustedStart.setDate(adjustedStart.getDate() - adjustedStart.getDay())

    const currentDate = new Date(adjustedStart)
    while (currentDate <= endDate) {
        dates.push(currentDate.toISOString().split('T')[0] ?? "damn")
        currentDate.setDate(currentDate.getDate() + 1)
    }

    return { dates, startDate, endDate }
}

const calculateLevel = (count: number): number => {
    if (count === 0) return 0
    if (count <= 3) return 1
    if (count <= 6) return 2
    if (count <= 9) return 3
    return 4
}

const canFetch = computed(() => {
    const githubReady = githubEnabled.value && !!githubToken.value
    const gitlabReady = gitlabEnabled.value && !!gitlabToken.value && !!gitlabUsername.value
    return githubReady || gitlabReady
})

const fetchContributions = async () => {
    if (!canFetch.value) {
        error.value = 'Enable at least one source (GitHub or GitLab) and provide its credentials.'
        return
    }

    loading.value = true
    error.value = ''

    try {
        const { dates, startDate, endDate } = generateDateRange()


        const [githubData, gitlabData] = await Promise.all([
            githubEnabled.value && githubToken.value
                ? fetchGitHubContributions()
                : Promise.resolve(new Map<string, ContributionDetail[]>()),
            gitlabEnabled.value && gitlabToken.value && gitlabUsername.value
                ? fetchGitLabCommits(startDate, endDate)
                : Promise.resolve(new Map<string, ContributionDetail[]>())
        ])
        // const [githubData, gitlabData] = await Promise.all([
        //     fetchGitHubContributions(),
        //     gitlabToken.value && gitlabUsername.value ? fetchGitLabCommits(startDate, endDate) : Promise.resolve(new Map())
        // ])

        const combinedContributions: ContributionDay[] = dates.map(date => {
            const githubDetails = githubData.get(date) || []
            const gitlabDetails = gitlabData.get(date) || []
            const allDetails = [...githubDetails, ...gitlabDetails]
            const totalCount = allDetails.length

            return {
                date,
                count: totalCount,
                level: calculateLevel(totalCount),
                details: allDetails
            }
        })

        contributions.value = combinedContributions
    } catch (err) {
        error.value = err instanceof Error ? err.message : 'An error occurred'
    } finally {
        loading.value = false
    }
}

onMounted(() => {
    fetchContributions()
})

watch(selectedYear, () => {
    fetchContributions()
})

const handleDayClick = (day: ContributionDay) => {
    if (day.count > 0) {
        selectedDay.value = day
        showModal.value = true
    }
}

const closeModal = () => {
    showModal.value = false
    selectedDay.value = null
}

const getTypeLabel = (type: string): string => {
    const labels: Record<string, string> = {
        commit: 'Commit',
        pr: 'Pull Request',
        issue: 'Issue',
        review: 'Review',
        merge_request: 'Merge Request'
    }
    return labels[type] || type
}

const getTypeIcon = (type: string): string => {
    const icons: Record<string, string> = {
        commit: '●',
        pr: '⊕',
        issue: '◉',
        review: '👁',
        merge_request: '⊕'
    }
    return icons[type] || '•'
}

const totalContributions = computed(() => {
    return contributions.value.reduce((sum, day) => sum + day.count, 0)
})
</script>

<template>
    <div class="contribution-graph">
        <div class="controls">
            <h2>Git Contribution Graph</h2>

            <!--
            <div class="source-toggle">
                <label class="checkbox-label">
                    <input type="checkbox" v-model="githubEnabled" />
                    Enable GitHub
                </label>
                <label class="checkbox-label">
                    <input type="checkbox" v-model="gitlabEnabled" />
                    Enable GitLab
                </label>
            </div>

            <div class="input-group" v-if="githubEnabled">
                <label>GitHub Personal Access Token:</label>
                <input v-model="githubToken" type="password" placeholder="ghp_xxxxxxxxxxxx" />
            </div>

            <div class="input-group" v-if="gitlabEnabled">
                <label>GitLab Personal Access Token:</label>
                <input v-model="gitlabToken" type="password" placeholder="glpat-xxxxxxxxxxxx" />
            </div>

            <div class="input-group" v-if="gitlabEnabled">
                <label>GitLab Username:</label>
                <input v-model="gitlabUsername" type="text" placeholder="your-username" />
            </div> -->

            <div class="input-group">
                <label>Time Period:</label>
                <div class="year-nav">
                    <button type="button" class="arrow-btn" :disabled="!canGoOlder" @click="goOlder"
                        aria-label="Previous year">
                        ‹
                    </button>
                    <select v-model="selectedYear">
                        <option v-for="year in availableYears" :key="year.value" :value="year.value">
                            {{ year.label }}
                        </option>
                    </select>
                    <button type="button" class="arrow-btn" :disabled="!canGoNewer" @click="goNewer"
                        aria-label="Next year">
                        ›
                    </button>
                </div>
            </div>

            <div v-if="loading" class="loading-indicator">Loading contributions…</div>
            <div v-if="error" class="error">{{ error }}</div>
        </div>

        <div v-if="contributions.length > 0" class="graph-container">
            <div class="stats">
                <strong>{{ totalContributions }}</strong> contributions
            </div>

            <div class="graph">
                <div class="months">
                    <span>Jan</span>
                    <span>Feb</span>
                    <span>Mar</span>
                    <span>Apr</span>
                    <span>May</span>
                    <span>Jun</span>
                    <span>Jul</span>
                    <span>Aug</span>
                    <span>Sep</span>
                    <span>Oct</span>
                    <span>Nov</span>
                    <span>Dec</span>
                </div>
                <div class="grid">
                    <div class="week" v-for="(week, index) in weeks" :key="index">
                        <div v-for="day in week" :key="day.date" class="day"
                            :style="{ backgroundColor: getLevelColor(day.level) }"
                            :title="`${day.count} contributions on ${day.date}`" @click="handleDayClick(day)" />
                    </div>
                </div>
            </div>

            <div class="legend">
                <span>Less</span>
                <div class="level" :style="{ backgroundColor: getLevelColor(0) }"></div>
                <div class="level" :style="{ backgroundColor: getLevelColor(1) }"></div>
                <div class="level" :style="{ backgroundColor: getLevelColor(2) }"></div>
                <div class="level" :style="{ backgroundColor: getLevelColor(3) }"></div>
                <div class="level" :style="{ backgroundColor: getLevelColor(4) }"></div>
                <span>More</span>
            </div>
        </div>

        <!-- Modal for contribution details -->
        <div v-if="showModal" class="modal-overlay" @click="closeModal">
            <div class="modal-content" @click.stop>
                <div class="modal-header">
                    <h3>Contributions on {{ selectedDay?.date }}</h3>
                    <button class="close-btn" @click="closeModal">×</button>
                </div>
                <div class="modal-body">
                    <p class="total-count">{{ selectedDay?.count }} total contributions</p>

                    <div class="contributions-list">
                        <div v-for="(detail, index) in selectedDay?.details" :key="index" class="contribution-item">
                            <div class="contribution-header">
                                <span class="type-badge" :class="`badge-${detail.type}`">
                                    {{ getTypeIcon(detail.type) }} {{ getTypeLabel(detail.type) }}
                                </span>
                                <span class="source-badge" :class="`source-${detail.source}`">
                                    {{ detail.source }}
                                </span>
                            </div>
                            <div class="contribution-title">{{ detail.title }}</div>
                            <div class="contribution-repo">{{ detail.repo }}</div>
                            <a v-if="detail.url" :href="detail.url" target="_blank" rel="noopener noreferrer"
                                class="contribution-link">
                                View →
                            </a>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.contribution-graph {
    --bg: #0d1117;
    --bg-elevated: #161b22;
    --border: #30363d;
    --text: #c9d1d9;
    --text-muted: #8b949e;
    --accent: #2f81f7;

    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    max-width: 1000px;
    margin: 0 auto;
    padding: 20px;
    background-color: var(--bg);
    color: var(--text);
}

.controls {
    margin-bottom: 30px;
}

h2 {
    margin-bottom: 20px;
    color: var(--text);
}

.source-toggle {
    display: flex;
    gap: 20px;
    margin-bottom: 15px;
}

.checkbox-label {
    display: flex;
    align-items: center;
    gap: 6px;
    font-weight: 500;
    color: var(--text);
    cursor: pointer;
}

.checkbox-label input[type="checkbox"] {
    width: 15px;
    height: 15px;
    cursor: pointer;
    accent-color: var(--accent);
}

.input-group {
    display: flex;
    justify-content: center;
    margin-bottom: 15px;
}

.input-group label {
    display: block;
    margin-bottom: 5px;
    font-weight: 500;
    color: var(--text);
}

.input-group input {
    width: 100%;
    padding: 8px 12px;
    border: 1px solid var(--border);
    border-radius: 6px;
    font-size: 14px;
    background-color: var(--bg-elevated);
    color: var(--text);
}

.input-group input::placeholder {
    color: var(--text-muted);
}

.input-group select {
    width: 100%;
    padding: 8px 12px;
    border: 1px solid var(--border);
    border-radius: 6px;
    font-size: 14px;
    background-color: var(--bg-elevated);
    color: var(--text);
    cursor: pointer;
}

.input-group select option {
    background-color: var(--bg-elevated);
    color: var(--text);
}

button {
    background-color: #2da44e;
    color: white;
    border: none;
    padding: 10px 20px;
    border-radius: 6px;
    font-size: 14px;
    cursor: pointer;
    font-weight: 500;
}

button:hover:not(:disabled) {
    background-color: #2c974b;
}

button:disabled {
    background-color: #235c33;
    color: var(--text-muted);
    cursor: not-allowed;
}

.error {
    margin-top: 10px;
    padding: 10px;
    background-color: #3b1a1c;
    border: 1px solid #f85149;
    border-radius: 6px;
    color: #ffa198;
}

.graph-container {
    background: var(--bg-elevated);
    border: 1px solid var(--border);
    border-radius: 6px;
    padding: 20px;
}

.stats {
    margin-bottom: 15px;
    font-size: 14px;
    color: var(--text-muted);
}

.graph {
    overflow-x: auto;
}

.months {
    display: flex;
    gap: 14px;
    margin-bottom: 5px;
    font-size: 12px;
    color: var(--text-muted);
    padding-left: 20px;
}

.months span {
    flex: 0 0 52px;
}

.grid {
    display: flex;
    gap: 3px;
}

.week {
    display: flex;
    flex-direction: column;
    gap: 3px;
}

.day {
    width: 11px;
    height: 11px;
    border-radius: 2px;
    cursor: pointer;
}

.day:hover {
    outline: 2px solid rgba(240, 246, 252, 0.2);
}

.legend {
    display: flex;
    align-items: center;
    gap: 5px;
    margin-top: 15px;
    font-size: 12px;
    color: var(--text-muted);
    justify-content: flex-end;
}

.level {
    width: 11px;
    height: 11px;
    border-radius: 2px;
}

/* Modal styles */
.modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.6);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    padding: 20px;
}

.modal-content {
    background: var(--bg-elevated);
    border: 1px solid var(--border);
    border-radius: 8px;
    max-width: 600px;
    width: 100%;
    max-height: 80vh;
    display: flex;
    flex-direction: column;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
}

.modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px;
    border-bottom: 1px solid var(--border);
}

.modal-header h3 {
    margin: 0;
    font-size: 18px;
    color: var(--text);
}

.close-btn {
    background: none;
    border: none;
    font-size: 28px;
    color: var(--text-muted);
    cursor: pointer;
    padding: 0;
    width: 32px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 4px;
}

.close-btn:hover {
    background: var(--border);
}

.modal-body {
    padding: 20px;
    overflow-y: auto;
}

.total-count {
    font-size: 14px;
    color: var(--text-muted);
    margin-bottom: 20px;
}

.contributions-list {
    display: flex;
    flex-direction: column;
    gap: 15px;
}

.contribution-item {
    padding: 12px;
    border: 1px solid var(--border);
    border-radius: 6px;
    background: #0d1117;
}

.contribution-header {
    display: flex;
    gap: 8px;
    margin-bottom: 8px;
    flex-wrap: wrap;
}

.type-badge,
.source-badge {
    padding: 2px 8px;
    border-radius: 12px;
    font-size: 12px;
    font-weight: 500;
}

.type-badge {
    background: #163650;
    color: #58a6ff;
}

.badge-commit {
    background: #163650;
    color: #58a6ff;
}

.badge-pr,
.badge-merge_request {
    background: #123822;
    color: #3fb950;
}

.badge-issue {
    background: #3d2e00;
    color: #d29922;
}

.badge-review {
    background: #4a2600;
    color: #db6d28;
}

.source-badge {
    background: var(--border);
    color: var(--text-muted);
    text-transform: uppercase;
}

.source-github {
    background: #f0f6fc;
    color: #0d1117;
}

.source-gitlab {
    background: #fc6d26;
    color: white;
}

.contribution-title {
    font-size: 14px;
    font-weight: 500;
    color: var(--text);
    margin-bottom: 4px;
}

.contribution-repo {
    font-size: 12px;
    color: var(--text-muted);
    margin-bottom: 8px;
}

.contribution-link {
    font-size: 12px;
    color: var(--accent);
    text-decoration: none;
    display: inline-flex;
    align-items: center;
    gap: 4px;
}

.contribution-link:hover {
    text-decoration: underline;
}
</style>