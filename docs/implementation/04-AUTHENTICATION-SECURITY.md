# Authentication & Security Implementation Guide

## Overview
Complete implementation of authentication, authorization, and security features including JWT tokens, OAuth, 2FA, role-based access control, and security best practices.

---

## Authentication System

### 1. JWT Token Management

**File: `app/core/security.py`**

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
security = HTTPBearer()

SECRET_KEY = settings.SECRET_KEY
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60  # 1 hour
REFRESH_TOKEN_EXPIRE_DAYS = 30  # 30 days

def hash_password(password: str) -> str:
    """Hash password using bcrypt"""
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(data: dict, expires_delta: timedelta = None) -> str:
    """Create JWT access token"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({
        "exp": expire,
        "iat": datetime.utcnow(),
        "type": "access"
    })

    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def create_refresh_token(data: dict) -> str:
    """Create JWT refresh token"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)

    to_encode.update({
        "exp": expire,
        "iat": datetime.utcnow(),
        "type": "refresh"
    })

    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def decode_token(token: str) -> dict:
    """Decode and validate JWT token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials"
        )

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db = Depends(get_db)
):
    """Get current authenticated user"""
    token = credentials.credentials

    # Decode token
    payload = decode_token(token)

    if payload.get("type") != "access":
        raise HTTPException(401, "Invalid token type")

    user_id = payload.get("sub")
    if not user_id:
        raise HTTPException(401, "Invalid token")

    # Check if token is revoked (check sessions table)
    session = db.query(Session).filter(
        Session.access_token_jti == payload.get("jti"),
        Session.revoked_at.is_(None)
    ).first()

    if not session:
        # Allow if jti is None (older tokens)
        if payload.get("jti"):
            raise HTTPException(401, "Token has been revoked")

    # Get user
    user = db.query(User).filter(User.id == user_id, User.is_active == True).first()

    if not user:
        raise HTTPException(401, "User not found")

    return user

async def get_current_active_user(
    current_user: User = Depends(get_current_user)
):
    """Ensure user is active"""
    if not current_user.is_active:
        raise HTTPException(400, "Inactive user")
    return current_user

def require_role(*roles: str):
    """Decorator to require specific role(s)"""
    async def role_checker(current_user: User = Depends(get_current_user)):
        if current_user.role not in roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions"
            )
        return current_user
    return role_checker
```

### 2. Authentication Endpoints

**File: `app/api/v1/auth.py`**

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from app.core.security import (
    hash_password, verify_password,
    create_access_token, create_refresh_token
)
from app.schemas.auth import RegisterRequest, LoginRequest, TokenResponse
from app.models.user import User
from app.models.team import Team
from app.models.session import Session as SessionModel
import uuid

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/register", response_model=TokenResponse)
async def register(
    data: RegisterRequest,
    db: Session = Depends(get_db)
):
    """Register new user"""

    # Check if email exists
    existing_user = db.query(User).filter(User.email == data.email).first()
    if existing_user:
        raise HTTPException(400, "Email already registered")

    # Create team if provided
    team = None
    if data.team_name:
        team_slug = data.team_name.lower().replace(" ", "-")
        team = Team(
            name=data.team_name,
            slug=team_slug,
            plan="free"
        )
        db.add(team)
        db.flush()

    # Create user
    user = User(
        email=data.email,
        hashed_password=hash_password(data.password),
        full_name=data.full_name,
        role="recruiter"
    )
    db.add(user)
    db.flush()

    # Link user to team
    if team:
        team.owner_id = user.id
        db.add(TeamMember(
            team_id=team.id,
            user_id=user.id,
            role="owner",
            joined_at=datetime.utcnow()
        ))

    db.commit()

    # Generate tokens
    access_token = create_access_token({"sub": str(user.id)})
    refresh_token = create_refresh_token({"sub": str(user.id)})

    # Store session
    session = SessionModel(
        user_id=user.id,
        refresh_token=refresh_token,
        expires_at=datetime.utcnow() + timedelta(days=30)
    )
    db.add(session)
    db.commit()

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer",
        "expires_in": 3600,
        "user": user.to_dict()
    }

@router.post("/login", response_model=TokenResponse)
async def login(
    data: LoginRequest,
    db: Session = Depends(get_db)
):
    """Login with email and password"""

    # Find user
    user = db.query(User).filter(User.email == data.email).first()

    if not user or not verify_password(data.password, user.hashed_password):
        # Increment failed attempts
        if user:
            user.failed_login_attempts += 1

            # Lock account after 5 failed attempts
            if user.failed_login_attempts >= 5:
                user.locked_until = datetime.utcnow() + timedelta(hours=1)

            db.commit()

        raise HTTPException(401, "Incorrect email or password")

    # Check if account is locked
    if user.locked_until and user.locked_until > datetime.utcnow():
        raise HTTPException(403, "Account is temporarily locked. Try again later.")

    # Check if user is active
    if not user.is_active:
        raise HTTPException(403, "Account is deactivated")

    # Reset failed attempts
    user.failed_login_attempts = 0
    user.last_login_at = datetime.utcnow()
    db.commit()

    # Generate tokens
    jti = str(uuid.uuid4())
    access_token = create_access_token({"sub": str(user.id), "jti": jti})
    refresh_token = create_refresh_token({"sub": str(user.id)})

    # Store session
    session = SessionModel(
        user_id=user.id,
        refresh_token=refresh_token,
        access_token_jti=jti,
        expires_at=datetime.utcnow() + timedelta(days=30)
    )
    db.add(session)
    db.commit()

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer",
        "expires_in": 3600,
        "user": user.to_dict()
    }

@router.post("/refresh")
async def refresh_token(
    refresh_token: str,
    db: Session = Depends(get_db)
):
    """Refresh access token"""

    # Decode refresh token
    try:
        payload = decode_token(refresh_token)

        if payload.get("type") != "refresh":
            raise HTTPException(401, "Invalid token type")

        user_id = payload.get("sub")

        # Check if session exists and is not revoked
        session = db.query(SessionModel).filter(
            SessionModel.refresh_token == refresh_token,
            SessionModel.revoked_at.is_(None)
        ).first()

        if not session:
            raise HTTPException(401, "Invalid or revoked token")

        # Check expiration
        if session.expires_at < datetime.utcnow():
            raise HTTPException(401, "Token has expired")

        # Generate new access token
        jti = str(uuid.uuid4())
        access_token = create_access_token({"sub": user_id, "jti": jti})

        # Update session
        session.access_token_jti = jti
        db.commit()

        return {
            "access_token": access_token,
            "token_type": "bearer",
            "expires_in": 3600
        }

    except JWTError:
        raise HTTPException(401, "Invalid token")

@router.post("/logout")
async def logout(
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Logout and revoke current session"""

    # Revoke all sessions for this user
    db.query(SessionModel).filter(
        SessionModel.user_id == current_user.id,
        SessionModel.revoked_at.is_(None)
    ).update({"revoked_at": datetime.utcnow()})

    db.commit()

    return {"success": True, "message": "Logged out successfully"}
