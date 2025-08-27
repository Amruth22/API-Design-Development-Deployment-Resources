# Testing & Security Frameworks - Testing Strategies + Security Architecture + Authentication + Authorization

## Professional PowerPoint Presentation

---

## Slide 1: Title Slide

### Testing & Security Frameworks
#### Testing Strategies, Security Architecture, Authentication, Authorization, JWT & OAuth2

**Comprehensive API Security and Testing Implementation**

*Professional Training Series*

---

## Slide 2: Testing Strategy Overview

### Comprehensive Testing Pyramid

**Testing Levels:**
```
        ┌─────────────────┐
        │   E2E Tests     │  ← Few, expensive, slow
        │   (5-10%)       │
        ├─────────────────┤
        │ Integration     │  ← Moderate, medium cost
        │ Tests (20-30%)  │
        ├─────────────────┤
        │   Unit Tests    │  ← Many, cheap, fast
        │   (60-70%)      │
        └─────────────────┘
```

**Testing Types:**
- **Unit Tests:** Individual functions/methods in isolation
- **Integration Tests:** Component interactions and API endpoints
- **End-to-End Tests:** Complete user workflows
- **Performance Tests:** Load, stress, and scalability testing
- **Security Tests:** Vulnerability and penetration testing

**Testing Principles:**
- Fast feedback loops
- Reliable and repeatable
- Independent and isolated
- Clear and maintainable
- Comprehensive coverage

---

## Slide 3: Unit Testing with pytest

### Testing Individual Components

**Basic Unit Test Structure:**
```python
import pytest
from unittest.mock import Mock, AsyncMock
from myapp.services import UserService
from myapp.models import User

class TestUserService:
    @pytest.fixture
    def mock_user_repository(self):
        return AsyncMock()
    
    @pytest.fixture
    def mock_email_service(self):
        return AsyncMock()
    
    @pytest.fixture
    def user_service(self, mock_user_repository, mock_email_service):
        return UserService(mock_user_repository, mock_email_service)
    
    async def test_create_user_success(self, user_service, mock_user_repository):
        # Arrange
        user_data = {"name": "John Doe", "email": "john@example.com"}
        expected_user = User(id=1, **user_data)
        mock_user_repository.create.return_value = expected_user
        
        # Act
        result = await user_service.create_user(user_data)
        
        # Assert
        assert result.name == "John Doe"
        assert result.email == "john@example.com"
        mock_user_repository.create.assert_called_once()
```

**Testing Async Functions:**
```python
@pytest.mark.asyncio
async def test_async_database_operation():
    async with AsyncSession() as session:
        user = User(name="Test User", email="test@example.com")
        session.add(user)
        await session.commit()
        
        result = await session.get(User, user.id)
        assert result.name == "Test User"
```

**Parametrized Testing:**
```python
@pytest.mark.parametrize("email,expected", [
    ("valid@example.com", True),
    ("invalid-email", False),
    ("", False),
    ("@example.com", False),
])
def test_email_validation(email, expected):
    result = validate_email(email)
    assert result == expected
```

---

## Slide 4: Integration Testing

### Testing API Endpoints and Component Interactions

**FastAPI Test Client:**
```python
from fastapi.testclient import TestClient
from myapp.main import app

client = TestClient(app)

def test_create_user_endpoint():
    response = client.post(
        "/users",
        json={"name": "John Doe", "email": "john@example.com"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "John Doe"
    assert "id" in data

def test_get_user_endpoint():
    # Create user first
    create_response = client.post(
        "/users",
        json={"name": "Jane Doe", "email": "jane@example.com"}
    )
    user_id = create_response.json()["id"]
    
    # Test get endpoint
    response = client.get(f"/users/{user_id}")
    assert response.status_code == 200
    assert response.json()["name"] == "Jane Doe"

def test_user_not_found():
    response = client.get("/users/999")
    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()
```

**Database Integration Testing:**
```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from myapp.database import Base, get_db
from myapp.main import app

# Test database setup
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture
def test_db():
    Base.metadata.create_all(bind=engine)
    yield
    Base.metadata.drop_all(bind=engine)

def override_get_db():
    try:
        db = TestingSessionLocal()
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = override_get_db
```

---

## Slide 5: Security Architecture Fundamentals

### Defense in Depth Strategy

