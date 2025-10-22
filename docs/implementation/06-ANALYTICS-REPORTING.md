# Analytics & Reporting Implementation Guide

## Overview
Build comprehensive analytics dashboards with charts, metrics, and export functionality.

---

## Backend Analytics Service

**File: `app/services/analytics_service.py`**

```python
from sqlalchemy.orm import Session
from sqlalchemy import func, and_
from app.models.candidate import Candidate
from app.models.candidate_activities import CandidateActivity
from datetime import datetime, timedelta
from typing import Dict, List

class AnalyticsService:
    def __init__(self, db: Session, team_id: str):
        self.db = db
        self.team_id = team_id

    def get_overview_metrics(self, start_date: datetime = None, end_date: datetime = None):
        """Get overview dashboard metrics"""

        if not start_date:
            start_date = datetime.now() - timedelta(days=30)
        if not end_date:
            end_date = datetime.now()

        # Total candidates
        total_candidates = self.db.query(Candidate).filter(
            Candidate.team_id == self.team_id
        ).count()

        # New this week
        week_ago = datetime.now() - timedelta(days=7)
        new_this_week = self.db.query(Candidate).filter(
            Candidate.team_id == self.team_id,
            Candidate.created_at >= week_ago
        ).count()

        # Average score
        avg_score = self.db.query(func.avg(Candidate.overall_score)).filter(
            Candidate.team_id == self.team_id,
            Candidate.overall_score.isnot(None)
        ).scalar() or 0

        # By status
        by_status = dict(
            self.db.query(Candidate.status, func.count(Candidate.id))
            .filter(Candidate.team_id == self.team_id)
            .group_by(Candidate.status)
            .all()
        )

        # By stage
        by_stage = dict(
            self.db.query(Candidate.stage, func.count(Candidate.id))
            .filter(Candidate.team_id == self.team_id)
            .group_by(Candidate.stage)
            .all()
        )

        # Top skills
        from collections import Counter
        all_skills = []
        candidates = self.db.query(Candidate).filter(
            Candidate.team_id == self.team_id
        ).all()

        for candidate in candidates:
            if candidate.primary_skills:
                all_skills.extend(candidate.primary_skills)

        skill_counts = Counter(all_skills)
        top_skills = [
            {"skill": skill, "count": count}
            for skill, count in skill_counts.most_common(10)
        ]

        return {
            "total_candidates": total_candidates,
            "new_this_week": new_this_week,
            "avg_score": round(avg_score, 2),
            "by_status": by_status,
            "by_stage": by_stage,
            "top_skills": top_skills
        }

    def get_hiring_funnel(self):
        """Calculate hiring funnel metrics"""

        stages = ["applied", "screening", "phone", "technical", "onsite", "offer", "hired"]
        funnel_data = []

        for stage in stages:
            count = self.db.query(Candidate).filter(
                Candidate.team_id == self.team_id,
                Candidate.stage == stage
            ).count()

            # Calculate average time in stage
            activities = self.db.query(CandidateActivity).join(Candidate).filter(
                Candidate.team_id == self.team_id,
                CandidateActivity.activity_type == "stage_change",
                CandidateActivity.metadata["new_stage"].astext == stage
            ).all()

            avg_time_days = 0
            if activities:
                total_time = sum([
                    (a.created_at - a.candidate.created_at).days
                    for a in activities
                ])
                avg_time_days = total_time / len(activities)

            funnel_data.append({
                "stage": stage,
                "count": count,
                "conversion_rate": 0,  # Calculated below
                "avg_time_days": round(avg_time_days, 1)
            })

        # Calculate conversion rates
        for i in range(len(funnel_data)):
            if i == 0:
                funnel_data[i]["conversion_rate"] = 100.0
            else:
                prev_count = funnel_data[i-1]["count"]
                if prev_count > 0:
                    funnel_data[i]["conversion_rate"] = round(
                        (funnel_data[i]["count"] / prev_count) * 100, 1
                    )

        return {"stages": funnel_data}

    def get_trends(self, metric: str, period: str, start_date: datetime, end_date: datetime):
        """Get trends over time"""

        # Determine grouping
        if period == "day":
            date_trunc = func.date_trunc('day', Candidate.created_at)
        elif period == "week":
            date_trunc = func.date_trunc('week', Candidate.created_at)
        else:  # month
            date_trunc = func.date_trunc('month', Candidate.created_at)

        if metric == "applications":
            # Count new applications
            results = self.db.query(
                date_trunc.label('date'),
                func.count(Candidate.id).label('value')
            ).filter(
                Candidate.team_id == self.team_id,
                Candidate.created_at.between(start_date, end_date)
            ).group_by('date').order_by('date').all()

        elif metric == "hires":
            # Count hired candidates
            results = self.db.query(
                date_trunc.label('date'),
                func.count(Candidate.id).label('value')
            ).filter(
                Candidate.team_id == self.team_id,
                Candidate.status == "hired",
                Candidate.created_at.between(start_date, end_date)
            ).group_by('date').order_by('date').all()

        elif metric == "avg_score":
            # Average score
            results = self.db.query(
                date_trunc.label('date'),
                func.avg(Candidate.overall_score).label('value')
            ).filter(
                Candidate.team_id == self.team_id,
                Candidate.created_at.between(start_date, end_date)
            ).group_by('date').order_by('date').all()

        series = [
            {
                "date": str(r.date),
                "value": round(float(r.value), 2) if r.value else 0
            }
            for r in results
        ]

        return {"series": series}

    def get_source_effectiveness(self):
        """Analyze effectiveness of candidate sources"""

        sources = self.db.query(
            Candidate.source,
            func.count(Candidate.id).label('total'),
            func.sum(func.cast(Candidate.status == "hired", Integer)).label('hired'),
            func.avg(Candidate.overall_score).label('avg_score')
        ).filter(
            Candidate.team_id == self.team_id,
            Candidate.source.isnot(None)
        ).group_by(Candidate.source).all()

        results = []
        for source in sources:
            conversion_rate = 0
            if source.total > 0:
                conversion_rate = round((source.hired / source.total) * 100, 1)

            results.append({
                "source": source.source,
                "total": source.total,
                "hired": source.hired or 0,
                "conversion_rate": conversion_rate,
                "avg_score": round(source.avg_score, 1) if source.avg_score else 0
            })

        return {"sources": results}

    def get_team_performance(self):
        """Get GitHub-based team performance metrics"""

        from app.models.candidate_github_profile import CandidateGitHubProfile

        # Aggregate all GitHub profiles
        profiles = self.db.query(CandidateGitHubProfile).join(Candidate).filter(
            Candidate.team_id == self.team_id
        ).all()

        total_commits = sum(p.total_commits or 0 for p in profiles)
        total_repos = sum(p.public_repos or 0 for p in profiles)

        # Language breakdown
        from collections import Counter
        all_languages = {}
        for profile in profiles:
            if profile.languages:
                for lang, percentage in profile.languages.items():
                    all_languages[lang] = all_languages.get(lang, 0) + percentage

        # Normalize percentages
        total = sum(all_languages.values())
        if total > 0:
            language_breakdown = {
                lang: round((count / total) * 100, 1)
                for lang, count in all_languages.items()
            }
        else:
            language_breakdown = {}

        # Top contributors
        top_contributors = sorted(
            profiles,
            key=lambda p: p.total_commits or 0,
            reverse=True
        )[:10]

        return {
            "total_commits": total_commits,
            "total_repos": total_repos,
            "language_breakdown": language_breakdown,
            "top_contributors": [
                {
                    "name": p.candidate.full_name,
                    "commits": p.total_commits,
                    "repos": p.public_repos,
                    "stars": p.total_stars
                }
                for p in top_contributors if p.candidate
            ]
        }
```

