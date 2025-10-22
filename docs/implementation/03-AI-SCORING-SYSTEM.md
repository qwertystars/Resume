# AI Scoring System Implementation Guide

## Overview
This guide covers implementing the intelligent scoring engine that evaluates candidates based on resume data, GitHub activity, LinkedIn profile, skills matching, and experience.

---

## Scoring Architecture

```
Input Data → Feature Extraction → Rule Evaluation → Weight Application → Score Aggregation → Percentile Ranking
```

---

## Backend Implementation

### 1. Scoring Engine Core

**File: `app/services/scoring_engine.py`**

```python
from typing import Dict, List, Any
from app.models.candidate import Candidate
from app.models.scoring_rules import ScoringRule
from sqlalchemy.orm import Session
import re

class ScoringEngine:
    def __init__(self, db: Session):
        self.db = db

    def calculate_score(self, candidate: Candidate) -> Dict[str, float]:
        """Calculate all scores for a candidate"""

        scores = {
            "resume_score": self.calculate_resume_score(candidate),
            "github_score": self.calculate_github_score(candidate),
            "linkedin_score": self.calculate_linkedin_score(candidate),
            "skills_score": self.calculate_skills_score(candidate),
            "experience_score": self.calculate_experience_score(candidate),
            "education_score": self.calculate_education_score(candidate)
        }

        # Calculate weighted overall score
        overall_score = self.calculate_overall_score(scores)
        scores["overall_score"] = overall_score

        return scores

    def calculate_resume_score(self, candidate: Candidate) -> float:
        """Score based on resume quality and completeness"""
        score = 0.0

        if not candidate.resumes:
            return 0.0

        resume = candidate.resumes[0]  # Get latest resume

        # Resume was successfully parsed
        if resume.status == "completed":
            score += 20.0

        parsed_data = resume.parsed_data or {}

        # Contact information completeness
        contact = parsed_data.get("contact", {})
        if contact.get("email"):
            score += 5.0
        if contact.get("phone"):
            score += 5.0
        if contact.get("location"):
            score += 5.0

        # Experience section
        experiences = parsed_data.get("experience", [])
        if experiences:
            score += 15.0
            # Bonus for detailed descriptions
            if any(len(exp.get("description", "")) > 100 for exp in experiences):
                score += 10.0

        # Education section
        education = parsed_data.get("education", [])
        if education:
            score += 10.0

        # Skills section
        skills = parsed_data.get("skills", [])
        if len(skills) > 0:
            score += 10.0
        if len(skills) >= 5:
            score += 5.0
        if len(skills) >= 10:
            score += 5.0

        # Summary/objective
        if parsed_data.get("summary"):
            score += 10.0

        return min(score, 100.0)

    def calculate_github_score(self, candidate: Candidate) -> float:
        """Score based on GitHub activity"""
        score = 0.0

        if not candidate.github_profile:
            return 0.0

        profile = candidate.github_profile

        # Account age (older = more credible)
        if profile.github_created_at:
            account_age_years = (datetime.now() - profile.github_created_at).days / 365
            if account_age_years >= 5:
                score += 15.0
            elif account_age_years >= 3:
                score += 10.0
            elif account_age_years >= 1:
                score += 5.0

        # Public repositories
        if profile.public_repos >= 50:
            score += 15.0
        elif profile.public_repos >= 20:
            score += 10.0
        elif profile.public_repos >= 10:
            score += 5.0

        # Stars received
        if profile.total_stars >= 500:
            score += 20.0
        elif profile.total_stars >= 100:
            score += 15.0
        elif profile.total_stars >= 50:
            score += 10.0
        elif profile.total_stars >= 10:
            score += 5.0

        # Contributions in last year
        if profile.contributions_last_year >= 1000:
            score += 20.0
        elif profile.contributions_last_year >= 500:
            score += 15.0
        elif profile.contributions_last_year >= 250:
            score += 10.0
        elif profile.contributions_last_year >= 100:
            score += 5.0

        # Followers
        if profile.followers >= 100:
            score += 10.0
        elif profile.followers >= 50:
            score += 5.0

        # Current streak bonus
        if profile.current_streak >= 30:
            score += 10.0
        elif profile.current_streak >= 7:
            score += 5.0

        # Language diversity
        languages = profile.languages or {}
        if len(languages) >= 5:
            score += 5.0

        return min(score, 100.0)

    def calculate_linkedin_score(self, candidate: Candidate) -> float:
        """Score based on LinkedIn profile"""
        score = 0.0

        if not candidate.linkedin_profile:
            return 0.0

        profile = candidate.linkedin_profile

        # Connection count
        if profile.connections >= 500:
            score += 20.0
        elif profile.connections >= 250:
            score += 15.0
        elif profile.connections >= 100:
            score += 10.0
        elif profile.connections >= 50:
            score += 5.0

        # Recommendations
        if profile.recommendations >= 10:
            score += 15.0
        elif profile.recommendations >= 5:
            score += 10.0
        elif profile.recommendations >= 3:
            score += 5.0

        # Experience entries
        experiences = profile.experience or []
        if len(experiences) >= 5:
            score += 15.0
        elif len(experiences) >= 3:
            score += 10.0
        elif len(experiences) >= 1:
            score += 5.0

        # Education entries
        education = profile.education or []
        if len(education) >= 2:
            score += 10.0
        elif len(education) >= 1:
            score += 5.0

        # Skills listed
        skills = profile.skills or []
        if len(skills) >= 20:
            score += 15.0
        elif len(skills) >= 10:
            score += 10.0
        elif len(skills) >= 5:
            score += 5.0

        # Headline quality
        if profile.headline and len(profile.headline) > 30:
            score += 10.0

        # Certifications
        certifications = profile.certifications or []
        if len(certifications) >= 3:
            score += 10.0
        elif len(certifications) >= 1:
            score += 5.0

        return min(score, 100.0)

    def calculate_skills_score(self, candidate: Candidate) -> float:
        """Score based on skills matching"""
        score = 0.0

        candidate_skills = set(s.lower() for s in (candidate.primary_skills or []))

        if not candidate_skills:
            return 0.0

        # Get team's scoring rules for skills
        rules = self.db.query(ScoringRule).filter(
            ScoringRule.team_id == candidate.team_id,
            ScoringRule.rule_type == "skills_match",
            ScoringRule.is_active == True
        ).all()

        for rule in rules:
            conditions = rule.conditions
            required_skills = set(s.lower() for s in conditions.get("keywords", []))

            # Calculate match percentage
            if required_skills:
                matched = candidate_skills.intersection(required_skills)
                match_percentage = len(matched) / len(required_skills)

                if match_percentage >= 0.8:
                    score += rule.points * rule.weight
                elif match_percentage >= 0.6:
                    score += (rule.points * 0.7) * rule.weight
                elif match_percentage >= 0.4:
                    score += (rule.points * 0.4) * rule.weight

        # Fallback: general skill count
        if not rules:
            skill_count = len(candidate_skills)
            if skill_count >= 20:
                score = 100.0
            elif skill_count >= 15:
                score = 80.0
            elif skill_count >= 10:
                score = 60.0
            elif skill_count >= 5:
                score = 40.0
            else:
                score = 20.0

        return min(score, 100.0)

    def calculate_experience_score(self, candidate: Candidate) -> float:
        """Score based on years of experience"""
        score = 0.0

        years = candidate.years_of_experience or 0

        # Check custom rules
        rules = self.db.query(ScoringRule).filter(
            ScoringRule.team_id == candidate.team_id,
            ScoringRule.rule_type == "experience_years",
            ScoringRule.is_active == True
        ).all()

        for rule in rules:
            conditions = rule.conditions
            min_years = conditions.get("min_years", 0)
            max_years = conditions.get("max_years", 999)

            if min_years <= years <= max_years:
                score += rule.points * rule.weight

        # Fallback: standard experience scoring
        if not rules:
            if years >= 10:
                score = 100.0
            elif years >= 7:
                score = 90.0
            elif years >= 5:
                score = 80.0
            elif years >= 3:
                score = 60.0
            elif years >= 1:
                score = 40.0
            else:
                score = 20.0

        return min(score, 100.0)

    def calculate_education_score(self, candidate: Candidate) -> float:
        """Score based on education level"""
        education_level = candidate.education_level

        if not education_level:
            return 0.0

        scores = {
            "phd": 100.0,
            "master": 85.0,
            "bachelor": 70.0,
            "associate": 50.0,
            "high_school": 30.0
        }

        return scores.get(education_level, 0.0)

    def calculate_overall_score(self, scores: Dict[str, float]) -> float:
        """Calculate weighted overall score"""

        # Default weights
        weights = {
            "resume_score": 0.15,
            "github_score": 0.25,
            "linkedin_score": 0.10,
            "skills_score": 0.30,
            "experience_score": 0.15,
            "education_score": 0.05
        }

        overall = sum(
            scores.get(key, 0) * weight
            for key, weight in weights.items()
        )

        return round(overall, 2)

    def calculate_percentile_rank(self, candidate: Candidate) -> float:
        """Calculate candidate's percentile rank"""

        # Get all candidates in same team
        all_candidates = self.db.query(Candidate).filter(
            Candidate.team_id == candidate.team_id,
            Candidate.overall_score.isnot(None)
        ).all()

        if not all_candidates:
            return 0.0

        # Count candidates with lower scores
        lower_count = sum(
            1 for c in all_candidates
            if c.overall_score < candidate.overall_score
        )

        percentile = (lower_count / len(all_candidates)) * 100
        return round(percentile, 1)
```

