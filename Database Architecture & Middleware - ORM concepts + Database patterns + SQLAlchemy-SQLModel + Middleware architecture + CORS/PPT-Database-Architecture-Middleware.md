# Database Architecture & Middleware - ORM + SQLAlchemy/SQLModel + Middleware + CORS

## Professional PowerPoint Presentation

---

## Slide 1: Title Slide

### Database Architecture & Middleware
#### ORM Concepts, SQLAlchemy/SQLModel, Middleware Architecture & CORS

**Enterprise Database Integration and Middleware Design**

*Professional Training Series*

---

## Slide 2: Object-Relational Mapping (ORM) Fundamentals

### Bridging Object-Oriented and Relational Worlds

**What is ORM?**
- Technique for converting data between incompatible systems
- Maps database tables to programming language objects
- Provides abstraction layer over raw SQL
- Enables object-oriented database operations

**Benefits of ORM:**
- **Database Independence:** Switch databases with minimal code changes
- **Productivity:** Faster development with less boilerplate code
- **Security:** Built-in protection against SQL injection
- **Maintainability:** Cleaner, more readable code
- **Type Safety:** Compile-time error checking

**ORM vs Raw SQL:**
```python
# Raw SQL
cursor.execute("SELECT * FROM users WHERE age > %s", (25,))
users = cursor.fetchall()

# ORM
users = session.query(User).filter(User.age > 25).all()
```

**Trade-offs:**
- Performance overhead vs. development speed
- Learning curve vs. long-term maintainability
- Flexibility vs. abstraction

---

## Slide 3: SQLAlchemy Architecture

### The Python SQL Toolkit

**SQLAlchemy Components:**
- **Core:** Expression language and schema definition
- **ORM:** Object-relational mapping layer
- **Engine:** Database connection and execution
- **Session:** Unit of work pattern implementation

**Architecture Layers:**
```
┌─────────────────────┐
│       ORM           │  ← High-level object mapping
├─────────────────────┤
│       Core          │  ← SQL expression language
├─────────────────────┤
│      Engine         │  ← Connection pooling
├─────────────────────┤
│       DBAPI         │  ← Database driver
└─────────────────────┘
```

**Basic SQLAlchemy Setup:**
```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# Database connection
engine = create_engine('postgresql://user:pass@localhost/db')
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Model definition
class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    email = Column(String, unique=True, index=True)
```

---

## Slide 4: SQLModel - Modern Type-Safe ORM

### FastAPI-Optimized Database Integration

**SQLModel Features:**
- Combines SQLAlchemy and Pydantic
- Type hints for database models
- Automatic API serialization
- FastAPI integration
- Editor support and validation

**SQLModel vs SQLAlchemy:**
```python
# Traditional SQLAlchemy
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)

# SQLModel approach
class User(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    email: str = Field(unique=True, index=True)
```

**Pydantic Integration:**
```python
from sqlmodel import SQLModel, Field
from typing import Optional

class UserBase(SQLModel):
    name: str
    email: str

class User(UserBase, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)

class UserCreate(UserBase):
    pass

class UserRead(UserBase):
    id: int

class UserUpdate(SQLModel):
    name: Optional[str] = None
    email: Optional[str] = None
```

**Benefits:**
- Single model definition for database and API
- Automatic validation and serialization
- Type safety throughout the application
- Reduced code duplication

---

## Slide 5: Database Patterns and Best Practices

### Repository and Unit of Work Patterns

**Repository Pattern Implementation:**
```python
from abc import ABC, abstractmethod
from typing import List, Optional

class UserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> Optional[User]:
        pass
    
    @abstractmethod
    async def get_by_email(self, email: str) -> Optional[User]:
        pass
    
    @abstractmethod
    async def create(self, user: User) -> User:
        pass
    
    @abstractmethod
    async def update(self, user: User) -> User:
        pass

class SQLUserRepository(UserRepository):
    def __init__(self, session: AsyncSession):
        self.session = session
    
    async def get_by_id(self, user_id: int) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()
    
    async def create(self, user: User) -> User:
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user
```

