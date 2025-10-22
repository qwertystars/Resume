# GitHub Analysis Implementation Guide

## Overview
This guide covers integrating with GitHub API to fetch user profiles, analyze repositories, calculate contribution metrics, and generate developer scores.

---

## GitHub API Integration

### 1. GitHub OAuth Setup

**File: `app/api/v1/github.py`**

```python
from fastapi import APIRouter, Depends, HTTPException
from app.core.config import settings
import httpx

router = APIRouter(prefix="/github", tags=["github"])

GITHUB_CLIENT_ID = settings.GITHUB_CLIENT_ID
GITHUB_CLIENT_SECRET = settings.GITHUB_CLIENT_SECRET
GITHUB_REDIRECT_URI = settings.GITHUB_REDIRECT_URI

@router.post("/connect")
async def connect_github(
    code: str,
    candidate_id: str = None,
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Exchange OAuth code for access token"""

    # Exchange code for token
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://github.com/login/oauth/access_token",
            json={
                "client_id": GITHUB_CLIENT_ID,
                "client_secret": GITHUB_CLIENT_SECRET,
                "code": code,
                "redirect_uri": GITHUB_REDIRECT_URI
            },
            headers={"Accept": "application/json"}
        )

    data = response.json()
    access_token = data.get("access_token")

    if not access_token:
        raise HTTPException(400, "Failed to get access token")

    # Fetch user info
    github_user = await fetch_github_user(access_token)

    # Store or update candidate
    if candidate_id:
        candidate = db.query(Candidate).filter(Candidate.id == candidate_id).first()
        candidate.github_url = github_user["html_url"]
    else:
        # Find or create candidate
        candidate = db.query(Candidate).filter(
            Candidate.email == github_user["email"]
        ).first()

        if not candidate:
            candidate = Candidate(
                team_id=current_user.team_id,
                full_name=github_user["name"],
                email=github_user["email"],
                github_url=github_user["html_url"]
            )
            db.add(candidate)
            db.commit()

    # Queue analysis job
    from app.workers.github_tasks import analyze_github_profile_task
    analyze_github_profile_task.delay(str(candidate.id), github_user["login"], access_token)

    return {"success": True, "data": {"candidate_id": str(candidate.id)}}
```

### 2. GitHub Service

**File: `app/services/github_analyzer.py`**

