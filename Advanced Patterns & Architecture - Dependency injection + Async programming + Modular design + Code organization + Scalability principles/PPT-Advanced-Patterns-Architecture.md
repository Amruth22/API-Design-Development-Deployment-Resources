# Advanced Patterns & Architecture - Dependency Injection + Async Programming + Modular Design

## Professional PowerPoint Presentation

---

## Slide 1: Title Slide

### Advanced Patterns & Architecture
#### Dependency Injection, Async Programming, Modular Design & Scalability

**Enterprise-Grade Software Architecture Principles**

*Professional Training Series*

---

## Slide 2: Dependency Injection Fundamentals

### Inversion of Control Pattern

**What is Dependency Injection?**
- Design pattern that implements Inversion of Control (IoC)
- Dependencies are provided to an object rather than created by it
- Promotes loose coupling and testability
- Enables better separation of concerns

**Without Dependency Injection:**
```python
class UserService:
    def __init__(self):
        self.db = PostgreSQLDatabase()  # Tight coupling
        self.email = SMTPEmailService()  # Hard to test
    
    def create_user(self, user_data):
        user = self.db.save(user_data)
        self.email.send_welcome(user.email)
        return user
```

**With Dependency Injection:**
```python
class UserService:
    def __init__(self, db: Database, email: EmailService):
        self.db = db  # Loose coupling
        self.email = email  # Easy to mock/test
    
    def create_user(self, user_data):
        user = self.db.save(user_data)
        self.email.send_welcome(user.email)
        return user
```

---

## Slide 3: Dependency Injection Implementation

### FastAPI Dependency System

**Basic Dependency:**
```python
from fastapi import FastAPI, Depends

def get_database():
    db = Database()
    try:
        yield db
    finally:
        db.close()

@app.post("/users")
async def create_user(
    user: User,
    db: Database = Depends(get_database)
):
    return db.create_user(user)
```

**Class-Based Dependencies:**
```python
class DatabaseService:
    def __init__(self):
        self.connection = create_connection()
    
    def get_user(self, user_id: int):
        return self.connection.query(f"SELECT * FROM users WHERE id = {user_id}")

class UserService:
    def __init__(self, db: DatabaseService = Depends()):
        self.db = db

@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    user_service: UserService = Depends()
):
    return user_service.get_user(user_id)
```

**Benefits:**
- Testability through mock injection
- Configuration flexibility
- Resource management (connection pooling)
- Clean separation of concerns

---

## Slide 4: Asynchronous Programming Concepts

### Understanding Async/Await

**Synchronous vs Asynchronous:**

**Synchronous Execution:**
```python
def fetch_user_data(user_id):
    user = database.get_user(user_id)      # Blocks for 100ms
    profile = api.get_profile(user_id)     # Blocks for 200ms
    settings = cache.get_settings(user_id) # Blocks for 50ms
    return combine_data(user, profile, settings)
# Total time: 350ms
```

**Asynchronous Execution:**
```python
async def fetch_user_data(user_id):
    user_task = database.get_user(user_id)
    profile_task = api.get_profile(user_id)
    settings_task = cache.get_settings(user_id)
    
    user, profile, settings = await asyncio.gather(
        user_task, profile_task, settings_task
    )
    return combine_data(user, profile, settings)
# Total time: 200ms (max of individual operations)
```

**Key Concepts:**
- **Coroutines:** Functions defined with async def
- **Await:** Pause execution until operation completes
- **Event Loop:** Manages and executes async operations
- **Concurrency:** Multiple operations in progress simultaneously

---

## Slide 5: Async Programming in APIs

### FastAPI Async Implementation

**Async Route Handlers:**
```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # Non-blocking database call
    user = await db.fetch_user(user_id)
    
    # Concurrent external API calls
    profile_task = external_api.get_profile(user_id)
    settings_task = cache_service.get_settings(user_id)
    
    profile, settings = await asyncio.gather(
        profile_task, settings_task
    )
    
    return {
        "user": user,
        "profile": profile,
        "settings": settings
    }
```

**Async Database Operations:**
```python
import asyncpg
from databases import Database

database = Database("postgresql://user:pass@localhost/db")

async def get_users():
    query = "SELECT * FROM users WHERE active = :active"
    return await database.fetch_all(query, values={"active": True})

async def create_user(user_data):
    query = """
        INSERT INTO users (name, email) 
        VALUES (:name, :email) 
        RETURNING id
    """
    return await database.fetch_one(query, values=user_data)
```

