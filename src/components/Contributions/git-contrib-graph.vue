<script setup lang="ts">
import {
    ref, computed,
    watch,
    onMounted
} from 'vue'

import './git-contrib-graph.css'

interface ContributionDay {
    date: string
    count: number
    level: number
    details: ContributionDetail[]
    inRange: boolean
}

interface ContributionDetail {
    type: 'commit' | 'pr' | 'issue' | 'review' | 'merge_request'
    title: string
    repo: string
    url: string
    source: 'github' | 'gitlab'
}

const githubToken = ref(import.meta.env.VITE_GITHUB_TOKEN ?? '')
const gitlabToken = ref(import.meta.env.VITE_GITLAB_TOKEN ?? '')
const gitlabUsername = ref(import.meta.env.VITE_GITLAB_USERNAME ?? "Wabtey")

const githubEnabled = ref(true)
const gitlabEnabled = ref(true)

const formatLocalDate = (date: Date): string => {
    const year = date.getFullYear()
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    return `${year}-${month}-${day}`
}

const selectedYear = ref<string>('2023') // ref<string>('last-year')
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
const currentYearIndex = computed(() =>
    availableYears.value.findIndex(y => y.value === selectedYear.value)
)

const canGoOlder = computed(() =>
    currentYearIndex.value !== -1 && currentYearIndex.value < availableYears.value.length - 1
)

const canGoNewer = computed(() =>
    currentYearIndex.value > 0
)

const goOlder = () => {
    if (!canGoOlder.value) return
    selectedYear.value = availableYears.value[currentYearIndex.value + 1]!.value
}

const goNewer = () => {
    if (!canGoNewer.value) return
    selectedYear.value = availableYears.value[currentYearIndex.value - 1]!.value
}

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

const monthLabels = computed(() => {
    const labels = new Map<number, string>()

    weeks.value.forEach((week, weekIndex) => {
        for (const day of week) {
            if (!day.inRange) continue
            const date = new Date(day.date + 'T00:00:00')
            if (date.getDate() === 1) {
                labels.set(weekIndex, date.toLocaleString('default', { month: 'short' }))
                break
            }
        }
    })

    return labels
})

const getMonthLabel = (weekIndex: number): string => {
    return monthLabels.value.get(weekIndex) ?? ''
}

const weekdayLabels = [
    { row: 0, text: 'Mon' },
    { row: 1, text: '' },
    { row: 2, text: 'Wed' },
    { row: 3, text: '' },
    { row: 4, text: 'Fri' },
    { row: 5, text: '' },
    { row: 6, text: '' }
]

const getLevelColor = (level: number): string => {
    const colors = [
        '#161b22',
        '#0e4429',
        '#006d32',
        '#26a641',
        '#39d353'
    ]
    return colors[Math.min(level, 4)] ?? '#161b22'
}

/* --------------------------------- Queries -------------------------------- */
const fetchGitHubContributions = async (fromDate: Date, toDate: Date): Promise<Map<string, ContributionDetail[]>> => {
    const query = `
    query($from: DateTime!, $to: DateTime!) {
      viewer {
        contributionsCollection(from: $from, to: $to) {
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
        body: JSON.stringify({
            query,
            variables: {
                // GitHub caps this range at 1 year; our per-year and
                // rolling windows both stay within that.
                from: fromDate.toISOString(),
                to: toDate.toISOString()
            }
        })
    })

    if (!response.ok) {
        throw new Error(`GitHub API error: ${response.statusText}`)
    }

    const data = await response.json()

    if (data.errors?.length) {
        // GitHub returns HTTP 200 even when some fields fail (e.g. insufficient
        // token scope to resolve isPrivate/owner for a given repo). Log instead
        // of failing outright, since data.data may still be partially usable.
        console.warn('GitHub GraphQL returned partial errors:', data.errors)
    }

    const detailsMap = new Map<string, ContributionDetail[]>()
    const collection = data.data.viewer.contributionsCollection

    // Process commits
    collection.commitContributionsByRepository.forEach((repo: any) => {
        if (!repo?.repository) return // couldn't resolve this repo — skip rather than crash
        const isPrivate = !!repo.repository.isPrivate
        const repoName = isPrivate ? 'Private repository' : `${repo.repository.owner?.login ?? '?'}/${repo.repository.name}`
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
        if (!repo?.repository) return
        const isPrivate = !!repo.repository.isPrivate
        const repoName = isPrivate ? 'Private repository' : `${repo.repository.owner?.login ?? '?'}/${repo.repository.name}`
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
        if (!repo?.repository) return
        const isPrivate = !!repo.repository.isPrivate
        const repoName = isPrivate ? 'Private repository' : `${repo.repository.owner?.login ?? '?'}/${repo.repository.name}`
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
        if (!repo?.repository) return
        const isPrivate = !!repo.repository.isPrivate
        const repoName = isPrivate ? 'Private repository' : `${repo.repository.owner?.login ?? '?'}/${repo.repository.name}`
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

// GitLab's events API doesn't say whether a project is private inline.
// Rather than making one *authenticated* call per project (which was
// throwing 403s for projects the token's scope can't fully resolve, even
// when the project is genuinely public), we:
//   1. Fetch the list of projects the user is actually a member of ONCE —
//      this covers almost every event and needs no per-project call.
//   2. For anything not in that list (e.g. a one-off public contribution
//      to a repo the user never joined), fall back to a single
//      *unauthenticated* lookup — public project info doesn't require a
//      token, so this can't fail due to scope, only for truly
//      private/internal projects, which is exactly what we want.
const gitlabProjectPrivacyCache = new Map<number, boolean>()
let gitlabMembershipLoaded = false

const loadGitlabMembershipProjects = async (): Promise<void> => {
    if (gitlabMembershipLoaded) return
    gitlabMembershipLoaded = true

    let page = 1
    while (page <= 10) { // safety cap
        try {
            const response = await fetch(
                `https://gitlab.com/api/v4/projects?membership=true&simple=true&per_page=100&page=${page}`,
                { headers: { 'PRIVATE-TOKEN': gitlabToken.value } }
            )

            if (!response.ok) break

            const projects = await response.json()
            if (!Array.isArray(projects) || projects.length === 0) break

            for (const project of projects) {
                gitlabProjectPrivacyCache.set(project.id, project.visibility !== 'public')
            }

            if (projects.length < 100) break
            page++
        } catch {
            break
        }
    }
}