**Security Layers:**
```
┌─────────────────────────────────────┐
│        Application Layer            │  ← Input validation, business logic
├─────────────────────────────────────┤
│        Authentication Layer         │  ← Identity verification
├─────────────────────────────────────┤
│        Authorization Layer          │  ← Permission checking
├─────────────────────────────────────┤
│        Transport Layer              │  ← HTTPS, TLS encryption
├─────────────────────────────────────┤
│        Network Layer                │  ← Firewalls, VPN
└─────────────────────────────────────┘
```

**Security Principles:**
- **Least Privilege:** Grant minimum necessary permissions
- **Defense in Depth:** Multiple security layers
- **Fail Secure:** Default to secure state on failure
- **Zero Trust:** Verify everything, trust nothing
- **Security by Design:** Build security from the start

**Common Vulnerabilities (OWASP Top 10):**
- Injection attacks (SQL, NoSQL, LDAP)
- Broken authentication and session management
- Cross-Site Scripting (XSS)
- Insecure direct object references
- Security misconfiguration
- Sensitive data exposure
- Missing function-level access control

---

## Slide 6: Authentication Implementation

### Identity Verification Mechanisms

**Password-Based Authentication:**
```python
from passlib.context import CryptContext
from datetime import datetime, timedelta
import jwt

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class AuthService:
    def __init__(self, secret_key: str):
        self.secret_key = secret_key
    
    def hash_password(self, password: str) -> str:
        return pwd_context.hash(password)
    
    def verify_password(self, plain_password: str, hashed_password: str) -> bool:
        return pwd_context.verify(plain_password, hashed_password)
    
    def create_access_token(self, data: dict, expires_delta: timedelta = None):
        to_encode = data.copy()
        if expires_delta:
            expire = datetime.utcnow() + expires_delta
        else:
            expire = datetime.utcnow() + timedelta(minutes=15)
        
        to_encode.update({"exp": expire})
        encoded_jwt = jwt.encode(to_encode, self.secret_key, algorithm="HS256")
        return encoded_jwt
    
    def verify_token(self, token: str):
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=["HS256"])
            return payload
        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail="Token expired")
        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail="Invalid token")
```

**Login Endpoint:**
```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = await authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    access_token_expires = timedelta(minutes=30)
    access_token = create_access_token(
        data={"sub": user.email}, expires_delta=access_token_expires
    )
    return {"access_token": access_token, "token_type": "bearer"}
```

---

## Slide 7: JSON Web Tokens (JWT)

### Stateless Authentication Tokens

**JWT Structure:**
```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**JWT Components:**
- **Header:** Algorithm and token type
- **Payload:** Claims (user data, expiration)
- **Signature:** Verification hash

**JWT Implementation:**
```python
import jwt
from datetime import datetime, timedelta
from typing import Optional

class JWTManager:
    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        self.secret_key = secret_key
        self.algorithm = algorithm
    
    def create_token(self, user_id: int, email: str, expires_in: int = 3600) -> str:
        payload = {
            "user_id": user_id,
            "email": email,
            "exp": datetime.utcnow() + timedelta(seconds=expires_in),
            "iat": datetime.utcnow(),
            "type": "access"
        }
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def create_refresh_token(self, user_id: int) -> str:
        payload = {
            "user_id": user_id,
            "exp": datetime.utcnow() + timedelta(days=30),
            "iat": datetime.utcnow(),
            "type": "refresh"
        }
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def verify_token(self, token: str) -> dict:
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return payload
        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail="Token has expired")
        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail="Invalid token")
```

**JWT Best Practices:**
- Use strong secret keys
- Set appropriate expiration times
- Implement token refresh mechanism
- Store sensitive data server-side, not in JWT
- Use HTTPS for token transmission

---

## Slide 8: Authorization and Role-Based Access Control

### Permission Management System

**RBAC Model:**
```python
from enum import Enum
from typing import List

class Permission(Enum):
    READ_USER = "read:user"
    WRITE_USER = "write:user"
    DELETE_USER = "delete:user"
    READ_ADMIN = "read:admin"
    WRITE_ADMIN = "write:admin"

class Role(BaseModel):
    name: str
    permissions: List[Permission]

# Predefined roles
ROLES = {
    "user": Role(name="user", permissions=[Permission.READ_USER]),
    "moderator": Role(name="moderator", permissions=[
        Permission.READ_USER, Permission.WRITE_USER
    ]),
    "admin": Role(name="admin", permissions=[
        Permission.READ_USER, Permission.WRITE_USER, Permission.DELETE_USER,
        Permission.READ_ADMIN, Permission.WRITE_ADMIN
    ])
}

