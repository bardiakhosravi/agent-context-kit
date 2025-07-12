# Domain Driven Design with Ports & Adapters Rules for Python Implementation

## Core Domain Model Rules

### 1. Entity Rules
- Entities MUST have a unique identity that persists throughout their lifecycle
- Use `@dataclass` with `frozen=False` for mutable entities
- Identity should be immutable once set (use `field(init=False)` for auto-generated IDs)
- Implement `__eq__` and `__hash__` based solely on identity, not attributes
- Entities MUST contain business logic as methods, not just data
- Avoid anemic domain models - entities should have behavior

```python
@dataclass
class User:
    id: UserId = field(init=False)
    email: Email
    name: str
    
    def __post_init__(self):
        if not hasattr(self, 'id'):
            self.id = UserId.generate()
    
    def change_email(self, new_email: Email) -> None:
        # Business logic here
        self.email = new_email
```

### 2. Value Object Rules
- Value objects MUST be immutable - use `@dataclass(frozen=True)`
- Equality is based on ALL attributes, not identity
- Should be small, focused, and represent a concept from the domain
- Include validation in `__post_init__` method
- Should have meaningful methods that operate on the value

```python
@dataclass(frozen=True)
class Email:
    value: str
    
    def __post_init__(self):
        if '@' not in self.value:
            raise ValueError("Invalid email format")
    
    @property
    def domain(self) -> str:
        return self.value.split('@')[1]
```

### 3. Aggregate Rules
- Aggregates MUST have a single Aggregate Root (an Entity)
- Only the Aggregate Root should be directly accessible from outside
- Internal entities within an aggregate should be accessed through the root
- Aggregate boundaries should align with transaction boundaries
- Use factory methods on aggregates for complex creation logic
- Aggregates should be small and focused

```python
@dataclass
class Order:  # Aggregate Root
    id: OrderId
    customer_id: CustomerId
    _line_items: list[OrderLineItem] = field(default_factory=list, init=False)
    
    def add_line_item(self, product_id: ProductId, quantity: int) -> None:
        # Business rules and validation
        line_item = OrderLineItem(product_id, quantity)
        self._line_items.append(line_item)
    
    @property
    def line_items(self) -> tuple[OrderLineItem, ...]:
        return tuple(self._line_items)  # Return immutable view
```

### 4. Domain Service Rules
- Create domain services ONLY when business logic doesn't naturally fit in entities or value objects
- Domain services should be stateless
- Use dependency injection for external dependencies
- Should operate on domain objects, not primitives
- Name services with domain language (not technical terms)

```python
class PricingService:
    def __init__(self, discount_repository: DiscountRepository):
        self._discount_repository = discount_repository
    
    def calculate_order_total(self, order: Order, customer: Customer) -> Money:
        # Complex pricing logic that spans multiple aggregates
        pass
```

## Repository Pattern Rules

### 5. Repository Interface Rules
- Define repository interfaces in the domain layer using ABC - they represent domain concepts
- Repositories should work with Aggregate Roots only
- Use domain-specific query methods, not generic CRUD
- Return domain objects, never DTOs or database models
- Should throw domain exceptions, not infrastructure exceptions

```python
# Domain Layer - domain/repositories/user_repository.py
from abc import ABC, abstractmethod

class UserRepository(ABC):
    @abstractmethod
    def find_by_email(self, email: Email) -> Optional[User]:
        pass
    
    @abstractmethod
    def save(self, user: User) -> None:
        pass
    
    @abstractmethod
    def find_active_users_in_department(self, department_id: DepartmentId) -> list[User]:
        pass
    
    @abstractmethod
    def find_by_id(self, user_id: UserId) -> Optional[User]:
        pass
```

### 6. Repository Implementation Rules
- Implement repositories in the infrastructure layer
- Use the Unit of Work pattern for transaction management
- Map between domain objects and persistence models
- Handle optimistic concurrency using version fields
- Repository should not contain business logic

## Domain Event Rules