```python
import httpx
from typing import Dict, List, Any
from datetime import datetime, timedelta
import asyncio

class GitHubAnalyzer:
    def __init__(self, access_token: str = None):
        self.access_token = access_token or settings.GITHUB_TOKEN
        self.base_url = "https://api.github.com"
        self.headers = {
            "Authorization": f"token {self.access_token}",
            "Accept": "application/vnd.github.v3+json"
        }

    async def get_user_profile(self, username: str) -> Dict[str, Any]:
        """Fetch user profile data"""
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/users/{username}",
                headers=self.headers
            )
            response.raise_for_status()
            return response.json()

    async def get_user_repos(self, username: str) -> List[Dict[str, Any]]:
        """Fetch all public repositories"""
        repos = []
        page = 1

        async with httpx.AsyncClient() as client:
            while True:
                response = await client.get(
                    f"{self.base_url}/users/{username}/repos",
                    headers=self.headers,
                    params={"page": page, "per_page": 100, "sort": "updated"}
                )
                response.raise_for_status()
                data = response.json()

                if not data:
                    break

                repos.extend(data)
                page += 1

                # Limit to 500 repos
                if len(repos) >= 500:
                    break

        return repos

    async def get_repo_languages(self, username: str, repo_name: str) -> Dict[str, int]:
        """Get language breakdown for a repository"""
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/repos/{username}/{repo_name}/languages",
                headers=self.headers
            )
            if response.status_code == 200:
                return response.json()
            return {}

    async def get_user_contributions(self, username: str) -> Dict[str, Any]:
        """Fetch contribution data for the last year"""
        async with httpx.AsyncClient() as client:
            # Use GraphQL API for contribution graph
            query = """
            query($username: String!) {
              user(login: $username) {
                contributionsCollection {
                  contributionCalendar {
                    totalContributions
                    weeks {
                      contributionDays {
                        contributionCount
                        date
                      }
                    }
                  }
                  commitContributionsByRepository {
                    repository {
                      name
                    }
                    contributions {
                      totalCount
                    }
                  }
                }
              }
            }
            """

            response = await client.post(
                "https://api.github.com/graphql",
                json={"query": query, "variables": {"username": username}},
                headers=self.headers
            )

            if response.status_code == 200:
                data = response.json()
                return data["data"]["user"]["contributionsCollection"]

            return {}

    async def calculate_commit_streak(self, contributions: List[Dict]) -> Dict[str, int]:
        """Calculate current and longest commit streak"""
        dates = [c["date"] for c in contributions if c["contributionCount"] > 0]

        if not dates:
            return {"current_streak": 0, "longest_streak": 0}

        dates = sorted([datetime.fromisoformat(d) for d in dates])

        current_streak = 0
        longest_streak = 0
        temp_streak = 1

        # Check if still active
        if (datetime.now() - dates[-1]).days <= 1:
            current_streak = 1

        for i in range(1, len(dates)):
            diff = (dates[i] - dates[i-1]).days

            if diff == 1:
                temp_streak += 1
                if i == len(dates) - 1 or (datetime.now() - dates[i]).days <= 1:
                    current_streak = temp_streak
            else:
                longest_streak = max(longest_streak, temp_streak)
                temp_streak = 1

        longest_streak = max(longest_streak, temp_streak)

        return {"current_streak": current_streak, "longest_streak": longest_streak}

    async def analyze_code_quality(self, username: str, repo_name: str) -> Dict[str, Any]:
        """Analyze code quality metrics"""
        metrics = {
            "has_readme": False,
            "has_tests": False,
            "has_ci": False,
            "readme_quality": 0
        }

        async with httpx.AsyncClient() as client:
            # Check README
            readme_response = await client.get(
                f"{self.base_url}/repos/{username}/{repo_name}/readme",
                headers=self.headers
            )

            if readme_response.status_code == 200:
                metrics["has_readme"] = True
                readme_data = readme_response.json()

                # Decode content and calculate quality score
                import base64
                content = base64.b64decode(readme_data["content"]).decode('utf-8')
                metrics["readme_quality"] = self._score_readme_quality(content)

            # Check for test files
            contents_response = await client.get(
                f"{self.base_url}/repos/{username}/{repo_name}/contents",
                headers=self.headers
            )

            if contents_response.status_code == 200:
                files = contents_response.json()
                file_names = [f["name"].lower() for f in files if isinstance(f, dict)]

                metrics["has_tests"] = any(
                    "test" in name or "spec" in name
                    for name in file_names
                )

                metrics["has_ci"] = any(
                    name in [".github", ".gitlab-ci.yml", ".travis.yml", "circle.yml"]
                    for name in file_names
                )

        return metrics

    def _score_readme_quality(self, content: str) -> int:
        """Score README quality (0-100)"""
        score = 0

        # Length
        if len(content) > 500:
            score += 20
        elif len(content) > 200:
            score += 10

        # Sections
        sections = ["installation", "usage", "example", "api", "contributing"]
        for section in sections:
            if section in content.lower():
                score += 10

        # Code blocks
        if "```" in content:
            score += 15

        # Images
        if "![" in content or "<img" in content:
            score += 10

        # Links
        if "[" in content and "](" in content:
            score += 5

        return min(score, 100)

    async def aggregate_language_stats(self, repos: List[Dict]) -> Dict[str, float]:
        """Calculate language percentages across all repos"""
        language_bytes = {}

        # Fetch languages for each repo
        tasks = []
        for repo in repos[:50]:  # Limit to top 50 repos
            if not repo["fork"]:
                tasks.append(
                    self.get_repo_languages(repo["owner"]["login"], repo["name"])
                )

        results = await asyncio.gather(*tasks, return_exceptions=True)

        for result in results:
            if isinstance(result, dict):
                for lang, bytes_count in result.items():
                    language_bytes[lang] = language_bytes.get(lang, 0) + bytes_count

        # Calculate percentages
        total_bytes = sum(language_bytes.values())
        if total_bytes == 0:
            return {}

        language_percentages = {
            lang: round((bytes_count / total_bytes) * 100, 1)
            for lang, bytes_count in language_bytes.items()
        }

        # Sort by percentage
        return dict(sorted(language_percentages.items(), key=lambda x: x[1], reverse=True))

    async def get_pull_request_stats(self, username: str) -> Dict[str, int]:
        """Get PR statistics"""
        async with httpx.AsyncClient() as client:
            # Search for PRs authored by user
            response = await client.get(
                f"{self.base_url}/search/issues",
                headers=self.headers,
                params={
                    "q": f"author:{username} type:pr",
                    "per_page": 1
                }
            )

            if response.status_code == 200:
                data = response.json()
                return {"total_prs": data["total_count"]}

            return {"total_prs": 0}

    async def complete_analysis(self, username: str) -> Dict[str, Any]:
        """Run complete GitHub profile analysis"""

        # Fetch all data in parallel
        user_profile, repos, contributions, pr_stats = await asyncio.gather(
            self.get_user_profile(username),
            self.get_user_repos(username),
            self.get_user_contributions(username),
            self.get_pull_request_stats(username)
        )

        # Calculate metrics
        total_stars = sum(repo["stargazers_count"] for repo in repos)
        total_forks = sum(repo["forks_count"] for repo in repos)

        # Language breakdown
        languages = await self.aggregate_language_stats(repos)

        # Contribution stats
        contribution_calendar = contributions.get("contributionCalendar", {})
        total_contributions = contribution_calendar.get("totalContributions", 0)

        # Streak calculation
        contribution_days = []
        for week in contribution_calendar.get("weeks", []):
            contribution_days.extend(week.get("contributionDays", []))

        streaks = await self.calculate_commit_streak(contribution_days)

        # Most active repo
        repos_sorted = sorted(repos, key=lambda r: r["stargazers_count"], reverse=True)
        most_active_repo = repos_sorted[0]["name"] if repos_sorted else None

        return {
            "github_username": username,
            "github_user_id": user_profile["id"],
            "profile_url": user_profile["html_url"],
            "public_repos": user_profile["public_repos"],
            "followers": user_profile["followers"],
            "following": user_profile["following"],
            "total_stars": total_stars,
            "total_forks": total_forks,
            "total_commits": total_contributions,
            "languages": languages,
            "contributions_last_year": total_contributions,
            "current_streak": streaks["current_streak"],
            "longest_streak": streaks["longest_streak"],
            "most_active_repo": most_active_repo,
            "total_prs": pr_stats["total_prs"],
            "github_created_at": user_profile["created_at"],
            "last_commit_at": repos_sorted[0]["updated_at"] if repos_sorted else None
        }