class User(BaseModel):
    id: int
    email: str
    roles: List[str]
    
    def has_permission(self, permission: Permission) -> bool:
        for role_name in self.roles:
            role = ROLES.get(role_name)
            if role and permission in role.permissions:
                return True
        return False
```

**Authorization Decorators:**
```python
from functools import wraps
from fastapi import Depends, HTTPException, status

def require_permission(permission: Permission):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
            if not current_user.has_permission(permission):
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="Insufficient permissions"
                )
            return await func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator

@app.delete("/users/{user_id}")
@require_permission(Permission.DELETE_USER)
async def delete_user(user_id: int, current_user: User = Depends(get_current_user)):
    return await user_service.delete_user(user_id)
```

---

## Slide 9: OAuth2 Implementation

### Third-Party Authentication Integration

**OAuth2 Flow:**
```
1. Client → Authorization Server: Request authorization
2. User → Authorization Server: Login and consent
3. Authorization Server → Client: Authorization code
4. Client → Authorization Server: Exchange code for token
5. Client → Resource Server: Access resources with token
```

**OAuth2 with FastAPI:**
```python
from authlib.integrations.starlette_client import OAuth
from starlette.config import Config

config = Config('.env')
oauth = OAuth(config)

# Google OAuth2 configuration
oauth.register(
    name='google',
    client_id=config('GOOGLE_CLIENT_ID'),
    client_secret=config('GOOGLE_CLIENT_SECRET'),
    server_metadata_url='https://accounts.google.com/.well-known/openid_configuration',
    client_kwargs={
        'scope': 'openid email profile'
    }
)

@app.get('/auth/google')
async def google_auth(request: Request):
    redirect_uri = request.url_for('google_callback')
    return await oauth.google.authorize_redirect(request, redirect_uri)

@app.get('/auth/google/callback')
async def google_callback(request: Request):
    token = await oauth.google.authorize_access_token(request)
    user_info = token.get('userinfo')
    
    # Create or update user in database
    user = await get_or_create_user(user_info)
    
    # Create JWT token for our application
    access_token = create_access_token({"sub": user.email})
    
    return {"access_token": access_token, "token_type": "bearer"}
```

**OAuth2 Scopes:**
```python
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={
        "read": "Read access",
        "write": "Write access",
        "admin": "Admin access"
    }
)

@app.get("/users/me")
async def read_users_me(
    token: str = Depends(oauth2_scheme),
    required_scopes: SecurityScopes = SecurityScopes(["read"])
):
    # Verify token and scopes
    payload = verify_token(token)
    token_scopes = payload.get("scopes", [])
    
    for scope in required_scopes.scopes:
        if scope not in token_scopes:
            raise HTTPException(
                status_code=401,
                detail="Not enough permissions"
            )
    
    return get_current_user(payload)
```

---

## Slide 10: Security Testing

### Vulnerability Assessment and Penetration Testing

**Security Test Categories:**
```python
class SecurityTests:
    
    def test_sql_injection_protection(self):
        """Test SQL injection vulnerability"""
        malicious_input = "'; DROP TABLE users; --"
        response = client.get(f"/users?name={malicious_input}")
        # Should not cause database error
        assert response.status_code != 500
    
    def test_xss_protection(self):
        """Test Cross-Site Scripting protection"""
        xss_payload = "<script>alert('XSS')</script>"
        response = client.post("/users", json={"name": xss_payload})
        # Should sanitize or reject malicious input
        assert "<script>" not in response.json().get("name", "")
    
    def test_authentication_required(self):
        """Test that protected endpoints require authentication"""
        response = client.get("/admin/users")
        assert response.status_code == 401
    
    def test_authorization_enforcement(self):
        """Test that users can only access authorized resources"""
        user_token = create_test_token(user_id=1, role="user")
        headers = {"Authorization": f"Bearer {user_token}"}
        
        response = client.delete("/admin/users/2", headers=headers)
        assert response.status_code == 403
    
    def test_rate_limiting(self):
        """Test rate limiting protection"""
        for _ in range(100):
            response = client.post("/login", json={"username": "test", "password": "wrong"})
        
        # Should be rate limited after many attempts
        assert response.status_code == 429
