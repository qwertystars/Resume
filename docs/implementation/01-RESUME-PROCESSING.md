# Resume Processing Implementation Guide

## Overview
This guide covers building the complete resume processing pipeline: file upload, storage, parsing (PDF/DOCX/TXT), field extraction, and structured data output.

---

## Architecture

```
Upload → Validate → Store (S3) → Queue Parse Job → Extract Text → Parse Fields → Save to DB → Calculate Score
```

---

## Backend Implementation

### 1. File Upload Handler

**File: `app/api/v1/resumes.py`**

```python
from fastapi import APIRouter, UploadFile, File, Depends, HTTPException
from app.services.resume_parser import ResumeParserService
from app.services.storage import StorageService
from app.models.resume import Resume
from app.core.security import get_current_user

router = APIRouter(prefix="/resumes", tags=["resumes"])

ALLOWED_EXTENSIONS = {".pdf", ".docx", ".doc", ".txt"}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

@router.post("/upload")
async def upload_resume(
    file: UploadFile = File(...),
    candidate_id: str = None,
    current_user = Depends(get_current_user),
    db = Depends(get_db),
    storage: StorageService = Depends(get_storage_service)
):
    # Validate file
    if not any(file.filename.lower().endswith(ext) for ext in ALLOWED_EXTENSIONS):
        raise HTTPException(400, "Invalid file type")

    # Read and validate size
    content = await file.read()
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(400, f"File too large. Max size: {MAX_FILE_SIZE} bytes")

    # Upload to S3
    file_path = f"resumes/{current_user.team_id}/{uuid.uuid4()}/{file.filename}"
    storage_url = await storage.upload(file_path, content, file.content_type)

    # Create resume record
    resume = Resume(
        candidate_id=candidate_id or await create_candidate_from_resume(db, current_user),
        file_name=file.filename,
        file_size=len(content),
        file_type=get_file_extension(file.filename),
        storage_path=file_path,
        storage_url=storage_url,
        status="pending",
        uploaded_by=current_user.id
    )

    db.add(resume)
    await db.commit()

    # Queue parsing job
    from app.workers.resume_tasks import parse_resume_task
    parse_resume_task.delay(str(resume.id))

    return {
        "success": True,
        "data": {"resume": resume.to_dict()}
    }
```

### 2. Storage Service

**File: `app/services/storage.py`**

```python
import boto3
from botocore.config import Config
from app.core.config import settings

class StorageService:
    def __init__(self):
        self.s3_client = boto3.client(
            's3',
            aws_access_key_id=settings.AWS_ACCESS_KEY_ID,
            aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY,
            region_name=settings.AWS_REGION,
            config=Config(signature_version='s3v4')
        )
        self.bucket = settings.S3_BUCKET_NAME

    async def upload(self, file_path: str, content: bytes, content_type: str) -> str:
        """Upload file to S3 and return URL"""
        self.s3_client.put_object(
            Bucket=self.bucket,
            Key=file_path,
            Body=content,
            ContentType=content_type
        )
        return f"https://{self.bucket}.s3.amazonaws.com/{file_path}"

    async def download(self, file_path: str) -> bytes:
        """Download file from S3"""
        response = self.s3_client.get_object(Bucket=self.bucket, Key=file_path)
        return response['Body'].read()

    async def generate_presigned_url(self, file_path: str, expiration=3600) -> str:
        """Generate temporary download URL"""
        return self.s3_client.generate_presigned_url(
            'get_object',
            Params={'Bucket': self.bucket, 'Key': file_path},
            ExpiresIn=expiration
        )

    async def delete(self, file_path: str):
        """Delete file from S3"""
        self.s3_client.delete_object(Bucket=self.bucket, Key=file_path)
```

### 3. Resume Parser Service

**File: `app/services/resume_parser.py`**