### 7. Domain Event Rules
- Domain events should be immutable value objects
- Events should represent something that happened in the past (use past tense)
- Events should contain all necessary data to handle the event
- Use `@dataclass(frozen=True)` for events
- Events should be raised by aggregates, not external code

```python
@dataclass(frozen=True)
class UserEmailChanged:
    user_id: UserId
    old_email: Email
    new_email: Email
    occurred_at: datetime
```

### 8. Event Handling Rules
- Domain event handlers should be in the application layer
- Handlers should be idempotent
- Use dependency injection for handler dependencies
- Handlers should not directly modify other aggregates
- Consider eventual consistency for cross-aggregate operations

## Application Service Rules

### 9. Use Case Rules
- Use cases represent single business operations that the application can perform
- Each use case should handle exactly one business workflow
- Use cases orchestrate domain objects but contain no business logic
- Should be stateless and focused on a single responsibility
- Handle cross-cutting concerns (transactions, events, etc.)
- Should not return domain objects directly - use DTOs
- Name use cases after business operations using domain language

```python
class CreateUserUseCase:
    def __init__(
        self, 
        user_repository: UserRepository,
        unit_of_work: UnitOfWork,
        event_publisher: EventPublisher
    ):
        self._user_repository = user_repository
        self._unit_of_work = unit_of_work
        self._event_publisher = event_publisher
    
    def execute(self, command: CreateUserCommand) -> CreateUserResponse:
        with self._unit_of_work:
            # Orchestration logic only
            email = Email(command.email)
            user = User.create(email, command.name)
            self._user_repository.save(user)
            self._event_publisher.publish(UserCreated(user.id, email))
            return CreateUserResponse(user.id.value)

class ChangeUserEmailUseCase:
    def __init__(self, user_repository: UserRepository, unit_of_work: UnitOfWork):
        self._user_repository = user_repository
        self._unit_of_work = unit_of_work
    
    def execute(self, command: ChangeEmailCommand) -> None:
        with self._unit_of_work:
            user = self._user_repository.find_by_id(command.user_id)
            if not user:
                raise UserNotFoundError(command.user_id)
            user.change_email(Email(command.new_email))
            self._user_repository.save(user)
```

## Ports & Adapters (Hexagonal Architecture) Rules

### 10. Port Definition Rules
- Ports define interfaces between layers and external systems
- **Primary ports** (driving) define application use cases - belong in application layer
- **Domain-driven secondary ports** (repositories, domain services) - belong in domain layer  
- **Infrastructure secondary ports** (email, messaging, external APIs) - belong in application layer
- Port interfaces should use domain language, not technical terms
- Ports should be focused and follow Single Responsibility Principle

```python
# Primary Ports (Application Layer) - application/ports/primary/
class CreateUserPort(ABC):
    @abstractmethod
    def execute(self, command: CreateUserCommand) -> CreateUserResponse:
        pass

class ChangeUserEmailPort(ABC):
    @abstractmethod
    def execute(self, command: ChangeEmailCommand) -> None:
        pass

# Domain-Driven Secondary Ports (Domain Layer) - domain/repositories/
class UserRepository(ABC):  # Already shown in rule 5
    @abstractmethod
    def find_by_email(self, email: Email) -> Optional[User]:
        pass

# Domain Services (Domain Layer) - domain/services/
class PricingServicePort(ABC):
    @abstractmethod
    def calculate_product_price(self, product: Product, customer: Customer) -> Money:
        pass

# Infrastructure Secondary Ports (Application Layer) - application/ports/secondary/
class EmailNotificationPort(ABC):
    @abstractmethod
    def send_welcome_email(self, user_email: Email, user_name: str) -> None:
        pass
    
    @abstractmethod
    def send_email_change_notification(self, old_email: Email, new_email: Email) -> None:
        pass

class EventPublisherPort(ABC):
    @abstractmethod
    def publish(self, event: DomainEvent) -> None:
        pass
```