**Performance Benefits:**
- Higher throughput for I/O-bound operations
- Better resource utilization
- Improved scalability under load

---

## Slide 6: Modular Design Principles

### Separation of Concerns

**Layered Architecture:**
```
┌─────────────────────┐
│   Presentation      │  ← API Routes, Request/Response
├─────────────────────┤
│   Business Logic    │  ← Services, Domain Logic
├─────────────────────┤
│   Data Access       │  ← Repositories, ORM
├─────────────────────┤
│   Infrastructure    │  ← Database, External APIs
└─────────────────────┘
```

**Module Structure Example:**
```
project/
├── api/
│   ├── routes/
│   │   ├── users.py
│   │   ├── products.py
│   │   └── orders.py
│   └── dependencies.py
├── services/
│   ├── user_service.py
│   ├── product_service.py
│   └── order_service.py
├── repositories/
│   ├── user_repository.py
│   └── product_repository.py
├── models/
│   ├── user.py
│   └── product.py
└── core/
    ├── config.py
    ├── database.py
    └── exceptions.py
```

---

## Slide 7: Repository Pattern Implementation

### Data Access Abstraction

**Repository Interface:**
```python
from abc import ABC, abstractmethod
from typing import List, Optional

class UserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> Optional[User]:
        pass
    
    @abstractmethod
    async def get_all(self) -> List[User]:
        pass
    
    @abstractmethod
    async def create(self, user: User) -> User:
        pass
    
    @abstractmethod
    async def update(self, user: User) -> User:
        pass
    
    @abstractmethod
    async def delete(self, user_id: int) -> bool:
        pass
```

**Concrete Implementation:**
```python
class SQLUserRepository(UserRepository):
    def __init__(self, database: Database):
        self.db = database
    
    async def get_by_id(self, user_id: int) -> Optional[User]:
        query = "SELECT * FROM users WHERE id = :id"
        row = await self.db.fetch_one(query, {"id": user_id})
        return User(**row) if row else None
    
    async def create(self, user: User) -> User:
        query = """
            INSERT INTO users (name, email) 
            VALUES (:name, :email) 
            RETURNING *
        """
        row = await self.db.fetch_one(query, user.dict())
        return User(**row)
```

**Benefits:**
- Database independence
- Easy testing with mock repositories
- Clean separation of data access logic

---

## Slide 8: Service Layer Pattern

### Business Logic Organization

**Service Layer Implementation:**
```python
class UserService:
    def __init__(
        self,
        user_repo: UserRepository,
        email_service: EmailService,
        cache_service: CacheService
    ):
        self.user_repo = user_repo
        self.email_service = email_service
        self.cache_service = cache_service
    
    async def create_user(self, user_data: UserCreate) -> User:
        # Business logic validation
        if await self.user_repo.get_by_email(user_data.email):
            raise UserAlreadyExistsError()
        
        # Create user
        user = await self.user_repo.create(User(**user_data.dict()))
        
        # Side effects
        await self.email_service.send_welcome_email(user.email)
        await self.cache_service.invalidate_user_cache()
        
        return user
    
    async def get_user_profile(self, user_id: int) -> UserProfile:
        # Try cache first
        cached = await self.cache_service.get_user_profile(user_id)
        if cached:
            return cached
        
        # Fetch from database
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            raise UserNotFoundError()
        
        profile = UserProfile.from_user(user)
        await self.cache_service.set_user_profile(user_id, profile)
        
        return profile
```

---

## Slide 9: Error Handling Patterns

### Robust Exception Management

**Custom Exception Hierarchy:**
```python
class APIException(Exception):
    def __init__(self, message: str, status_code: int = 500):
        self.message = message
        self.status_code = status_code
        super().__init__(self.message)

class ValidationError(APIException):
    def __init__(self, message: str):
        super().__init__(message, 400)

class NotFoundError(APIException):
    def __init__(self, resource: str, identifier: str):
        message = f"{resource} with id {identifier} not found"
        super().__init__(message, 404)

class BusinessLogicError(APIException):
    def __init__(self, message: str):
        super().__init__(message, 422)
```

**Global Exception Handler:**
```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

@app.exception_handler(APIException)
async def api_exception_handler(request: Request, exc: APIException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "message": exc.message,
                "type": exc.__class__.__name__,
                "path": request.url.path
            }
        }
    )

@app.exception_handler(ValidationError)
async def validation_exception_handler(request: Request, exc: ValidationError):
    return JSONResponse(
        status_code=400,
        content={
            "error": {
                "message": "Validation failed",
                "details": exc.message
            }
        }
    )
```