```python
import re
import io
from typing import Dict, Any, List
from PyPDF2 import PdfReader
from docx import Document
import pytesseract
from PIL import Image
import spacy

class ResumeParserService:
    def __init__(self):
        # Load spaCy NER model
        self.nlp = spacy.load("en_core_web_sm")

    def extract_text(self, file_content: bytes, file_type: str) -> str:
        """Extract text from various file formats"""
        if file_type == "pdf":
            return self._extract_text_from_pdf(file_content)
        elif file_type in ["docx", "doc"]:
            return self._extract_text_from_docx(file_content)
        elif file_type == "txt":
            return file_content.decode('utf-8')
        else:
            raise ValueError(f"Unsupported file type: {file_type}")

    def _extract_text_from_pdf(self, content: bytes) -> str:
        """Extract text from PDF"""
        text = ""
        pdf_reader = PdfReader(io.BytesIO(content))

        for page in pdf_reader.pages:
            page_text = page.extract_text()
            text += page_text + "\n"

        # If no text extracted (scanned PDF), use OCR
        if not text.strip():
            text = self._ocr_pdf(content)

        return text

    def _extract_text_from_docx(self, content: bytes) -> str:
        """Extract text from DOCX"""
        doc = Document(io.BytesIO(content))
        return "\n".join([paragraph.text for paragraph in doc.paragraphs])

    def parse_resume(self, text: str) -> Dict[str, Any]:
        """Parse resume text and extract structured data"""
        return {
            "contact": self._extract_contact(text),
            "experience": self._extract_experience(text),
            "education": self._extract_education(text),
            "skills": self._extract_skills(text),
            "certifications": self._extract_certifications(text),
            "summary": self._extract_summary(text)
        }

    def _extract_contact(self, text: str) -> Dict[str, str]:
        """Extract contact information"""
        contact = {}

        # Email regex
        email_pattern = r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
        emails = re.findall(email_pattern, text)
        if emails:
            contact["email"] = emails[0]

        # Phone regex (US format)
        phone_pattern = r'(\+?\d{1,2}[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}'
        phones = re.findall(phone_pattern, text)
        if phones:
            contact["phone"] = "".join(phones[0])

        # Name (first few words before email/phone)
        lines = text.split('\n')
        for line in lines[:5]:
            if line.strip() and len(line.split()) <= 4:
                contact["name"] = line.strip()
                break

        # Location
        location_pattern = r'([A-Z][a-z]+(?:\s+[A-Z][a-z]+)*),\s*([A-Z]{2})'
        locations = re.findall(location_pattern, text)
        if locations:
            contact["location"] = f"{locations[0][0]}, {locations[0][1]}"

        # LinkedIn
        linkedin_pattern = r'linkedin\.com/in/([a-zA-Z0-9-]+)'
        linkedin = re.search(linkedin_pattern, text)
        if linkedin:
            contact["linkedin"] = f"https://linkedin.com/in/{linkedin.group(1)}"

        # GitHub
        github_pattern = r'github\.com/([a-zA-Z0-9-]+)'
        github = re.search(github_pattern, text)
        if github:
            contact["github"] = f"https://github.com/{github.group(1)}"

        return contact

    def _extract_experience(self, text: str) -> List[Dict[str, str]]:
        """Extract work experience"""
        experiences = []

        # Find experience section
        exp_section = self._extract_section(text, ["experience", "work history", "employment"])

        if not exp_section:
            return experiences

        # Split by job entries (look for dates)
        date_pattern = r'(\d{4})\s*[-–—]\s*(\d{4}|present|current)'
        entries = re.split(date_pattern, exp_section, flags=re.IGNORECASE)

        for i in range(0, len(entries)-2, 3):
            text_block = entries[i]
            start_year = entries[i+1]
            end_year = entries[i+2]

            lines = [l.strip() for l in text_block.split('\n') if l.strip()]

            if len(lines) >= 2:
                experience = {
                    "title": lines[-2] if len(lines) >= 2 else "",
                    "company": lines[-1] if len(lines) >= 1 else "",
                    "start_date": start_year,
                    "end_date": end_year.lower(),
                    "description": "\n".join(lines[:-2]) if len(lines) > 2 else ""
                }
                experiences.append(experience)

        return experiences

    def _extract_education(self, text: str) -> List[Dict[str, str]]:
        """Extract education"""
        education_list = []

        edu_section = self._extract_section(text, ["education", "academic", "qualifications"])

        if not edu_section:
            return education_list

        # Degree patterns
        degree_pattern = r'(Bachelor|Master|PhD|Associate|B\.S\.|M\.S\.|B\.A\.|M\.A\.).*?(\d{4})?'

        matches = re.finditer(degree_pattern, edu_section, re.IGNORECASE)

        for match in matches:
            # Extract context around the match
            start = max(0, match.start() - 100)
            end = min(len(edu_section), match.end() + 100)
            context = edu_section[start:end]

            lines = [l.strip() for l in context.split('\n') if l.strip()]

            education = {
                "degree": match.group(0),
                "school": lines[1] if len(lines) > 1 else "",
                "graduation_year": match.group(2) if match.group(2) else "",
                "field": self._extract_field_of_study(context)
            }
            education_list.append(education)

        return education_list

    def _extract_skills(self, text: str) -> List[str]:
        """Extract skills"""
        skills_section = self._extract_section(text, ["skills", "technical skills", "competencies"])

        if not skills_section:
            # Fall back to keyword matching
            return self._extract_skills_by_keywords(text)

        # Common skill delimiters
        skills = re.split(r'[,;•\n]', skills_section)
        skills = [s.strip() for s in skills if s.strip() and len(s.strip()) > 2]

        # Clean up
        skills = [s for s in skills if not any(x in s.lower() for x in ['skill', 'proficient', 'familiar'])]

        return skills[:30]  # Limit to top 30

    def _extract_skills_by_keywords(self, text: str) -> List[str]:
        """Extract skills by matching known keywords"""
        skill_keywords = [
            # Programming Languages
            "Python", "JavaScript", "Java", "C++", "C#", "Ruby", "Go", "Rust", "Swift",
            "Kotlin", "TypeScript", "PHP", "Scala", "R", "MATLAB",

            # Frameworks
            "React", "Angular", "Vue", "Django", "Flask", "FastAPI", "Node.js", "Express",
            "Spring", "Laravel", "Rails", ".NET", "ASP.NET",

            # Databases
            "PostgreSQL", "MySQL", "MongoDB", "Redis", "Elasticsearch", "Oracle", "SQL Server",
            "DynamoDB", "Cassandra", "Neo4j",

            # Cloud & DevOps
            "AWS", "Azure", "GCP", "Docker", "Kubernetes", "Terraform", "Jenkins", "CI/CD",
            "Git", "GitHub", "GitLab", "Linux",

            # Data Science
            "Machine Learning", "Deep Learning", "TensorFlow", "PyTorch", "Pandas", "NumPy",
            "Scikit-learn", "NLP", "Computer Vision",

            # Other
            "REST API", "GraphQL", "Microservices", "Agile", "Scrum", "JIRA"
        ]

        found_skills = []
        text_lower = text.lower()

        for skill in skill_keywords:
            if skill.lower() in text_lower:
                found_skills.append(skill)

        return found_skills

    def _extract_certifications(self, text: str) -> List[str]:
        """Extract certifications"""
        cert_section = self._extract_section(text, ["certification", "licenses", "credentials"])

        if not cert_section:
            return []

        # Common certification patterns
        cert_keywords = ["AWS", "Azure", "GCP", "PMP", "Scrum Master", "CISSP", "CEH", "CompTIA"]

        certifications = []
        for keyword in cert_keywords:
            if keyword.lower() in cert_section.lower():
                certifications.append(keyword)

        return certifications

    def _extract_summary(self, text: str) -> str:
        """Extract professional summary"""
        summary_section = self._extract_section(text, ["summary", "objective", "profile", "about"])

        if summary_section:
            # Return first 500 characters
            return summary_section[:500]

        # Fallback: return first few sentences
        sentences = text.split('.')[:3]
        return '. '.join(sentences) + '.'

    def _extract_section(self, text: str, section_names: List[str]) -> str:
        """Extract a section by header name"""
        text_lower = text.lower()

        for section_name in section_names:
            pattern = rf'\n\s*{section_name}\s*\n'
            match = re.search(pattern, text_lower)

            if match:
                start = match.end()

                # Find next section header
                next_section_pattern = r'\n\s*[A-Z][A-Za-z\s]{3,30}\s*\n'
                next_match = re.search(next_section_pattern, text[start:])

                if next_match:
                    end = start + next_match.start()
                else:
                    end = len(text)

                return text[start:end].strip()

        return ""

    def _extract_field_of_study(self, text: str) -> str:
        """Extract field of study from education context"""
        fields = ["Computer Science", "Engineering", "Business", "Mathematics",
                 "Physics", "Chemistry", "Biology", "Economics", "Finance"]

        text_lower = text.lower()
        for field in fields:
            if field.lower() in text_lower:
                return field

        return ""

    def calculate_years_of_experience(self, experiences: List[Dict]) -> float:
        """Calculate total years of experience"""
        from datetime import datetime

        total_months = 0
        current_year = datetime.now().year

        for exp in experiences:
            start = int(exp.get("start_date", 0))
            end_str = exp.get("end_date", "").lower()

            if "present" in end_str or "current" in end_str:
                end = current_year
            else:
                try:
                    end = int(end_str)
                except:
                    continue

            if start and end:
                total_months += (end - start) * 12

        return round(total_months / 12, 1)

    def classify_education_level(self, education: List[Dict]) -> str:
        """Determine highest education level"""
        if not education:
            return None

        degrees = " ".join([e.get("degree", "").lower() for e in education])

        if any(x in degrees for x in ["phd", "ph.d", "doctorate"]):
            return "phd"
        elif any(x in degrees for x in ["master", "m.s", "m.a", "mba"]):
            return "master"
        elif any(x in degrees for x in ["bachelor", "b.s", "b.a"]):
            return "bachelor"
        elif any(x in degrees for x in ["associate"]):
            return "associate"
        else:
            return "high_school"
```