### 11. Primary Adapter Rules
- Primary adapters are the entry points (web controllers, CLI, message consumers)
- Should translate external requests to domain commands/queries
- Must not contain business logic - only translation and validation
- Should handle framework-specific concerns (HTTP status codes, serialization)
- Should be thin and delegate to use cases through primary ports

```python
# FastAPI Controller (Primary Adapter)
from fastapi import APIRouter, HTTPException, Depends, status
from pydantic import BaseModel

class CreateUserRequest(BaseModel):
    email: str
    name: str

class ChangeEmailRequest(BaseModel):
    email: str

class CreateUserResponse(BaseModel):
    user_id: str

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", status_code=status.HTTP_201_CREATED, response_model=CreateUserResponse)
async def create_user(
    request: CreateUserRequest,
    use_case: CreateUserPort = Depends()
) -> CreateUserResponse:
    try:
        command = CreateUserCommand(
            email=request.email,
            name=request.name
        )
        response = use_case.execute(command)
        return CreateUserResponse(user_id=response.user_id)
    except InvalidEmailError as e:
        raise HTTPException(status_code=400, detail=f"Invalid email: {str(e)}")
    except UserAlreadyExistsError as e:
        raise HTTPException(status_code=409, detail=str(e))
    except DomainException as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.patch("/{user_id}/email", status_code=status.HTTP_204_NO_CONTENT)
async def change_user_email(
    user_id: str,
    request: ChangeEmailRequest,
    use_case: ChangeUserEmailPort = Depends()
) -> None:
    try:
        command = ChangeEmailCommand(
            user_id=UserId(user_id),
            new_email=request.email
        )
        use_case.execute(command)
    except UserNotFoundError:
        raise HTTPException(status_code=404, detail="User not found")
    except InvalidEmailError as e:
        raise HTTPException(status_code=400, detail=f"Invalid email: {str(e)}")
    except DomainException as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/{user_id}", response_model=GetUserResponse)
async def get_user(
    user_id: str,
    use_case: GetUserPort = Depends()
) -> GetUserResponse:
    try:
        query = GetUserQuery(user_id=UserId(user_id))
        response = use_case.execute(query)
        return GetUserResponse(
            user_id=response.user_id,
            email=response.email,
            name=response.name
        )
    except UserNotFoundError:
        raise HTTPException(status_code=404, detail="User not found")
    except DomainException as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def deactivate_user(
    user_id: str,
    use_case: DeactivateUserPort = Depends()
) -> None:
    try:
        command = DeactivateUserCommand(user_id=UserId(user_id))
        use_case.execute(command)
    except UserNotFoundError:
        raise HTTPException(status_code=404, detail="User not found")
    except UserAlreadyDeactivatedError as e:
        raise HTTPException(status_code=409, detail=str(e))
    except DomainException as e:
        raise HTTPException(status_code=400, detail=str(e))
```

### 12. Secondary Adapter Rules
- Secondary adapters implement secondary ports defined in domain/application layers
- Organize secondary adapters by technology for shared infrastructure and easier maintenance
- Should handle all external system complexities (database mapping, API calls, etc.)
- Must translate between domain objects and external representations
- Should not expose external system details to the domain
- Include error handling and retry logic when appropriate
- Keep technology-specific models/schemas within their adapter implementations