---

## Analytics API Endpoints

**File: `app/api/v1/analytics.py`**

```python
from fastapi import APIRouter, Depends, Query
from app.services.analytics_service import AnalyticsService
from datetime import datetime, timedelta

router = APIRouter(prefix="/analytics", tags=["analytics"])

@router.get("/overview")
async def get_overview(
    start_date: datetime = Query(None),
    end_date: datetime = Query(None),
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get overview dashboard metrics"""

    analytics = AnalyticsService(db, current_user.team_id)
    metrics = analytics.get_overview_metrics(start_date, end_date)

    return {
        "success": True,
        "data": metrics
    }

@router.get("/funnel")
async def get_funnel(
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get hiring funnel metrics"""

    analytics = AnalyticsService(db, current_user.team_id)
    funnel = analytics.get_hiring_funnel()

    return {
        "success": True,
        "data": funnel
    }

@router.get("/trends")
async def get_trends(
    metric: str = Query(...),
    period: str = Query("week"),
    start_date: datetime = Query(None),
    end_date: datetime = Query(None),
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get trends over time"""

    if not start_date:
        start_date = datetime.now() - timedelta(days=30)
    if not end_date:
        end_date = datetime.now()

    analytics = AnalyticsService(db, current_user.team_id)
    trends = analytics.get_trends(metric, period, start_date, end_date)

    return {
        "success": True,
        "data": trends
    }

@router.get("/sources")
async def get_sources(
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get source effectiveness"""

    analytics = AnalyticsService(db, current_user.team_id)
    sources = analytics.get_source_effectiveness()

    return {
        "success": True,
        "data": sources
    }

@router.get("/team-performance")
async def get_team_performance(
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get team performance metrics"""

    analytics = AnalyticsService(db, current_user.team_id)
    performance = analytics.get_team_performance()

    return {
        "success": True,
        "data": performance
    }
```