### 4. Celery Task for Async Processing

**File: `app/workers/resume_tasks.py`**

```python
from celery import shared_task
from app.core.database import SessionLocal
from app.models.resume import Resume
from app.models.candidate import Candidate
from app.services.resume_parser import ResumeParserService
from app.services.storage import StorageService
import logging

logger = logging.getLogger(__name__)

@shared_task(bind=True, max_retries=3)
def parse_resume_task(self, resume_id: str):
    """Background task to parse resume"""
    db = SessionLocal()

    try:
        # Get resume record
        resume = db.query(Resume).filter(Resume.id == resume_id).first()
        if not resume:
            logger.error(f"Resume {resume_id} not found")
            return

        # Update status
        resume.status = "processing"
        db.commit()

        # Download file from S3
        storage = StorageService()
        file_content = await storage.download(resume.storage_path)

        # Extract text
        parser = ResumeParserService()
        raw_text = parser.extract_text(file_content, resume.file_type)

        # Parse structured data
        parsed_data = parser.parse_resume(raw_text)

        # Save to resume
        resume.raw_text = raw_text
        resume.parsed_data = parsed_data
        resume.status = "completed"
        resume.processed_at = datetime.utcnow()

        # Update candidate with parsed data
        if resume.candidate_id:
            candidate = db.query(Candidate).filter(Candidate.id == resume.candidate_id).first()

            if candidate:
                # Update contact info
                contact = parsed_data.get("contact", {})
                candidate.email = candidate.email or contact.get("email")
                candidate.phone = candidate.phone or contact.get("phone")
                candidate.location = candidate.location or contact.get("location")
                candidate.linkedin_url = candidate.linkedin_url or contact.get("linkedin")
                candidate.github_url = candidate.github_url or contact.get("github")

                # Calculate experience
                experiences = parsed_data.get("experience", [])
                candidate.years_of_experience = parser.calculate_years_of_experience(experiences)

                # Classify education
                education = parsed_data.get("education", [])
                candidate.education_level = parser.classify_education_level(education)

                # Extract skills
                skills = parsed_data.get("skills", [])
                candidate.primary_skills = skills

        db.commit()

        # Trigger scoring
        from app.workers.scoring_tasks import calculate_score_task
        calculate_score_task.delay(str(resume.candidate_id))

        logger.info(f"Successfully parsed resume {resume_id}")

    except Exception as e:
        logger.error(f"Error parsing resume {resume_id}: {str(e)}")

        resume.status = "failed"
        resume.error_message = str(e)
        db.commit()

        # Retry
        raise self.retry(exc=e, countdown=60)

    finally:
        db.close()
```