---

## Slide 10: Scalability Principles

### Designing for Growth

**Horizontal Scaling Strategies:**
- **Stateless Design:** No server-side session storage
- **Database Sharding:** Distribute data across multiple databases
- **Caching Layers:** Redis, Memcached for frequently accessed data
- **Load Balancing:** Distribute requests across multiple instances
- **Microservices:** Break monolith into independent services

**Vertical Scaling Optimizations:**
- **Connection Pooling:** Reuse database connections
- **Async Processing:** Non-blocking I/O operations
- **Background Tasks:** Offload heavy processing
- **Query Optimization:** Efficient database queries
- **Resource Management:** Memory and CPU optimization

**Caching Strategy:**
```python
from functools import wraps
import redis

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cache_result(expiration: int = 300):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key
            cache_key = f"{func.__name__}:{hash(str(args) + str(kwargs))}"
            
            # Try to get from cache
            cached_result = redis_client.get(cache_key)
            if cached_result:
                return json.loads(cached_result)
            
            # Execute function and cache result
            result = await func(*args, **kwargs)
            redis_client.setex(
                cache_key, 
                expiration, 
                json.dumps(result, default=str)
            )
            
            return result
        return wrapper
    return decorator

@cache_result(expiration=600)
async def get_user_statistics(user_id: int):
    # Expensive computation
    return await compute_user_stats(user_id)
```

---

## Slide 11: Testing Strategies

### Comprehensive Testing Approach

**Unit Testing with Dependency Injection:**
```python
import pytest
from unittest.mock import AsyncMock

class TestUserService:
    @pytest.fixture
    def mock_user_repo(self):
        return AsyncMock(spec=UserRepository)
    
    @pytest.fixture
    def mock_email_service(self):
        return AsyncMock(spec=EmailService)
    
    @pytest.fixture
    def user_service(self, mock_user_repo, mock_email_service):
        return UserService(mock_user_repo, mock_email_service)
    
    async def test_create_user_success(self, user_service, mock_user_repo):
        # Arrange
        user_data = UserCreate(name="John", email="john@example.com")
        expected_user = User(id=1, name="John", email="john@example.com")
        mock_user_repo.get_by_email.return_value = None
        mock_user_repo.create.return_value = expected_user
        
        # Act
        result = await user_service.create_user(user_data)
        
        # Assert
        assert result == expected_user
        mock_user_repo.create.assert_called_once()
```

**Integration Testing:**
```python
from fastapi.testclient import TestClient

def test_create_user_integration():
    with TestClient(app) as client:
        response = client.post(
            "/users",
            json={"name": "John Doe", "email": "john@example.com"}
        )
        assert response.status_code == 201
        assert response.json()["name"] == "John Doe"
```

---

## Slide 12: Summary and Best Practices

### Architecture Excellence Guidelines

**Key Architectural Principles:**
- **Single Responsibility:** Each class/module has one reason to change
- **Dependency Inversion:** Depend on abstractions, not concretions
- **Open/Closed:** Open for extension, closed for modification
- **Interface Segregation:** Many specific interfaces vs. one general
- **DRY (Don't Repeat Yourself):** Eliminate code duplication

**Implementation Checklist:**
- ✓ Use dependency injection for loose coupling
- ✓ Implement async/await for I/O-bound operations
- ✓ Organize code into logical modules and layers
- ✓ Create comprehensive error handling strategy
- ✓ Design for horizontal and vertical scaling
- ✓ Implement caching at appropriate levels
- ✓ Write testable code with proper abstractions
- ✓ Monitor and measure performance continuously

**Performance Optimization:**
- Profile before optimizing
- Cache frequently accessed data
- Use connection pooling
- Implement proper indexing
- Monitor resource usage
- Plan for capacity growth

**Maintainability Focus:**
- Clear naming conventions
- Comprehensive documentation
- Consistent code style
- Regular refactoring
- Automated testing
- Code review processes

---

## Presentation Notes

**Target Audience:** Senior developers, architects, technical leads
**Duration:** 60-75 minutes
**Prerequisites:** Advanced Python knowledge, web development experience
**Learning Objectives:**
- Master dependency injection patterns
- Implement async programming effectively
- Design scalable modular architectures
- Apply enterprise-grade best practices