### 2. Scoring API Endpoints

**File: `app/api/v1/scoring.py`**

```python
from fastapi import APIRouter, Depends, HTTPException
from app.services.scoring_engine import ScoringEngine
from app.models.candidate import Candidate
from app.models.score_history import ScoreHistory

router = APIRouter(prefix="/scoring", tags=["scoring"])

@router.post("/calculate/{candidate_id}")
async def calculate_score(
    candidate_id: str,
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Calculate or recalculate scores for candidate"""

    candidate = db.query(Candidate).filter(
        Candidate.id == candidate_id,
        Candidate.team_id == current_user.team_id
    ).first()

    if not candidate:
        raise HTTPException(404, "Candidate not found")

    # Calculate scores
    engine = ScoringEngine(db)
    scores = engine.calculate_score(candidate)

    # Save score history
    for score_type, score_value in scores.items():
        if score_type != "overall_score":
            continue

        history = ScoreHistory(
            candidate_id=candidate.id,
            score_type=score_type,
            score_value=score_value,
            previous_value=getattr(candidate, score_type, None),
            calculated_by="system"
        )
        db.add(history)

    # Update candidate
    for score_type, score_value in scores.items():
        setattr(candidate, score_type, score_value)

    db.commit()

    # Calculate percentile
    percentile = engine.calculate_percentile_rank(candidate)

    return {
        "success": True,
        "data": {
            "scores": scores,
            "percentile": percentile,
            "breakdown": {
                "resume": scores["resume_score"],
                "github": scores["github_score"],
                "linkedin": scores["linkedin_score"],
                "skills": scores["skills_score"],
                "experience": scores["experience_score"],
                "education": scores["education_score"]
            }
        }
    }

@router.get("/history/{candidate_id}")
async def get_score_history(
    candidate_id: str,
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get score history for candidate"""

    history = db.query(ScoreHistory).filter(
        ScoreHistory.candidate_id == candidate_id
    ).order_by(ScoreHistory.calculated_at.desc()).all()

    return {
        "success": True,
        "data": {
            "history": [h.to_dict() for h in history]
        }
    }

@router.get("/rules")
async def get_scoring_rules(
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get scoring rules for team"""

    rules = db.query(ScoringRule).filter(
        ScoringRule.team_id == current_user.team_id
    ).all()

    return {
        "success": True,
        "data": {
            "rules": [r.to_dict() for r in rules]
        }
    }

@router.post("/rules")
async def create_scoring_rule(
    rule_data: dict,
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Create new scoring rule"""

    rule = ScoringRule(
        team_id=current_user.team_id,
        created_by=current_user.id,
        **rule_data
    )

    db.add(rule)
    db.commit()

    return {
        "success": True,
        "data": {"rule": rule.to_dict()}
    }
```