```python
# SQL Database Adapter - infrastructure/adapters/secondary/sql/sql_user_repository.py
class SqlUserRepository(UserRepository):
    def __init__(self, session: Session):
        self._session = session
    
    def save(self, user: User) -> None:
        user_model = UserModel(
            id=user.id.value,
            email=user.email.value,
            name=user.name,
            version=user.version
        )
        self._session.merge(user_model)
    
    def find_by_email(self, email: Email) -> Optional[User]:
        model = self._session.query(UserModel).filter_by(email=email.value).first()
        return self._to_domain(model) if model else None
    
    def _to_domain(self, model: UserModel) -> User:
        return User(
            id=UserId(model.id),
            email=Email(model.email),
            name=model.name
        )

# HTTP External Service Adapter - infrastructure/adapters/secondary/http/http_email_service.py
class HttpEmailNotificationAdapter(EmailNotificationPort):
    def __init__(self, http_client: HTTPClient, api_config: EmailAPIConfig):
        self._http_client = http_client
        self._api_config = api_config
    
    def send_welcome_email(self, user_email: Email, user_name: str) -> None:
        payload = {
            'to': user_email.value,
            'template': 'welcome',
            'variables': {'name': user_name}
        }
        response = self._http_client.post(
            f"{self._api_config.base_url}/send",
            json=payload,
            headers={'Authorization': f'Bearer {self._api_config.api_key}'}
        )
        if response.status_code != 200:
            raise EmailDeliveryError(f"Failed to send email: {response.text}")
```

### 13. Adapter Configuration Rules
- Use Dependency Injection container to wire adapters to ports
- Configuration should happen at application startup in a composition root
- Adapters should be configurable through environment variables or config files
- Use factory patterns for complex adapter creation
- Keep configuration separate from business logic
- Configuration layer manages all technology-specific adapter instantiation

```python
# Configuration Layer - configuration/di_container.py
class DIContainer:
    def __init__(self, config: Config):
        self._config = config
        self._sql_session = self._create_sql_session()
        self._mongo_client = self._create_mongo_client()
        self._http_client = self._create_http_client()
    
    # Domain port implementations (SQL technology)
    def sql_user_repository(self) -> UserRepository:
        return SqlUserRepository(self._sql_session)
    
    # Domain port implementations (MongoDB technology)  
    def mongo_user_repository(self) -> UserRepository:
        return MongoUserRepository(self._mongo_client)
    
    # Infrastructure port implementations
    def http_email_notification_service(self) -> EmailNotificationPort:
        return HttpEmailNotificationAdapter(
            self._http_client,
            self._config.email_api_config
        )
    
    def rabbitmq_event_publisher(self) -> EventPublisherPort:
        return RabbitMqEventPublisher(
            connection=self._create_rabbitmq_connection()
        )
    
    # Use case implementations
    def create_user_use_case(self) -> CreateUserPort:
        return CreateUserUseCase(
            user_repository=self.sql_user_repository(),  # Choose technology
            email_service=self.http_email_notification_service(),
            event_publisher=self.rabbitmq_event_publisher(),
            unit_of_work=UnitOfWork(self._sql_session)
        )
```

## Integrated Project Structure Rules

## Integrated Project Structure Rules

### 14. Hexagonal Package Structure Rules
- Organize by hexagonal architecture layers with clear port placement
- **Domain ports**: Repository interfaces and domain service ports in domain layer
- **Application ports**: Primary ports (use cases) and infrastructure service ports in application layer
- **Secondary adapters**: Organize by technology for shared infrastructure and easier maintenance
- **Technology-specific models**: Persistence models live within their respective technology adapters
- **Configuration layer**: Cross-cutting concerns that wire all layers together
- Domain and application layers should only depend on their respective port interfaces
- Infrastructure layer contains all adapter implementations