```

### 3. Celery Task

**File: `app/workers/github_tasks.py`**

```python
from celery import shared_task
from app.services.github_analyzer import GitHubAnalyzer
from app.models.candidate_github_profile import CandidateGitHubProfile
from app.core.database import SessionLocal
import logging

logger = logging.getLogger(__name__)

@shared_task(bind=True, max_retries=3)
def analyze_github_profile_task(self, candidate_id: str, username: str, access_token: str = None):
    """Analyze GitHub profile in background"""
    db = SessionLocal()

    try:
        analyzer = GitHubAnalyzer(access_token)

        # Run complete analysis
        analysis = await analyzer.complete_analysis(username)

        # Save to database
        profile = db.query(CandidateGitHubProfile).filter(
            CandidateGitHubProfile.candidate_id == candidate_id
        ).first()

        if profile:
            # Update existing
            for key, value in analysis.items():
                setattr(profile, key, value)
        else:
            # Create new
            profile = CandidateGitHubProfile(
                candidate_id=candidate_id,
                **analysis
            )
            db.add(profile)

        profile.last_fetched_at = datetime.utcnow()
        db.commit()

        # Trigger re-scoring
        from app.workers.scoring_tasks import calculate_score_task
        calculate_score_task.delay(candidate_id)

        logger.info(f"Successfully analyzed GitHub profile for {username}")

    except Exception as e:
        logger.error(f"Error analyzing GitHub profile {username}: {str(e)}")
        raise self.retry(exc=e, countdown=60)

    finally:
        db.close()