### 3. Celery Task

**File: `app/workers/scoring_tasks.py`**

```python
from celery import shared_task
from app.services.scoring_engine import ScoringEngine
from app.core.database import SessionLocal
import logging

logger = logging.getLogger(__name__)

@shared_task
def calculate_score_task(candidate_id: str):
    """Calculate scores in background"""
    db = SessionLocal()

    try:
        candidate = db.query(Candidate).filter(Candidate.id == candidate_id).first()

        if not candidate:
            logger.error(f"Candidate {candidate_id} not found")
            return

        engine = ScoringEngine(db)
        scores = engine.calculate_score(candidate)

        # Update candidate
        for score_type, score_value in scores.items():
            setattr(candidate, score_type, score_value)

        db.commit()

        logger.info(f"Calculated scores for candidate {candidate_id}: {scores['overall_score']}")

    except Exception as e:
        logger.error(f"Error calculating scores for {candidate_id}: {str(e)}")
        raise

    finally:
        db.close()

@shared_task
def bulk_recalculate_scores(team_id: str):
    """Recalculate scores for all candidates in team"""
    db = SessionLocal()

    try:
        candidates = db.query(Candidate).filter(Candidate.team_id == team_id).all()

        for candidate in candidates:
            calculate_score_task.delay(str(candidate.id))

        logger.info(f"Queued score recalculation for {len(candidates)} candidates")

    finally:
        db.close()
```