```
src/
├── domain/
│   ├── model/
│   │   ├── user/
│   │   │   ├── user.py              # Entity
│   │   │   └── email.py             # Value Object
│   │   └── order/
│   ├── ports/                       # Domain Ports (Secondary)
│   │   ├── user_repository.py       # Repository interface (domain concept)
│   │   ├── order_repository.py
│   │   ├── pricing_service_port.py  # Domain service interface
│   │   ├── inventory_service_port.py
│   │   └── domain_event_store_port.py  # Domain-specific event storage
│   └── events/                      # Domain Events
├── application/
│   ├── ports/
│   │   ├── primary/                 # Primary Ports (Use Cases)
│   │   │   ├── create_user_port.py          # Primary Port
│   │   │   ├── change_user_email_port.py    # Primary Port
│   │   │   └── deactivate_user_port.py      # Primary Port
│   │   └── secondary/               # Infrastructure Ports (Secondary)
│   │       ├── email_notification_port.py   # Infrastructure service
│   │       ├── event_publisher_port.py      # Infrastructure service
│   │       └── payment_gateway_port.py      # Infrastructure service
│   ├── use_cases/
│   │   ├── create_user_use_case.py          # Use Case (Primary Port Implementation)
│   │   ├── change_user_email_use_case.py    # Use Case (Primary Port Implementation)
│   │   └── deactivate_user_use_case.py      # Use Case (Primary Port Implementation)
│   ├── commands/
│   ├── queries/
│   └── handlers/
├── infrastructure/
│   └── adapters/
│       ├── primary/
│       │   ├── web/
│       │   │   ├── user_controller.py       # Primary Adapter
│       │   │   └── order_controller.py
│       │   ├── cli/
│       │   └── messaging/
│       └── secondary/                       # Organized by Technology
│           ├── sql/
│           │   ├── models/                          # SQLAlchemy models
│           │   │   ├── user_model.py
│           │   │   └── order_model.py
│           │   ├── base_sql_repository.py          # Shared base class
│           │   ├── sql_connection_manager.py       # Shared connection handling
│           │   ├── sql_user_repository.py          # Implements UserRepository
│           │   ├── sql_order_repository.py         # Implements OrderRepository
│           │   └── sql_domain_event_store.py       # Implements DomainEventStorePort
│           ├── mongodb/
│           │   ├── schemas/                         # MongoDB schemas
│           │   │   ├── user_schema.py
│           │   │   └── order_schema.py
│           │   ├── mongo_connection.py             # Shared connection
│           │   ├── mongo_user_repository.py        # Implements UserRepository
│           │   └── mongo_order_repository.py       # Implements OrderRepository
│           ├── http/
│           │   ├── base_http_client.py             # Shared HTTP utilities
│           │   ├── http_retry_policy.py            # Shared retry logic
│           │   ├── http_pricing_service.py         # Implements PricingServicePort
│           │   ├── http_payment_gateway.py         # Implements PaymentGatewayPort
│           │   └── http_email_service.py           # Implements EmailNotificationPort
│           ├── messaging/
│           │   ├── rabbitmq_connection.py          # Shared connection
│           │   ├── rabbitmq_event_publisher.py     # Implements EventPublisherPort
│           │   └── rabbitmq_notification_sender.py # Implements NotificationPort
│           └── redis/
│               ├── redis_connection.py             # Shared connection
│               ├── redis_cache_service.py          # Implements CacheServicePort
│               └── redis_session_store.py          # Implements SessionStorePort
└── configuration/                           # Cross-cutting Configuration Layer
    ├── di_container.py                      # Dependency injection container
    ├── database_config.py                  # Database configuration
    ├── app_settings.py                     # Application settings
    └── environment_config.py               # Environment-specific config
```

### 15. Integration Flow Rules
- Primary adapters call primary ports (use cases)
- Use cases orchestrate domain objects and use secondary ports for external systems
- Secondary adapters implement secondary ports and handle external complexities
- Domain objects should never directly depend on adapters
- Use events for loose coupling between bounded contexts

```python
# Flow Example: Web Request → Controller → Use Case → Repository
class UserController:  # Primary Adapter
    def create_user(self, request) -> Response:
        command = CreateUserCommand(request.email, request.name)
        response = self._create_user_use_case.execute(command)  # → Primary Port
        return Response(201, {'user_id': response.user_id})

class CreateUserUseCase(CreateUserPort):  # Primary Port Implementation
    def execute(self, command: CreateUserCommand) -> CreateUserResponse:
        email = Email(command.email)
        user = User.create(email, command.name)
        self._user_repository.save(user)  # → Secondary Port
        self._email_service.send_welcome_email(email, command.name)  # → Secondary Port
        return CreateUserResponse(user.id.value)
```

## Synergy Rules for DDD + Ports & Adapters