```

---

## Frontend Implementation

### GitHub Connect Component

**File: `frontend/src/components/github/GitHubConnect.tsx`**

```typescript
import React, { useEffect } from 'react';
import { Github } from 'lucide-react';
import { useSearchParams } from 'react-router-dom';
import { connectGitHub } from '../../services/githubService';

const GITHUB_CLIENT_ID = import.meta.env.VITE_GITHUB_CLIENT_ID;
const REDIRECT_URI = `${window.location.origin}/auth/github/callback`;

export const GitHubConnect: React.FC<{ candidateId?: string }> = ({ candidateId }) => {
  const [searchParams] = useSearchParams();
  const [loading, setLoading] = React.useState(false);
  const [error, setError] = React.useState<string | null>(null);

  useEffect(() => {
    // Handle OAuth callback
    const code = searchParams.get('code');
    if (code) {
      handleCallback(code);
    }
  }, [searchParams]);

  const handleConnect = () => {
    const state = Math.random().toString(36).substring(7);
    sessionStorage.setItem('github_state', state);

    const authUrl = `https://github.com/login/oauth/authorize?` +
      `client_id=${GITHUB_CLIENT_ID}&` +
      `redirect_uri=${encodeURIComponent(REDIRECT_URI)}&` +
      `scope=read:user,repo&` +
      `state=${state}`;

    window.location.href = authUrl;
  };

  const handleCallback = async (code: string) => {
    setLoading(true);
    setError(null);

    try {
      const state = searchParams.get('state');
      const savedState = sessionStorage.getItem('github_state');

      if (state !== savedState) {
        throw new Error('Invalid state parameter');
      }

      await connectGitHub(code, candidateId);

      // Redirect or show success
      window.location.href = '/candidates';
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
      sessionStorage.removeItem('github_state');
    }
  };

  return (
    <div>
      <button
        onClick={handleConnect}
        disabled={loading}
        className="flex items-center space-x-2 px-4 py-2 bg-gray-900 text-white rounded-lg hover:bg-gray-800 transition-colors"
      >
        <Github className="w-5 h-5" />
        <span>{loading ? 'Connecting...' : 'Connect GitHub'}</span>
      </button>

      {error && (
        <p className="mt-2 text-sm text-red-600">{error}</p>
      )}
    </div>
  );
};
```

### GitHub Profile Display

**File: `frontend/src/components/github/GitHubProfile.tsx`**

```typescript
import React from 'react';
import { Github, Star, GitFork, Code, TrendingUp } from 'lucide-react';

interface GitHubProfileProps {
  profile: {
    github_username: string;
    public_repos: number;
    followers: number;
    total_stars: number;
    total_forks: number;
    total_commits: number;
    languages: Record<string, number>;
    contributions_last_year: number;
    current_streak: number;
    longest_streak: number;
  };
}