```

**Automated Security Scanning:**
```python
import bandit  # Security linter for Python
import safety  # Check for known security vulnerabilities

# Example security checks in CI/CD pipeline
def run_security_checks():
    # Static code analysis
    bandit_result = subprocess.run(["bandit", "-r", "myapp/"], capture_output=True)
    
    # Dependency vulnerability check
    safety_result = subprocess.run(["safety", "check"], capture_output=True)
    
    # Custom security tests
    pytest_result = subprocess.run(["pytest", "tests/security/"], capture_output=True)
    
    return all([
        bandit_result.returncode == 0,
        safety_result.returncode == 0,
        pytest_result.returncode == 0
    ])
```

---

## Slide 11: Performance and Load Testing

### Testing System Scalability and Performance

**Load Testing with Locust:**
```python
from locust import HttpUser, task, between

class APIUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        """Login and get authentication token"""
        response = self.client.post("/token", data={
            "username": "testuser",
            "password": "testpass"
        })
        self.token = response.json()["access_token"]
        self.headers = {"Authorization": f"Bearer {self.token}"}
    
    @task(3)
    def get_users(self):
        """Test GET /users endpoint"""
        self.client.get("/users", headers=self.headers)
    
    @task(1)
    def create_user(self):
        """Test POST /users endpoint"""
        self.client.post("/users", 
            json={"name": "Test User", "email": "test@example.com"},
            headers=self.headers
        )
    
    @task(2)
    def get_user_profile(self):
        """Test GET /users/{id} endpoint"""
        user_id = random.randint(1, 1000)
        self.client.get(f"/users/{user_id}", headers=self.headers)
```

**Performance Benchmarking:**
```python
import time
import asyncio
from statistics import mean, median

async def benchmark_endpoint(url: str, concurrent_requests: int = 100):
    """Benchmark API endpoint performance"""
    async with aiohttp.ClientSession() as session:
        start_time = time.time()
        
        tasks = []
        for _ in range(concurrent_requests):
            task = session.get(url)
            tasks.append(task)
        
        responses = await asyncio.gather(*tasks)
        end_time = time.time()
        
        # Calculate metrics
        total_time = end_time - start_time
        requests_per_second = concurrent_requests / total_time
        
        response_times = [r.elapsed.total_seconds() for r in responses]
        avg_response_time = mean(response_times)
        median_response_time = median(response_times)
        
        return {
            "requests_per_second": requests_per_second,
            "avg_response_time": avg_response_time,
            "median_response_time": median_response_time,
            "total_requests": concurrent_requests,
            "total_time": total_time
        }
```

---

## Slide 12: Summary and Security Checklist

### Comprehensive Security and Testing Strategy

**Testing Best Practices:**
- ✓ Implement comprehensive test coverage (unit, integration, e2e)
- ✓ Use test-driven development (TDD) approach
- ✓ Automate testing in CI/CD pipeline
- ✓ Perform regular security testing
- ✓ Conduct performance and load testing
- ✓ Mock external dependencies in tests
- ✓ Maintain test data isolation
- ✓ Regular test maintenance and updates

**Security Implementation Checklist:**
- ✓ Strong authentication mechanisms
- ✓ Role-based authorization system
- ✓ JWT token management with refresh tokens
- ✓ OAuth2 integration for third-party auth
- ✓ Input validation and sanitization
- ✓ SQL injection protection
- ✓ XSS prevention measures
- ✓ HTTPS enforcement
- ✓ Rate limiting implementation
- ✓ Security headers configuration
- ✓ Regular security audits
- ✓ Dependency vulnerability scanning

**Production Security Measures:**
- Environment variable management
- Secrets management system
- Database encryption at rest
- API rate limiting and throttling
- Comprehensive logging and monitoring
- Incident response procedures
- Regular security updates
- Backup and disaster recovery plans

**Continuous Improvement:**
- Regular security training for development team
- Stay updated with OWASP guidelines
- Implement security code reviews
- Automated security testing in CI/CD
- Regular penetration testing
- Security metrics and KPI tracking

---

## Presentation Notes

**Target Audience:** Developers, security engineers, QA professionals
**Duration:** 70-85 minutes
**Prerequisites:** API development experience, basic security concepts
**Learning Objectives:**
- Implement comprehensive testing strategies
- Design secure authentication and authorization systems
- Master JWT and OAuth2 implementation
- Conduct effective security testing and audits