### 16. Repository as Secondary Port Rules
- Repository interfaces are domain ports (interfaces in domain layer)
- Repository implementations are secondary adapters (in infrastructure layer)
- Repositories should work with Aggregate Roots and use domain language
- Repository adapters handle ORM mapping and database specifics
- Keep repository interfaces focused on domain needs, not database capabilities

### 17. Use Case as Primary Port Implementation Rules
- Use cases implement primary ports and orchestrate domain objects
- They use both domain ports (repositories) and infrastructure ports (email, messaging)
- Should not contain business logic - delegate to domain objects
- Handle cross-cutting concerns like transactions and event publishing
- Serve as the application's use case boundary
- Each use case should represent exactly one business workflow

```python
class CreateUserUseCase(CreateUserPort):
    def __init__(
        self,
        user_repository: UserRepository,  # Domain port
        email_service: EmailNotificationPort,  # Infrastructure port
        event_publisher: EventPublisherPort,  # Infrastructure port
        unit_of_work: UnitOfWork
    ):
        self._user_repository = user_repository
        self._email_service = email_service
        self._event_publisher = event_publisher
        self._unit_of_work = unit_of_work
    
    def execute(self, command: CreateUserCommand) -> CreateUserResponse:
        with self._unit_of_work:
            email = Email(command.email)
            user = User.create(email, command.name)
            self._user_repository.save(user)  # Domain port
            self._email_service.send_welcome_email(email, command.name)  # Infrastructure port
            self._event_publisher.publish(UserCreated(user.id, email))  # Infrastructure port
            return CreateUserResponse(user.id.value)
```

### 18. Event Integration Rules
- Domain events should be published through infrastructure ports
- Event handlers can be implemented as separate use cases
- Use event-driven architecture for cross-bounded context communication
- Events enable loose coupling between adapters and domain logic
- Consider eventual consistency for distributed operations

```python
# Event Publishing through Infrastructure Port
class EventPublisherPort(ABC):  # Application layer
    @abstractmethod
    def publish(self, event: DomainEvent) -> None:
        pass

class CreateUserUseCase(CreateUserPort):
    def __init__(
        self, 
        user_repo: UserRepository,  # Domain port
        event_publisher: EventPublisherPort  # Infrastructure port
    ):
        self._user_repo = user_repo
        self._event_publisher = event_publisher
    
    def execute(self, command: CreateUserCommand) -> CreateUserResponse:
        user = User.create(Email(command.email), command.name)
        self._user_repo.save(user)  # Domain port
        self._event_publisher.publish(UserCreated(user.id, user.email))  # Infrastructure port
        return CreateUserResponse(user.id.value)

# Event Handler as Use Case
class SendWelcomeEmailUseCase:
    def __init__(self, email_service: EmailNotificationPort):  # Infrastructure port
        self._email_service = email_service
    
    def handle(self, event: UserCreated) -> None:
        self._email_service.send_welcome_email(event.email, event.name)
```

### 19. Cross-Cutting Concern Rules
- Handle infrastructure concerns (logging, metrics, caching) in adapters
- Use decorators or middleware patterns for cross-cutting concerns
- Keep domain objects free from infrastructure dependencies
- Implement concerns like retries, circuit breakers in secondary adapters
- Use aspect-oriented patterns at the adapter boundaries

```python
# Decorator for logging in adapters
def logged_repository(func):
    def wrapper(*args, **kwargs):
        logger.info(f"Repository operation: {func.__name__}")
        try:
            result = func(*args, **kwargs)
            logger.info(f"Operation successful: {func.__name__}")
            return result
        except Exception as e:
            logger.error(f"Operation failed: {func.__name__}, error: {e}")
            raise
    return wrapper

class SqlUserRepository(UserRepository):
    @logged_repository
    def save(self, user: User) -> None:
        # Implementation
        pass
```