### 5. Bulk Upload Handler

**File: `app/api/v1/resumes.py` (continued)**

```python
@router.post("/bulk-upload")
async def bulk_upload_resumes(
    files: List[UploadFile] = File(...),
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Upload multiple resumes at once"""

    if len(files) > 50:
        raise HTTPException(400, "Maximum 50 files per upload")

    # Create background job record
    job = BackgroundJob(
        team_id=current_user.team_id,
        user_id=current_user.id,
        job_type="bulk_resume_upload",
        status="pending",
        input_data={"file_count": len(files)}
    )
    db.add(job)
    await db.commit()

    # Queue bulk processing task
    from app.workers.resume_tasks import bulk_upload_task
    bulk_upload_task.delay(str(job.id), [f.filename for f in files])

    return {
        "success": True,
        "data": {
            "job_id": str(job.id),
            "total_files": len(files),
            "status": "processing"
        }
    }
```

---

## Frontend Implementation

### 1. Resume Upload Component

**File: `frontend/src/components/resumes/ResumeUpload.tsx`**

```typescript
import React, { useCallback, useState } from 'react';
import { useDropzone } from 'react-dropzone';
import { Upload, File, X, CheckCircle, AlertCircle } from 'lucide-react';
import { uploadResume } from '../../services/resumeService';

interface UploadedFile {
  file: File;
  status: 'pending' | 'uploading' | 'success' | 'error';
  progress: number;
  error?: string;
  resumeId?: string;
}

export const ResumeUpload: React.FC = () => {
  const [files, setFiles] = useState<UploadedFile[]>([]);

  const onDrop = useCallback((acceptedFiles: File[]) => {
    const newFiles = acceptedFiles.map(file => ({
      file,
      status: 'pending' as const,
      progress: 0
    }));

    setFiles(prev => [...prev, ...newFiles]);

    // Start uploading
    newFiles.forEach(uploadFile);
  }, []);

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: {
      'application/pdf': ['.pdf'],
      'application/msword': ['.doc'],
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document': ['.docx'],
      'text/plain': ['.txt']
    },
    maxSize: 10 * 1024 * 1024, // 10MB
    multiple: true
  });

  const uploadFile = async (uploadedFile: UploadedFile) => {
    setFiles(prev => prev.map(f =>
      f.file === uploadedFile.file
        ? { ...f, status: 'uploading' }
        : f
    ));

    try {
      const result = await uploadResume(uploadedFile.file, (progress) => {
        setFiles(prev => prev.map(f =>
          f.file === uploadedFile.file
            ? { ...f, progress }
            : f
        ));
      });

      setFiles(prev => prev.map(f =>
        f.file === uploadedFile.file
          ? { ...f, status: 'success', progress: 100, resumeId: result.data.resume.id }
          : f
      ));
    } catch (error) {
      setFiles(prev => prev.map(f =>
        f.file === uploadedFile.file
          ? { ...f, status: 'error', error: error.message }
          : f
      ));
    }
  };

  const removeFile = (file: File) => {
    setFiles(prev => prev.filter(f => f.file !== file));
  };

  return (
    <div className="space-y-4">
      {/* Dropzone */}
      <div
        {...getRootProps()}
        className={`border-2 border-dashed rounded-lg p-8 text-center cursor-pointer transition-colors
          ${isDragActive ? 'border-blue-500 bg-blue-50' : 'border-gray-300 hover:border-gray-400'}`}
      >
        <input {...getInputProps()} />
        <Upload className="w-12 h-12 mx-auto text-gray-400 mb-4" />
        {isDragActive ? (
          <p className="text-blue-600">Drop the files here...</p>
        ) : (
          <>
            <p className="text-gray-600 mb-2">
              Drag & drop resume files here, or click to select
            </p>
            <p className="text-sm text-gray-400">
              PDF, DOCX, DOC, TXT (max 10MB each)
            </p>
          </>
        )}
      </div>

      {/* File List */}
      {files.length > 0 && (
        <div className="space-y-2">
          <h3 className="font-medium text-gray-900">Uploaded Files</h3>
          {files.map((uploadedFile, index) => (
            <div
              key={index}
              className="flex items-center justify-between p-3 bg-white border rounded-lg"
            >
              <div className="flex items-center space-x-3 flex-1">
                <File className="w-5 h-5 text-gray-400" />
                <div className="flex-1 min-w-0">
                  <p className="text-sm font-medium text-gray-900 truncate">
                    {uploadedFile.file.name}
                  </p>
                  <p className="text-xs text-gray-500">
                    {(uploadedFile.file.size / 1024 / 1024).toFixed(2)} MB
                  </p>
                </div>
              </div>

              {/* Status */}
              <div className="flex items-center space-x-2">
                {uploadedFile.status === 'uploading' && (
                  <div className="w-24">
                    <div className="h-2 bg-gray-200 rounded-full overflow-hidden">
                      <div
                        className="h-full bg-blue-500 transition-all duration-300"
                        style={{ width: `${uploadedFile.progress}%` }}
                      />
                    </div>
                  </div>
                )}

                {uploadedFile.status === 'success' && (
                  <CheckCircle className="w-5 h-5 text-green-500" />
                )}

                {uploadedFile.status === 'error' && (
                  <AlertCircle className="w-5 h-5 text-red-500" />
                )}

                <button
                  onClick={() => removeFile(uploadedFile.file)}
                  className="p-1 hover:bg-gray-100 rounded"
                >
                  <X className="w-4 h-4 text-gray-400" />
                </button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};
```