---

## Frontend Implementation

### Score Display Component

**File: `frontend/src/components/scoring/ScoreDisplay.tsx`**

```typescript
import React from 'react';
import { TrendingUp, Award } from 'lucide-react';

interface ScoreDisplayProps {
  overall: number;
  breakdown: {
    resume: number;
    github: number;
    linkedin: number;
    skills: number;
    experience: number;
    education: number;
  };
  percentile: number;
}

export const ScoreDisplay: React.FC<ScoreDisplayProps> = ({
  overall,
  breakdown,
  percentile
}) => {
  const getScoreColor = (score: number) => {
    if (score >= 80) return 'text-green-600';
    if (score >= 60) return 'text-yellow-600';
    return 'text-red-600';
  };

  const getScoreBgColor = (score: number) => {
    if (score >= 80) return 'bg-green-100';
    if (score >= 60) return 'bg-yellow-100';
    return 'bg-red-100';
  };

  return (
    <div className="bg-white rounded-lg shadow-lg p-6">
      {/* Overall Score */}
      <div className="text-center mb-8">
        <div className="inline-flex items-center justify-center w-32 h-32 rounded-full bg-gradient-to-br from-blue-500 to-purple-600 text-white mb-4">
          <div className="text-center">
            <div className="text-4xl font-bold">{overall}</div>
            <div className="text-sm opacity-90">/ 100</div>
          </div>
        </div>

        <div className="flex items-center justify-center space-x-2 text-gray-600">
          <Award className="w-5 h-5" />
          <span className="text-lg">
            Top {100 - percentile}% of candidates
          </span>
        </div>
      </div>

      {/* Score Breakdown */}
      <div className="space-y-4">
        <h4 className="font-semibold text-gray-900 mb-3">Score Breakdown</h4>

        {Object.entries(breakdown).map(([category, score]) => (
          <div key={category}>
            <div className="flex justify-between items-center mb-2">
              <span className="text-sm font-medium text-gray-700 capitalize">
                {category.replace('_', ' ')}
              </span>
              <span className={`text-sm font-bold ${getScoreColor(score)}`}>
                {score}/100
              </span>
            </div>
            <div className="h-2 bg-gray-200 rounded-full overflow-hidden">
              <div
                className={`h-full transition-all duration-500 ${
                  score >= 80 ? 'bg-green-500' :
                  score >= 60 ? 'bg-yellow-500' :
                  'bg-red-500'
                }`}
                style={{ width: `${score}%` }}
              />
            </div>
          </div>
        ))}
      </div>

      {/* Score Label */}
      <div className="mt-6 text-center">
        <span className={`inline-block px-4 py-2 rounded-full text-sm font-semibold ${getScoreBgColor(overall)} ${getScoreColor(overall)}`}>
          {overall >= 80 ? 'Excellent Match' :
           overall >= 60 ? 'Good Match' :
           overall >= 40 ? 'Fair Match' :
           'Below Average'}
        </span>
      </div>
    </div>
  );
};
```

---

## Machine Learning Enhancement (Future)

For advanced scoring, consider training ML models:

```python
from sklearn.ensemble import RandomForestRegressor
import pandas as pd

class MLScoringEngine:
    def __init__(self):
        self.model = RandomForestRegressor()

    def train(self, training_data: pd.DataFrame):
        """Train model on historical hiring data"""
        features = ['years_experience', 'github_stars', 'education_level_numeric']
        X = training_data[features]
        y = training_data['hired']  # 1 if hired, 0 if not

        self.model.fit(X, y)

    def predict_hire_probability(self, candidate_features: dict) -> float:
        """Predict probability of hiring"""
        X = pd.DataFrame([candidate_features])
        probability = self.model.predict_proba(X)[0][1]
        return probability * 100
```

---

## Testing

```python
import pytest
from app.services.scoring_engine import ScoringEngine

def test_resume_score_calculation():
    candidate = create_test_candidate()
    engine = ScoringEngine(db)

    score = engine.calculate_resume_score(candidate)
    assert 0 <= score <= 100

def test_overall_score_aggregation():
    candidate = create_test_candidate()
    engine = ScoringEngine(db)

    scores = engine.calculate_score(candidate)
    assert "overall_score" in scores
    assert 0 <= scores["overall_score"] <= 100
```

This scoring system provides intelligent, customizable candidate evaluation with transparent breakdown and historical tracking.