const isGitlabProjectPrivate = async (projectId: number): Promise<boolean> => {
    if (gitlabProjectPrivacyCache.has(projectId)) {
        return gitlabProjectPrivacyCache.get(projectId)!
    }

    try {
        // Unauthenticated on purpose: a public project resolves fine without
        // a token, and anything that isn't public correctly 401/403/404s here.
        const response = await fetch(`https://gitlab.com/api/v4/projects/${projectId}`)

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

const fetchGitLabCommits = async (startStr: string, endStr: string): Promise<Map<string, ContributionDetail[]>> => {
    const detailsMap = new Map<string, ContributionDetail[]>()

    await loadGitlabMembershipProjects()

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

            if (date >= startStr && date <= endStr) {
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

    // Start from the Monday on/before the start date
    // getDay(): Sun=0..Sat=6, so days-since-Monday is (day + 6) % 7
    const adjustedStart = new Date(startDate)
    adjustedStart.setDate(adjustedStart.getDate() - ((adjustedStart.getDay() + 6) % 7))

    const currentDate = new Date(adjustedStart)
    while (currentDate <= endDate) {
        dates.push(formatLocalDate(currentDate))
        currentDate.setDate(currentDate.getDate() + 1)
    }

    /* -------------- Pad out week days and months for a rectangle -------------- */
    while (currentDate.getDay() !== 1) {
        dates.push(formatLocalDate(currentDate))
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
        const startStr = formatLocalDate(startDate)
        const endStr = formatLocalDate(endDate)

        const [githubData, gitlabData] = await Promise.all([
            githubEnabled.value && githubToken.value
                ? fetchGitHubContributions(startDate, endDate)
                : Promise.resolve(new Map<string, ContributionDetail[]>()),
            gitlabEnabled.value && gitlabToken.value && gitlabUsername.value
                ? fetchGitLabCommits(startStr, endStr)
                : Promise.resolve(new Map<string, ContributionDetail[]>())
        ])

        const combinedContributions: ContributionDay[] = dates.map(date => {
            const githubDetails = githubData.get(date) || []
            const gitlabDetails = gitlabData.get(date) || []
            const allDetails = [...githubDetails, ...gitlabDetails]
            const totalCount = allDetails.length

            return {
                date,
                count: totalCount,
                level: calculateLevel(totalCount),
                details: allDetails,
                inRange: date >= startStr && date <= endStr
            }
        })

        contributions.value = combinedContributions
    } catch (err) {
        error.value = err instanceof Error ? err.message : 'An error occurred'
    } finally {
        loading.value = false
    }
}

// NEW: auto-fetch on load and whenever the selected time period changes
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
                <div class="graph-inner">
                    <div class="weekday-col">
                        <span v-for="label in weekdayLabels" :key="label.row" class="weekday-cell">{{ label.text
                            }}</span>
                    </div>
                    <div class="graph-scroll">
                        <div class="months">
                            <span v-for="(week, index) in weeks" :key="index" class="month-cell">
                                {{ getMonthLabel(index) }}
                            </span>
                        </div>
                        <div class="grid">
                            <div class="week" v-for="(week, index) in weeks" :key="index">
                                <div v-for="day in week" :key="day.date" class="day"
                                    :class="{ 'day-out-of-range': !day.inRange }"
                                    :style="{ backgroundColor: day.inRange ? getLevelColor(day.level) : 'transparent' }"
                                    :title="day.inRange ? `${day.count} contributions on ${day.date}` : undefined"
                                    @click="handleDayClick(day)" />
                            </div>
                        </div>
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