### 2. Resume Service

**File: `frontend/src/services/resumeService.ts`**

```typescript
import { api } from './api';

export const uploadResume = async (
  file: File,
  onProgress?: (progress: number) => void
) => {
  const formData = new FormData();
  formData.append('file', file);

  return api.post('/resumes/upload', formData, {
    headers: {
      'Content-Type': 'multipart/form-data'
    },
    onUploadProgress: (progressEvent) => {
      const progress = Math.round(
        (progressEvent.loaded * 100) / (progressEvent.total || 1)
      );
      onProgress?.(progress);
    }
  });
};

export const getResume = async (resumeId: string) => {
  return api.get(`/resumes/${resumeId}`);
};

export const downloadResume = async (resumeId: string) => {
  const response = await api.get(`/resumes/${resumeId}/download`, {
    responseType: 'blob'
  });

  // Trigger download
  const url = window.URL.createObjectURL(new Blob([response.data]));
  const link = document.createElement('a');
  link.href = url;
  link.setAttribute('download', `resume_${resumeId}.pdf`);
  document.body.appendChild(link);
  link.click();
  link.remove();
};
```

---

## Testing

### Unit Tests

**File: `backend/tests/test_resume_parser.py`**

```python
import pytest
from app.services.resume_parser import ResumeParserService

@pytest.fixture
def parser():
    return ResumeParserService()

def test_extract_email(parser):
    text = "Contact me at john.doe@example.com"
    contact = parser._extract_contact(text)
    assert contact["email"] == "john.doe@example.com"

def test_extract_phone(parser):
    text = "Phone: (555) 123-4567"
    contact = parser._extract_contact(text)
    assert "555" in contact["phone"]

def test_extract_skills(parser):
    text = "Skills: Python, JavaScript, React, Docker"
    skills = parser._extract_skills(text)
    assert "Python" in skills
    assert "React" in skills

def test_calculate_years_of_experience(parser):
    experiences = [
        {"start_date": "2018", "end_date": "2020"},
        {"start_date": "2020", "end_date": "present"}
    ]
    years = parser.calculate_years_of_experience(experiences)
    assert years >= 5.0
```

---

## Performance Considerations

1. **Async Processing**: Use Celery for parsing to avoid blocking API
2. **Batch Processing**: Process multiple resumes in parallel
3. **Caching**: Cache parsed results
4. **OCR Optimization**: Only use OCR when text extraction fails
5. **File Size Limits**: Enforce 10MB limit per file

---

## Security Considerations

1. **File Validation**: Check file types and signatures (magic bytes)
2. **Virus Scanning**: Integrate ClamAV for malware detection
3. **Sandboxing**: Parse files in isolated environment
4. **Access Control**: Only team members can access resumes
5. **Encryption**: Encrypt files at rest in S3

---

## Next Steps

1. Implement duplicate detection
2. Add resume comparison feature
3. Implement versioning (multiple resumes per candidate)
4. Add confidence scores for extracted fields
5. Train custom NER model for better accuracy

This implementation provides a robust resume processing pipeline with support for multiple file formats, async processing, and structured data extraction.