---

## Frontend Analytics Dashboard

**File: `frontend/src/pages/AnalyticsPage.tsx`**

```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { getAnalyticsOverview, getFunnel, getTrends } from '../services/analyticsService';
import { MetricCard } from '../components/analytics/MetricCard';
import { TrendChart } from '../components/analytics/TrendChart';
import { FunnelChart } from '../components/analytics/FunnelChart';
import { SkillsChart } from '../components/analytics/SkillsChart';
import { TrendingUp, Users, Star, CheckCircle } from 'lucide-react';

export const AnalyticsPage: React.FC = () => {
  const { data: overview } = useQuery({
    queryKey: ['analytics-overview'],
    queryFn: getAnalyticsOverview
  });

  const { data: funnel } = useQuery({
    queryKey: ['analytics-funnel'],
    queryFn: getFunnel
  });

  const { data: trends } = useQuery({
    queryKey: ['analytics-trends'],
    queryFn: () => getTrends('applications', 'week')
  });

  if (!overview) return <div>Loading...</div>;

  const metrics = overview.data;

  return (
    <div className="min-h-screen bg-gray-50 p-6">
      <div className="max-w-7xl mx-auto">
        <h1 className="text-2xl font-bold text-gray-900 mb-6">Analytics</h1>

        {/* Metric Cards */}
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-6">
          <MetricCard
            title="Total Candidates"
            value={metrics.total_candidates}
            icon={<Users className="w-6 h-6 text-blue-600" />}
            trend={`+${metrics.new_this_week} this week`}
          />

          <MetricCard
            title="Average Score"
            value={metrics.avg_score.toFixed(1)}
            icon={<Star className="w-6 h-6 text-yellow-600" />}
          />

          <MetricCard
            title="In Progress"
            value={metrics.by_status['reviewing'] || 0}
            icon={<TrendingUp className="w-6 h-6 text-orange-600" />}
          />

          <MetricCard
            title="Hired"
            value={metrics.by_status['hired'] || 0}
            icon={<CheckCircle className="w-6 h-6 text-green-600" />}
          />
        </div>

        {/* Charts */}
        <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
          {/* Trends Chart */}
          <div className="bg-white rounded-lg shadow p-6">
            <h3 className="text-lg font-semibold mb-4">Application Trends</h3>
            {trends && <TrendChart data={trends.data.series} />}
          </div>

          {/* Skills Distribution */}
          <div className="bg-white rounded-lg shadow p-6">
            <h3 className="text-lg font-semibold mb-4">Top Skills</h3>
            <SkillsChart data={metrics.top_skills} />
          </div>
        </div>

        {/* Hiring Funnel */}
        <div className="bg-white rounded-lg shadow p-6">
          <h3 className="text-lg font-semibold mb-4">Hiring Funnel</h3>
          {funnel && <FunnelChart data={funnel.data.stages} />}
        </div>
      </div>
    </div>
  );
};
```