## Validation and Error Handling Rules
- Test domain logic in isolation without any adapters
- Test primary adapters by mocking primary ports
- Test secondary adapters by mocking external dependencies
- Use in-memory implementations of secondary ports for integration tests
- Test the full flow from primary adapter to secondary adapter for end-to-end tests

```python
# Testing with port isolation
class TestUserManagementService:
    def test_create_user_success(self):
        # Arrange
        mock_repo = Mock(spec=UserRepository)
        mock_events = Mock(spec=EventPublisherPort)
        service = UserManagementService(mock_repo, mock_events)
        
        # Act
        result = service.create_user(CreateUserCommand("test@example.com", "John"))
        
        # Assert
        mock_repo.save.assert_called_once()
        mock_events.publish.assert_called_once()
        assert isinstance(result.user_id, str)

# In-memory adapter for testing
class InMemoryUserRepository(UserRepository):
    def __init__(self):
        self._users: dict[UserId, User] = {}
    
    def save(self, user: User) -> None:
        self._users[user.id] = user
    
    def find_by_email(self, email: Email) -> Optional[User]:
        return next((u for u in self._users.values() if u.email == email), None)
```

### 21. Validation and Error Handling Rules
- Domain validation should happen in domain objects (entities, value objects)
- Use domain exceptions that extend a base domain exception
- Validation should be explicit and fail fast
- Input validation in application services should be minimal
- Use factory methods for complex validation scenarios

```python
class DomainException(Exception):
    pass

class InvalidEmailError(DomainException):
    pass

@dataclass(frozen=True)
class Email:
    value: str
    
    def __post_init__(self):
        if not self._is_valid_email(self.value):
            raise InvalidEmailError(f"Invalid email: {self.value}")
```

### 22. Naming Convention Rules
- Use domain language (Ubiquitous Language) for all class and method names
- Avoid technical terms in domain layer (no "Manager", "Helper", "Util")  
- Use intention-revealing names for methods
- Value objects should be named after the concept they represent
- Repository methods should reflect business queries
- **Port Naming**: End primary ports with "Port", secondary ports with "Port"
- **Adapter Naming**: Include the technology/framework in secondary adapter names
- **Clear Port vs Adapter distinction**: Ports define interfaces, Adapters implement them

```python
# Good port names
class UserManagementPort(ABC): pass           # Primary port
class EmailNotificationPort(ABC): pass       # Secondary port
class PaymentProcessingPort(ABC): pass       # Secondary port

# Good adapter names  
class RestUserController:                    # Primary adapter (REST)
class GraphQLUserController:                 # Primary adapter (GraphQL)
class SqlUserRepository(UserRepository):     # Secondary adapter (SQL)
class MongoUserRepository(UserRepository):   # Secondary adapter (MongoDB)
class SmtpEmailAdapter(EmailNotificationPort): # Secondary adapter (SMTP)
class SendGridEmailAdapter(EmailNotificationPort): # Secondary adapter (SendGrid)
```

### 23. Dependency Rules
- Domain layer should have no external dependencies except standard library
- Application layer can depend on domain but should use dependency inversion for external concerns
- Infrastructure layer implements all external dependencies through adapters
- **Domain Port Dependencies**: Domain objects can depend on domain ports (repositories, domain services)
- **Infrastructure Port Dependencies**: Use cases depend on infrastructure ports for external concerns
- **Port Placement**: Domain ports in domain layer, infrastructure ports in application layer
- **Inversion of Control**: Use DI container to wire adapters to ports at startup
- Use dependency inversion - depend on abstractions, not concretions
- Inject dependencies through constructors
- Use factory pattern for complex object creation