**Unit of Work Pattern:**
```python
class UnitOfWork:
    def __init__(self, session_factory):
        self.session_factory = session_factory
    
    async def __aenter__(self):
        self.session = self.session_factory()
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            await self.session.rollback()
        await self.session.close()
    
    async def commit(self):
        await self.session.commit()
    
    async def rollback(self):
        await self.session.rollback()
```

---

## Slide 6: Database Connection Management

### Connection Pooling and Session Handling

**Async Database Setup:**
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Async engine with connection pooling
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=20,
    max_overflow=0,
    pool_pre_ping=True,
    pool_recycle=300
)

# Async session factory
AsyncSessionLocal = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

# Dependency for FastAPI
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

**Connection Pool Configuration:**
```python
engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,          # Number of connections to maintain
    max_overflow=20,       # Additional connections when pool is full
    pool_timeout=30,       # Seconds to wait for connection
    pool_recycle=3600,     # Seconds before recreating connection
    pool_pre_ping=True,    # Validate connections before use
    echo=False             # Log all SQL statements
)
```

**Session Management Best Practices:**
- Use dependency injection for session management
- Always close sessions properly
- Handle transactions explicitly
- Use connection pooling for performance
- Monitor connection usage and leaks

---

## Slide 7: Middleware Architecture Concepts

### Request/Response Processing Pipeline

**What is Middleware?**
- Software layer between application and infrastructure
- Processes requests before they reach route handlers
- Modifies responses before returning to client
- Implements cross-cutting concerns

**Middleware Pipeline:**
```
Request → Middleware 1 → Middleware 2 → Route Handler
                                            ↓
Response ← Middleware 1 ← Middleware 2 ← Route Handler
```

**FastAPI Middleware Implementation:**
```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
import time

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        
        # Process request
        response = await call_next(request)
        
        # Process response
        process_time = time.time() - start_time
        response.headers["X-Process-Time"] = str(process_time)
        
        return response

app = FastAPI()
app.add_middleware(TimingMiddleware)
```

**Common Middleware Use Cases:**
- Authentication and authorization
- Request/response logging
- CORS handling
- Rate limiting
- Error handling
- Performance monitoring

---

## Slide 8: CORS (Cross-Origin Resource Sharing)

### Enabling Cross-Domain API Access

**CORS Problem:**
- Browser security policy blocks cross-origin requests
- APIs need to explicitly allow cross-domain access
- Different origins: protocol, domain, or port differences

**CORS Headers:**
```http
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

**FastAPI CORS Implementation:**
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://frontend.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)

# Development configuration (less restrictive)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allow all origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**CORS Security Considerations:**
- Never use wildcard (*) with credentials in production
- Specify exact origins when possible
- Limit allowed methods and headers
- Consider preflight request handling

---

## Slide 9: Custom Middleware Development

### Building Specialized Middleware Components

**Authentication Middleware:**
```python
from fastapi import HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

class AuthenticationMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, secret_key: str):
        super().__init__(app)
        self.secret_key = secret_key
        self.security = HTTPBearer()
    
    async def dispatch(self, request: Request, call_next):
        # Skip authentication for public endpoints
        if request.url.path in ["/docs", "/openapi.json", "/health"]:
            return await call_next(request)
        
        # Extract and validate token
        authorization = request.headers.get("Authorization")
        if not authorization:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Authorization header required"
            )
        
        try:
            token = authorization.split(" ")[1]
            payload = jwt.decode(token, self.secret_key, algorithms=["HS256"])
            request.state.user_id = payload.get("user_id")
        except jwt.InvalidTokenError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token"
            )
        
        return await call_next(request)
```

**Logging Middleware:**
```python
import logging
import uuid

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Generate request ID
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id
        
        # Log request
        logger.info(
            f"Request {request_id}: {request.method} {request.url}",
            extra={
                "request_id": request_id,
                "method": request.method,
                "url": str(request.url),
                "user_agent": request.headers.get("user-agent")
            }
        )
        
        # Process request
        start_time = time.time()
        response = await call_next(request)
        process_time = time.time() - start_time
        
        # Log response
        logger.info(
            f"Response {request_id}: {response.status_code} in {process_time:.3f}s",
            extra={
                "request_id": request_id,
                "status_code": response.status_code,
                "process_time": process_time
            }
        )
        
        return response
