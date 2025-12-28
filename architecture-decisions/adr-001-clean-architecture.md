# ADR-001: Clean Architecture (Onion Architecture)

## Status
**Accepted** - December 28, 2025

## Context
We need a consistent architecture pattern across all .NET projects that:
- Promotes separation of concerns
- Makes code testable and maintainable
- Keeps business logic independent of frameworks
- Allows easy replacement of external dependencies
- Supports team scalability and code consistency

## Decision
We will adopt **Clean Architecture** (Onion Architecture) as our standard architectural pattern for all .NET applications.

### Layer Structure

#### Layer 1: Domain (Core)
**Purpose**: Pure business logic and entities  
**Dependencies**: None  
**Location**: `ProjectName.Domain` project

**Contains**:
- Domain entities with business logic
- Value objects
- Domain events
- Domain exceptions
- Repository and service interfaces (abstractions)
- Business rules and invariants

**Example**:
```csharp
// Domain/Entities/Customer.cs
public class Customer : BaseEntity
{
    public string FirstName { get; private set; }
    public string Email { get; private set; }
    
    public static Customer Create(string firstName, string email)
    {
        if (string.IsNullOrWhiteSpace(firstName))
            throw new DomainException("First name is required");
        
        return new Customer { FirstName = firstName, Email = email };
    }
    
    public void UpdateEmail(string newEmail)
    {
        if (!IsValidEmail(newEmail))
            throw new DomainException("Invalid email");
        Email = newEmail;
    }
}

// Domain/Interfaces/ICustomerRepository.cs
public interface ICustomerRepository : IRepository<Customer>
{
    Task<Customer?> GetByEmailAsync(string email);
}
```

#### Layer 2: Application
**Purpose**: Application business rules and use cases  
**Dependencies**: Domain only  
**Location**: `ProjectName.Application` project

**Contains**:
- Use cases / Commands / Queries (CQRS pattern)
- DTOs and view models
- Application service interfaces
- Validators (FluentValidation)
- Mapping profiles (AutoMapper)
- Application exceptions

**Example**:
```csharp
// Application/Commands/CreateCustomerCommand.cs
public record CreateCustomerCommand(
    string FirstName,
    string LastName,
    string Email) : IRequest<Result<int>>;

// Application/Handlers/CreateCustomerCommandHandler.cs
public class CreateCustomerCommandHandler 
    : IRequestHandler<CreateCustomerCommand, Result<int>>
{
    private readonly ICustomerRepository _repository;
    
    public CreateCustomerCommandHandler(ICustomerRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<Result<int>> Handle(
        CreateCustomerCommand request,
        CancellationToken cancellationToken)
    {
        var customer = Customer.Create(
            request.FirstName,
            request.Email);
        
        var created = await _repository.AddAsync(customer);
        return Result<int>.Success(created.Id);
    }
}
```

#### Layer 3: Infrastructure
**Purpose**: External concerns and data access  
**Dependencies**: Application + Domain  
**Location**: `ProjectName.Infrastructure` project

**Contains**:
- EF Core DbContext and configurations
- Repository implementations
- External API clients
- File system access
- Email/SMS services
- Caching implementations
- Database migrations

**Example**:
```csharp
// Infrastructure/Data/ApplicationDbContext.cs
public class ApplicationDbContext : DbContext
{
    public DbSet<Customer> Customers => Set<Customer>();
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(
            Assembly.GetExecutingAssembly());
    }
}

// Infrastructure/Repositories/CustomerRepository.cs
public class CustomerRepository : Repository<Customer>, ICustomerRepository
{
    public CustomerRepository(ApplicationDbContext context) : base(context) { }
    
    public async Task<Customer?> GetByEmailAsync(string email)
    {
        return await _dbSet
            .FirstOrDefaultAsync(c => c.Email == email);
    }
}
```

#### Layer 4: Presentation (API/UI)
**Purpose**: User interface and API endpoints  
**Dependencies**: Application + Infrastructure  
**Location**: `ProjectName.Api` or `ProjectName.Web` project

**Contains**:
- Controllers / Minimal APIs
- Request/Response models
- Middleware
- Filters and attributes
- Program.cs / Startup.cs
- Dependency injection setup

**Example**:
```csharp
// API/Controllers/CustomersController.cs
[ApiController]
[Route("api/[controller]")]
public class CustomersController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public CustomersController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    [HttpPost]
    public async Task<IActionResult> Create(CreateCustomerCommand command)
    {
        var result = await _mediator.Send(command);
        
        if (!result.IsSuccess)
            return BadRequest(result.Error);
        
        return CreatedAtAction(
            nameof(GetById),
            new { id = result.Value },
            result.Value);
    }
}
```