```

### 3. Two-Factor Authentication (2FA)

**File: `app/api/v1/auth.py` (continued)**

```python
import pyotp
import qrcode
import io
import base64

@router.post("/enable-2fa")
async def enable_2fa(
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Enable 2FA for user"""

    if current_user.is_2fa_enabled:
        raise HTTPException(400, "2FA is already enabled")

    # Generate secret
    secret = pyotp.random_base32()

    # Generate QR code
    totp = pyotp.TOTP(secret)
    provisioning_uri = totp.provisioning_uri(
        name=current_user.email,
        issuer_name="HireSight"
    )

    # Create QR code image
    qr = qrcode.QRCode(version=1, box_size=10, border=5)
    qr.add_data(provisioning_uri)
    qr.make(fit=True)

    img = qr.make_image(fill_color="black", back_color="white")

    # Convert to base64
    buffer = io.BytesIO()
    img.save(buffer, format="PNG")
    qr_code_base64 = base64.b64encode(buffer.getvalue()).decode()

    # Store secret temporarily (not enabled yet)
    current_user.totp_secret = secret
    db.commit()

    return {
        "success": True,
        "data": {
            "secret": secret,
            "qr_code_url": f"data:image/png;base64,{qr_code_base64}"
        }
    }

@router.post("/verify-2fa")
async def verify_2fa(
    token: str,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Verify 2FA token and enable"""

    if not current_user.totp_secret:
        raise HTTPException(400, "2FA not initiated")

    totp = pyotp.TOTP(current_user.totp_secret)

    if not totp.verify(token):
        raise HTTPException(400, "Invalid token")

    # Enable 2FA
    current_user.is_2fa_enabled = True
    db.commit()

    return {
        "success": True,
        "message": "2FA enabled successfully"
    }

@router.post("/disable-2fa")
async def disable_2fa(
    password: str,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Disable 2FA (requires password)"""

    if not verify_password(password, current_user.hashed_password):
        raise HTTPException(401, "Incorrect password")

    current_user.is_2fa_enabled = False
    current_user.totp_secret = None
    db.commit()

    return {
        "success": True,
        "message": "2FA disabled successfully"
    }
```

### 4. OAuth Integration (Google & GitHub)

**File: `app/services/oauth_service.py`**

```python
import httpx
from app.core.config import settings

class OAuthService:
    @staticmethod
    async def google_login(id_token: str, db: Session):
        """Authenticate with Google ID token"""

        # Verify Google ID token
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"https://oauth2.googleapis.com/tokeninfo?id_token={id_token}"
            )

            if response.status_code != 200:
                raise HTTPException(401, "Invalid Google token")

            data = response.json()

        email = data.get("email")
        name = data.get("name")
        google_id = data.get("sub")

        # Find or create user
        user = db.query(User).filter(User.email == email).first()

        if not user:
            # Create new user
            user = User(
                email=email,
                full_name=name,
                google_id=google_id,
                is_verified=True,
                email_verified_at=datetime.utcnow()
            )
            db.add(user)
            db.commit()
        else:
            # Link Google account
            user.google_id = google_id
            db.commit()

        return user

    @staticmethod
    async def github_login(code: str, db: Session):
        """Authenticate with GitHub OAuth code"""

        # Exchange code for access token
        async with httpx.AsyncClient() as client:
            token_response = await client.post(
                "https://github.com/login/oauth/access_token",
                json={
                    "client_id": settings.GITHUB_CLIENT_ID,
                    "client_secret": settings.GITHUB_CLIENT_SECRET,
                    "code": code
                },
                headers={"Accept": "application/json"}
            )

            token_data = token_response.json()
            access_token = token_data.get("access_token")

            if not access_token:
                raise HTTPException(401, "Failed to get GitHub access token")

            # Get user info
            user_response = await client.get(
                "https://api.github.com/user",
                headers={"Authorization": f"token {access_token}"}
            )

            github_user = user_response.json()

        email = github_user.get("email")
        name = github_user.get("name")
        github_id = str(github_user.get("id"))

        # Find or create user
        user = db.query(User).filter(User.email == email).first()

        if not user:
            user = User(
                email=email,
                full_name=name,
                github_id=github_id,
                is_verified=True
            )
            db.add(user)
            db.commit()
        else:
            user.github_id = github_id
            db.commit()

        return user
```

---

## Security Features

### 1. Rate Limiting

**File: `app/middleware/rate_limit.py`**

```python
from fastapi import Request, HTTPException
from redis import Redis
import time

redis_client = Redis(host=settings.REDIS_HOST, port=settings.REDIS_PORT, decode_responses=True)

async def rate_limit_middleware(request: Request, call_next):
    """Rate limit requests"""

    # Get client IP
    client_ip = request.client.host

    # Rate limit key
    key = f"rate_limit:{client_ip}"

    # Get current count
    count = redis_client.get(key)

    if count is None:
        # First request
        redis_client.setex(key, 3600, 1)  # 1 hour window
    else:
        count = int(count)

        if count >= 1000:  # 1000 requests per hour
            raise HTTPException(429, "Too many requests. Please try again later.")

        redis_client.incr(key)

    response = await call_next(request)
    return response
```

### 2. Input Validation & Sanitization

**File: `app/middleware/security.py`**

```python
from fastapi import Request
import bleach

async def sanitize_input_middleware(request: Request, call_next):
    """Sanitize user input to prevent XSS"""

    if request.method in ["POST", "PUT", "PATCH"]:
        # Get request body
        body = await request.json()

        # Sanitize string fields
        sanitized = sanitize_dict(body)

        # Replace request body
        request._body = json.dumps(sanitized).encode()

    response = await call_next(request)
    return response

def sanitize_dict(data: dict) -> dict:
    """Recursively sanitize dictionary"""
    sanitized = {}

    for key, value in data.items():
        if isinstance(value, str):
            # Remove HTML tags and dangerous characters
            sanitized[key] = bleach.clean(value, strip=True)
        elif isinstance(value, dict):
            sanitized[key] = sanitize_dict(value)
        elif isinstance(value, list):
            sanitized[key] = [
                sanitize_dict(item) if isinstance(item, dict) else item
                for item in value
            ]
        else:
            sanitized[key] = value

    return sanitized
```

### 3. CORS Configuration

**File: `app/main.py`**

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:5173",  # Frontend dev
        "https://app.hiresight.com"  # Production
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-RateLimit-Limit", "X-RateLimit-Remaining"]
)
```

### 4. API Key Authentication

**File: `app/core/security.py` (continued)**

```python
async def get_api_key(
    api_key: str = Depends(HTTPBearer()),
    db: Session = Depends(get_db)
):
    """Authenticate using API key"""

    # Extract key prefix
    key_prefix = api_key[:8]

    # Find API key
    api_key_record = db.query(APIKey).filter(
        APIKey.key_prefix == key_prefix,
        APIKey.revoked_at.is_(None)
    ).first()

    if not api_key_record:
        raise HTTPException(401, "Invalid API key")

    # Verify full key (hashed)
    if not verify_password(api_key, api_key_record.hashed_key):
        raise HTTPException(401, "Invalid API key")

    # Check expiration
    if api_key_record.expires_at and api_key_record.expires_at < datetime.utcnow():
        raise HTTPException(401, "API key has expired")

    # Update last used
    api_key_record.last_used_at = datetime.utcnow()
    api_key_record.usage_count += 1
    db.commit()

    # Check rate limit
    key = f"api_rate_limit:{api_key_record.id}"
    count = redis_client.get(key)

    limit = 10000  # 10k requests per hour
    if count and int(count) >= limit:
        raise HTTPException(429, "API rate limit exceeded")

    redis_client.incr(key)
    redis_client.expire(key, 3600)

    return api_key_record
```

---

## Frontend Implementation

### Auth Context

**File: `frontend/src/contexts/AuthContext.tsx`**

```typescript
import React, { createContext, useContext, useState, useEffect } from 'react';
import { login as apiLogin, logout as apiLogout, refreshToken } from '../services/authService';

interface User {
  id: string;
  email: string;
  full_name: string;
  role: string;
}

interface AuthContextType {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  isAuthenticated: boolean;
  isLoading: boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Check if user is logged in
    const token = localStorage.getItem('access_token');
    if (token) {
      // Verify token and get user
      fetchUser();
    } else {
      setIsLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await fetch('/api/v1/users/me', {
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('access_token')}`
        }
      });

      if (response.ok) {
        const data = await response.json();
        setUser(data.data.user);
      } else {
        // Token invalid, try refresh
        await refreshToken();
        await fetchUser();
      }
    } catch (error) {
      localStorage.removeItem('access_token');
      localStorage.removeItem('refresh_token');
    } finally {
      setIsLoading(false);
    }
  };

  const login = async (email: string, password: string) => {
    const data = await apiLogin(email, password);

    localStorage.setItem('access_token', data.access_token);
    localStorage.setItem('refresh_token', data.refresh_token);

    setUser(data.user);
  };

  const logout = async () => {
    await apiLogout();

    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');

    setUser(null);
  };

  return (
    <AuthContext.Provider value={{
      user,
      login,
      logout,
      isAuthenticated: !!user,
      isLoading
    }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

---

## Security Best Practices

1. **Password Requirements**: Min 8 chars, uppercase, lowercase, number, special char
2. **Token Storage**: HttpOnly cookies (server-side) or localStorage with XSS protection
3. **HTTPS Only**: Enforce HTTPS in production
4. **SQL Injection**: Use parameterized queries (SQLAlchemy ORM)
5. **CSRF Protection**: SameSite cookies + CSRF tokens
6. **File Upload**: Validate file types, scan for malware
7. **Logging**: Log all authentication events
8. **Secrets**: Use environment variables, never commit secrets
9. **Dependencies**: Regularly update and scan for vulnerabilities
10. **Audit Trail**: Log all sensitive operations

This implementation provides enterprise-grade authentication and security for the HIRESIGHT platform.