```

---

## Slide 10: Database Migration Strategies

### Schema Evolution and Version Control

**Alembic Migration Setup:**
```python
# alembic/env.py
from alembic import context
from sqlalchemy import engine_from_config, pool
from myapp.models import Base

target_metadata = Base.metadata

def run_migrations_online():
    connectable = engine_from_config(
        context.config.get_section(context.config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata
        )
        
        with context.begin_transaction():
            context.run_migrations()
```

**Migration Best Practices:**
```bash
# Create new migration
alembic revision --autogenerate -m "Add user table"

# Review generated migration
# Edit migration file if needed

# Apply migration
alembic upgrade head

# Rollback migration
alembic downgrade -1
```

**Migration Guidelines:**
- Always review auto-generated migrations
- Test migrations on copy of production data
- Plan for zero-downtime deployments
- Keep migrations small and focused
- Document breaking changes
- Backup database before major migrations

---

## Slide 11: Performance Optimization

### Database and Middleware Performance

**Query Optimization:**
```python
# Inefficient N+1 query problem
users = session.query(User).all()
for user in users:
    print(user.orders)  # Separate query for each user

# Efficient eager loading
users = session.query(User).options(
    joinedload(User.orders)
).all()
for user in users:
    print(user.orders)  # Single query with join
```

**Caching Strategies:**
```python
from functools import lru_cache
import redis

# In-memory caching
@lru_cache(maxsize=128)
def get_user_permissions(user_id: int):
    return session.query(Permission).join(User).filter(
        User.id == user_id
    ).all()

# Redis caching
redis_client = redis.Redis(host='localhost', port=6379, db=0)

async def get_cached_user(user_id: int):
    cache_key = f"user:{user_id}"
    cached_user = redis_client.get(cache_key)
    
    if cached_user:
        return json.loads(cached_user)
    
    user = await session.get(User, user_id)
    if user:
        redis_client.setex(
            cache_key, 
            300,  # 5 minutes
            json.dumps(user.dict())
        )
    
    return user
```

**Connection Pool Monitoring:**
```python
# Monitor connection pool status
def get_pool_status():
    pool = engine.pool
    return {
        "size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "invalid": pool.invalid()
    }
```

---

## Slide 12: Summary and Best Practices

### Database Architecture Excellence

**ORM Best Practices:**
- Use appropriate loading strategies (lazy vs eager)
- Implement proper session management
- Handle transactions explicitly
- Monitor query performance
- Use database indexes effectively
- Implement connection pooling

**Middleware Design Principles:**
- Keep middleware focused and lightweight
- Handle errors gracefully
- Maintain request/response immutability when possible
- Use dependency injection for configuration
- Implement proper logging and monitoring
- Consider middleware order carefully

**Security Considerations:**
- Validate all input data
- Use parameterized queries to prevent SQL injection
- Implement proper authentication and authorization
- Configure CORS restrictively for production
- Log security events
- Regular security audits

**Performance Optimization:**
- Profile database queries regularly
- Implement appropriate caching strategies
- Use connection pooling effectively
- Monitor middleware performance impact
- Optimize database schema and indexes
- Plan for horizontal scaling

**Deployment Checklist:**
- ✓ Database migrations tested and documented
- ✓ Connection pool configured appropriately
- ✓ CORS settings secure for production
- ✓ Middleware order optimized
- ✓ Error handling comprehensive
- ✓ Monitoring and logging in place
- ✓ Performance benchmarks established

---

## Presentation Notes

**Target Audience:** Backend developers, database architects, DevOps engineers
**Duration:** 65-80 minutes
**Prerequisites:** Database concepts, Python web development
**Learning Objectives:**
- Master ORM concepts and SQLAlchemy/SQLModel
- Implement effective middleware architecture
- Configure secure CORS policies
- Optimize database performance and connections