## Dependency Flow
```
┌─────────────────────────────────┐
│      Presentation Layer         │  ← Controllers, UI
│  (Depends on Application)       │
└─────────────────────────────────┘
            ↓
┌─────────────────────────────────┐
│      Application Layer          │  ← Use Cases, DTOs
│  (Depends on Domain)            │
└─────────────────────────────────┘
            ↓
┌─────────────────────────────────┐
│      Domain Layer               │  ← Entities, Business Logic
│  (No Dependencies)              │
└─────────────────────────────────┘
            ↑
┌─────────────────────────────────┐
│   Infrastructure Layer          │  ← EF Core, External Services
│  (Depends on Application +      │
│   Domain)                       │
└─────────────────────────────────┘
```

## Project Structure
```
Solution/
├── src/
│   ├── ProjectName.Domain/
│   │   ├── Entities/
│   │   │   ├── Customer.cs
│   │   │   └── Order.cs
│   │   ├── ValueObjects/
│   │   │   └── Address.cs
│   │   ├── Interfaces/
│   │   │   ├── ICustomerRepository.cs
│   │   │   └── IRepository.cs
│   │   └── Exceptions/
│   │       └── DomainException.cs
│   │
│   ├── ProjectName.Application/
│   │   ├── Commands/
│   │   │   └── CreateCustomerCommand.cs
│   │   ├── Queries/
│   │   │   └── GetCustomerByIdQuery.cs
│   │   ├── DTOs/
│   │   │   └── CustomerDto.cs
│   │   └── Validators/
│   │       └── CreateCustomerValidator.cs
│   │
│   ├── ProjectName.Infrastructure/
│   │   ├── Data/
│   │   │   ├── ApplicationDbContext.cs
│   │   │   └── Configurations/
│   │   │       └── CustomerConfiguration.cs
│   │   ├── Repositories/
│   │   │   ├── Repository.cs
│   │   │   └── CustomerRepository.cs
│   │   └── Migrations/
│   │
│   └── ProjectName.Api/
│       ├── Controllers/
│       │   └── CustomersController.cs
│       ├── Program.cs
│       └── appsettings.json
│
└── tests/
    ├── ProjectName.Domain.Tests/
    ├── ProjectName.Application.Tests/
    ├── ProjectName.Infrastructure.Tests/
    └── ProjectName.Api.Tests/
```

## Consequences

### Positive ✅
- **Testability**: Business logic testable without external dependencies
- **Maintainability**: Clear separation makes code easier to modify
- **Flexibility**: Easy to swap infrastructure implementations
- **Framework Independence**: Domain doesn't depend on EF Core, ASP.NET, etc.
- **Team Consistency**: Everyone follows the same pattern
- **Scalability**: New features fit into established structure

### Negative ❌
- **Initial Complexity**: More projects and files for simple apps
- **Learning Curve**: Team must understand architecture principles
- **Boilerplate**: More abstraction layers mean more code
- **Slower Initial Development**: Takes longer to set up than simple architecture

### Mitigation Strategies
- Provide project templates for quick setup
- Document common patterns in team memory bank
- Pair programming for onboarding
- For POCs, allow simplified 2-layer structure
- Create code generators for common use cases

## When to Deviate

### Allowed Simplifications
1. **Proof of Concepts**: Single project acceptable
2. **Microservices**: Collapse Application + Domain into "Core"
3. **Simple CRUD**: 2-layer (API + Infrastructure) with documentation
4. **Learning Projects**: Simplified structure for training

### Requirements for Deviation
- **Document**: Must explain why and for how long
- **Approve**: Lead architect approval required
- **Plan**: Migration path to full architecture
- **Time-box**: Specify when to refactor to standard

## Implementation Checklist

- [ ] Create 4 project structure (Domain, Application, Infrastructure, API)
- [ ] Define repository interfaces in Domain
- [ ] Implement entities with business logic in Domain
- [ ] Create commands/queries in Application
- [ ] Implement repositories in Infrastructure
- [ ] Configure EF Core in Infrastructure
- [ ] Set up dependency injection in API
- [ ] Add XML documentation to public APIs
- [ ] Write unit tests for Domain and Application
- [ ] Write integration tests for Infrastructure

## Related Decisions
- ADR-002: Repository Pattern (to be created)
- ADR-003: CQRS with MediatR (to be created)
- ADR-004: Result Pattern for Error Handling (to be created)

## References
- Clean Architecture by Robert C. Martin
- https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- Onion Architecture by Jeffrey Palermo
- https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/

## Review Date
**Next Review**: June 28, 2026 (6 months)