export const GitHubProfile: React.FC<GitHubProfileProps> = ({ profile }) => {
  const topLanguages = Object.entries(profile.languages).slice(0, 5);

  return (
    <div className="bg-white rounded-lg shadow p-6">
      <div className="flex items-center space-x-3 mb-6">
        <Github className="w-6 h-6" />
        <h3 className="text-lg font-semibold">GitHub Profile</h3>
        <a
          href={`https://github.com/${profile.github_username}`}
          target="_blank"
          rel="noopener noreferrer"
          className="text-blue-600 hover:underline text-sm"
        >
          @{profile.github_username}
        </a>
      </div>

      {/* Stats Grid */}
      <div className="grid grid-cols-2 md:grid-cols-4 gap-4 mb-6">
        <div className="text-center">
          <div className="text-2xl font-bold text-gray-900">
            {profile.public_repos}
          </div>
          <div className="text-sm text-gray-500">Repositories</div>
        </div>

        <div className="text-center">
          <div className="text-2xl font-bold text-gray-900">
            {profile.followers}
          </div>
          <div className="text-sm text-gray-500">Followers</div>
        </div>

        <div className="text-center">
          <div className="flex items-center justify-center space-x-1">
            <Star className="w-4 h-4 text-yellow-500" />
            <div className="text-2xl font-bold text-gray-900">
              {profile.total_stars}
            </div>
          </div>
          <div className="text-sm text-gray-500">Total Stars</div>
        </div>

        <div className="text-center">
          <div className="text-2xl font-bold text-gray-900">
            {profile.total_commits.toLocaleString()}
          </div>
          <div className="text-sm text-gray-500">Commits</div>
        </div>
      </div>

      {/* Languages */}
      <div className="mb-6">
        <h4 className="text-sm font-medium text-gray-700 mb-3">
          Top Languages
        </h4>
        <div className="space-y-2">
          {topLanguages.map(([lang, percentage]) => (
            <div key={lang}>
              <div className="flex justify-between text-sm mb-1">
                <span className="font-medium">{lang}</span>
                <span className="text-gray-500">{percentage}%</span>
              </div>
              <div className="h-2 bg-gray-200 rounded-full overflow-hidden">
                <div
                  className="h-full bg-blue-500"
                  style={{ width: `${percentage}%` }}
                />
              </div>
            </div>
          ))}
        </div>
      </div>

      {/* Streaks */}
      <div className="grid grid-cols-2 gap-4">
        <div className="bg-gray-50 rounded-lg p-3">
          <div className="flex items-center space-x-2 text-orange-600 mb-1">
            <TrendingUp className="w-4 h-4" />
            <span className="text-sm font-medium">Current Streak</span>
          </div>
          <div className="text-xl font-bold">
            {profile.current_streak} days
          </div>
        </div>

        <div className="bg-gray-50 rounded-lg p-3">
          <div className="flex items-center space-x-2 text-green-600 mb-1">
            <TrendingUp className="w-4 h-4" />
            <span className="text-sm font-medium">Longest Streak</span>
          </div>
          <div className="text-xl font-bold">
            {profile.longest_streak} days
          </div>
        </div>
      </div>
    </div>
  );
};
```

---

## Rate Limiting

GitHub API has rate limits:
- **Authenticated**: 5,000 requests/hour
- **Unauthenticated**: 60 requests/hour

Implement caching and smart fetching:

```python
from functools import lru_cache
from datetime import timedelta

@lru_cache(maxsize=1000)
async def get_cached_user_profile(username: str):
    # Cache for 1 hour
    return await analyzer.get_user_profile(username)
```

---

## Testing

```python
import pytest
from app.services.github_analyzer import GitHubAnalyzer

@pytest.mark.asyncio
async def test_github_profile_fetch():
    analyzer = GitHubAnalyzer()
    profile = await analyzer.get_user_profile("torvalds")

    assert profile["login"] == "torvalds"
    assert profile["public_repos"] > 0

@pytest.mark.asyncio
async def test_language_aggregation():
    analyzer = GitHubAnalyzer()
    repos = await analyzer.get_user_repos("octocat")
    languages = await analyzer.aggregate_language_stats(repos)

    assert isinstance(languages, dict)
    assert sum(languages.values()) <= 100.1  # Percentage sum
```

This implementation provides comprehensive GitHub analysis with profile fetching, contribution metrics, and code quality assessment.