### Chart Components

**File: `frontend/src/components/analytics/TrendChart.tsx`**

```typescript
import React from 'react';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

interface TrendChartProps {
  data: Array<{ date: string; value: number }>;
}

export const TrendChart: React.FC<TrendChartProps> = ({ data }) => {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis
          dataKey="date"
          tickFormatter={(date) => new Date(date).toLocaleDateString()}
        />
        <YAxis />
        <Tooltip
          labelFormatter={(date) => new Date(date).toLocaleDateString()}
        />
        <Line
          type="monotone"
          dataKey="value"
          stroke="#3b82f6"
          strokeWidth={2}
          dot={{ fill: '#3b82f6' }}
        />
      </LineChart>
    </ResponsiveContainer>
  );
};
```

**File: `frontend/src/components/analytics/FunnelChart.tsx`**

```typescript
import React from 'react';
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, Cell } from 'recharts';

interface FunnelChartProps {
  data: Array<{
    stage: string;
    count: number;
    conversion_rate: number;
  }>;
}

const COLORS = ['#3b82f6', '#6366f1', '#8b5cf6', '#a855f7', '#c026d3', '#db2777', '#e11d48'];

export const FunnelChart: React.FC<FunnelChartProps> = ({ data }) => {
  return (
    <ResponsiveContainer width="100%" height={400}>
      <BarChart data={data} layout="vertical">
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis type="number" />
        <YAxis dataKey="stage" type="category" width={100} />
        <Tooltip
          content={({ payload }) => {
            if (!payload || !payload[0]) return null;
            const data = payload[0].payload;
            return (
              <div className="bg-white p-3 border rounded shadow-lg">
                <p className="font-semibold capitalize">{data.stage}</p>
                <p className="text-sm">Count: {data.count}</p>
                <p className="text-sm">Conversion: {data.conversion_rate}%</p>
                <p className="text-sm">Avg Time: {data.avg_time_days} days</p>
              </div>
            );
          }}
        />
        <Bar dataKey="count" radius={[0, 8, 8, 0]}>
          {data.map((entry, index) => (
            <Cell key={`cell-${index}`} fill={COLORS[index % COLORS.length]} />
          ))}
        </Bar>
      </BarChart>
    </ResponsiveContainer>
  );
};
```

---

## Export Functionality

**File: `app/api/v1/candidates.py` (add endpoint)**

```python
from fastapi.responses import StreamingResponse
import csv
import io

@router.get("/export")
async def export_candidates(
    format: str = Query("csv"),
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Export candidates to CSV or Excel"""

    candidates = db.query(Candidate).filter(
        Candidate.team_id == current_user.team_id
    ).all()

    if format == "csv":
        # Create CSV
        output = io.StringIO()
        writer = csv.writer(output)

        # Header
        writer.writerow([
            'Name', 'Email', 'Phone', 'Location', 'Status', 'Stage',
            'Score', 'Experience (years)', 'Education', 'Skills', 'Source'
        ])

        # Rows
        for candidate in candidates:
            writer.writerow([
                candidate.full_name,
                candidate.email,
                candidate.phone,
                candidate.location,
                candidate.status,
                candidate.stage,
                candidate.overall_score,
                candidate.years_of_experience,
                candidate.education_level,
                ', '.join(candidate.primary_skills or []),
                candidate.source
            ])

        output.seek(0)

        return StreamingResponse(
            iter([output.getvalue()]),
            media_type="text/csv",
            headers={
                "Content-Disposition": f"attachment; filename=candidates_{datetime.now().strftime('%Y%m%d')}.csv"
            }
        )
```

This analytics implementation provides comprehensive insights into the hiring process with visual dashboards and export capabilities.