```python
# Domain layer - can depend on domain ports
class User:  # Domain entity
    def __init__(self, id: UserId, email: Email, name: str):
        self.id = id
        self.email = email
        self.name = name
    
    def change_email(self, new_email: Email) -> None:
        # Business logic here
        self.email = new_email

class UserDomainService:  # Domain service
    def __init__(self, user_repo: UserRepository):  # Domain port dependency
        self._user_repo = user_repo
    
    def is_email_unique(self, email: Email) -> bool:
        existing_user = self._user_repo.find_by_email(email)
        return existing_user is None

# Application layer - depends on domain + infrastructure ports
class CreateUserUseCase(CreateUserPort):
    def __init__(
        self, 
        user_repo: UserRepository,  # Domain port
        user_domain_service: UserDomainService,  # Domain service
        email_service: EmailNotificationPort,  # Infrastructure port
        event_publisher: EventPublisherPort  # Infrastructure port
    ):
        self._user_repo = user_repo
        self._user_domain_service = user_domain_service
        self._email_service = email_service
        self._event_publisher = event_publisher

# Infrastructure layer - implements ports with external dependencies
class SqlUserRepository(UserRepository):  # Implements domain port
    def __init__(self, session: SqlAlchemySession):  # External dependency
        self._session = session

class SmtpEmailAdapter(EmailNotificationPort):  # Implements infrastructure port
    def __init__(self, smtp_client: SMTPClient):  # External dependency
        self._smtp_client = smtp_client
```

### 24. Testing Rules
- Write unit tests for domain logic without mocking domain objects
- **Test Ports in Isolation**: Mock secondary ports when testing use cases
- **Test Adapters Separately**: Test each adapter implementation independently
- **Integration Testing**: Use in-memory adapters for full workflow testing
- Test domain events are raised correctly
- Integration tests should test aggregate boundaries
- Use builders or factories for test data creation
- **Contract Testing**: Ensure all adapter implementations satisfy their port contracts
- **Use Case Testing**: Test each use case independently with mocked dependencies

```python
# Contract test for all UserRepository implementations
class UserRepositoryContractTest:
    def test_save_and_find_user(self, repository: UserRepository):
        # This test should pass for SqlUserRepository, MongoUserRepository, etc.
        user = User.create(Email("test@example.com"), "John")
        repository.save(user)
        found = repository.find_by_email(Email("test@example.com"))
        assert found is not None
        assert found.email == user.email

# Use Case Integration Test
class TestCreateUserUseCaseIntegration:
    def test_full_workflow_with_in_memory_adapters(self):
        # Arrange
        user_repo = InMemoryUserRepository()
        email_service = InMemoryEmailService()
        event_publisher = InMemoryEventPublisher()
        use_case = CreateUserUseCase(user_repo, email_service, event_publisher)
        
        # Act
        result = use_case.execute(CreateUserCommand("test@example.com", "John"))
        
        # Assert
        assert result.user_id is not None
        saved_user = user_repo.find_by_email(Email("test@example.com"))
        assert saved_user is not None
        assert len(email_service.sent_emails) == 1
        assert len(event_publisher.published_events) == 1

# Technology-specific adapter testing
class TestSqlUserRepository:
    def test_save_user_with_sql_models(self):
        # Arrange
        session = create_test_sql_session()
        repository = SqlUserRepository(session)
        user = User.create(Email("test@example.com"), "John")
        
        # Act
        repository.save(user)
        
        # Assert
        saved_user = repository.find_by_email(Email("test@example.com"))
        assert saved_user is not None
        assert saved_user.email == user.email
        
        # Verify SQL model was created correctly
        user_model = session.query(UserModel).filter_by(email="test@example.com").first()
        assert user_model is not None
        assert user_model.name == "John"

class TestMongoUserRepository:
    def test_save_user_with_mongo_schemas(self):
        # Arrange
        mongo_client = create_test_mongo_client()
        repository = MongoUserRepository(mongo_client)
        user = User.create(Email("test@example.com"), "John")
        
        # Act
        repository.save(user)
        
        # Assert
        saved_user = repository.find_by_email(Email("test@example.com"))
        assert saved_user is not None
        assert saved_user.email == user.email
```

These integrated rules ensure that Domain Driven Design and Ports & Adapters (Hexagonal Architecture) work together seamlessly in Python implementations. The combination provides clean separation of concerns, testability, and flexibility while maintaining domain focus and proper dependency management.