# .NET System Patterns - Team Standards

## Architecture: Clean Architecture (Onion)

### Layer Structure
```
Domain Layer (Core)
├── Entities
├── Value Objects
├── Domain Events
├── Interfaces (abstracts)
└── Exceptions

Application Layer
├── Use Cases / Commands / Queries
├── DTOs
├── Application Interfaces
├── Mapping Profiles
└── Validators

Infrastructure Layer
├── Persistence (EF Core)
├── Repositories (implementations)
├── External Services
├── File System
└── Email/SMS Services

Presentation Layer (API/UI)
├── Controllers
├── ViewModels
├── Middleware
└── Program.cs / Startup
```

### Dependency Rules
- **Domain**: No dependencies on other layers
- **Application**: Depends only on Domain
- **Infrastructure**: Depends on Application & Domain
- **Presentation**: Depends on Application & Infrastructure

## Repository Pattern

### Interface Definition (Domain Layer)
```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
    Task<bool> ExistsAsync(int id);
}

// Specific repositories
public interface ICustomerRepository : IRepository<Customer>
{
    Task<Customer?> GetByEmailAsync(string email);
    Task<IEnumerable<Customer>> GetActiveCustomersAsync();
}
```

### Implementation (Infrastructure Layer)
```csharp
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id)
    {
        return await _dbSet.FindAsync(id);
    }

    public async Task<IEnumerable<T>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }

    public async Task<T> AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
        await _context.SaveChangesAsync();
        return entity;
    }

    public async Task UpdateAsync(T entity)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var entity = await GetByIdAsync(id);
        if (entity != null)
        {
            _dbSet.Remove(entity);
            await _context.SaveChangesAsync();
        }
    }

    public async Task<bool> ExistsAsync(int id)
    {
        return await _dbSet.FindAsync(id) != null;
    }
}
```

### Specific Repository Implementation
```csharp
public class CustomerRepository : Repository<Customer>, ICustomerRepository
{
    public CustomerRepository(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<Customer?> GetByEmailAsync(string email)
    {
        return await _dbSet
            .FirstOrDefaultAsync(c => c.Email == email);
    }

    public async Task<IEnumerable<Customer>> GetActiveCustomersAsync()
    {
        return await _dbSet
            .Where(c => c.IsActive)
            .ToListAsync();
    }
}
```

## EF Core Patterns

### DbContext Setup
```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Apply all configurations from assembly
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}
```

### Entity Configuration (Fluent API)
```csharp
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("Customers");
        
        builder.HasKey(c => c.Id);
        
        builder.Property(c => c.FirstName)
            .IsRequired()
            .HasMaxLength(100);
            
        builder.Property(c => c.Email)
            .IsRequired()
            .HasMaxLength(255);
            
        builder.HasIndex(c => c.Email)
            .IsUnique();
            
        // Relationships
        builder.HasMany(c => c.Orders)
            .WithOne(o => o.Customer)
            .HasForeignKey(o => o.CustomerId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### Connection String (appsettings.json)
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyDatabase;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=true"
  }
}
```

### Program.cs Registration
```csharp
// Add DbContext
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        b => b.MigrationsAssembly("Infrastructure")));

// Add repositories
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();
```

## Domain Entities

### Base Entity
```csharp
public abstract class BaseEntity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public string? CreatedBy { get; set; }
    public string? UpdatedBy { get; set; }
}
```

### Domain Entity Example
```csharp
public class Customer : BaseEntity
{
    public string FirstName { get; private set; } = string.Empty;
    public string LastName { get; private set; } = string.Empty;
    public string Email { get; private set; } = string.Empty;
    public bool IsActive { get; private set; }
    
    // Navigation properties
    public virtual ICollection<Order> Orders { get; private set; } = new List<Order>();
    
    // Private constructor for EF Core
    private Customer() { }
    
    // Factory method
    public static Customer Create(string firstName, string lastName, string email)
    {
        // Validation
        if (string.IsNullOrWhiteSpace(firstName))
            throw new ArgumentException("First name is required", nameof(firstName));
            
        return new Customer
        {
            FirstName = firstName,
            LastName = lastName,
            Email = email,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };
    }
    
    // Business methods
    public void Activate() => IsActive = true;
    public void Deactivate() => IsActive = false;
    
    public void UpdateEmail(string newEmail)
    {
        if (string.IsNullOrWhiteSpace(newEmail))
            throw new ArgumentException("Email is required", nameof(newEmail));
            
        Email = newEmail;
        UpdatedAt = DateTime.UtcNow;
    }
}
```

## CQRS Pattern

### Command Example
```csharp
// Command
public record CreateCustomerCommand(
    string FirstName,
    string LastName,
    string Email) : IRequest<Result<int>>;

// Handler
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
        try
        {
            var customer = Customer.Create(
                request.FirstName,
                request.LastName,
                request.Email);
                
            var created = await _repository.AddAsync(customer);
            
            return Result<int>.Success(created.Id);
        }
        catch (Exception ex)
        {
            return Result<int>.Failure(ex.Message);
        }
    }
}
```

### Query Example
```csharp
// Query
public record GetCustomerByIdQuery(int Id) : IRequest<Result<CustomerDto>>;

// Handler
public class GetCustomerByIdQueryHandler 
    : IRequestHandler<GetCustomerByIdQuery, Result<CustomerDto>>
{
    private readonly ICustomerRepository _repository;
    private readonly IMapper _mapper;

    public GetCustomerByIdQueryHandler(
        ICustomerRepository repository,
        IMapper mapper)
    {
        _repository = repository;
        _mapper = mapper;
    }

    public async Task<Result<CustomerDto>> Handle(
        GetCustomerByIdQuery request,
        CancellationToken cancellationToken)
    {
        var customer = await _repository.GetByIdAsync(request.Id);
        
        if (customer == null)
            return Result<CustomerDto>.Failure("Customer not found");
            
        var dto = _mapper.Map<CustomerDto>(customer);
        return Result<CustomerDto>.Success(dto);
    }
}
```

## Result Pattern

### Result<T> Implementation
```csharp
public class Result<T>
{
    public bool IsSuccess { get; private set; }
    public T? Value { get; private set; }
    public string? Error { get; private set; }
    public List<string> Errors { get; private set; } = new();

    private Result(bool isSuccess, T? value, string? error, List<string>? errors = null)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
        Errors = errors ?? new List<string>();
    }

    public static Result<T> Success(T value) => 
        new(true, value, null);
        
    public static Result<T> Failure(string error) => 
        new(false, default, error, new List<string> { error });
        
    public static Result<T> Failure(List<string> errors) => 
        new(false, default, null, errors);
}
```

## Dependency Injection

### Service Registration Pattern
```csharp
// Program.cs or extension method
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // DbContext
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection")));
        
        // Repositories
        services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
        services.AddScoped<ICustomerRepository, CustomerRepository>();
        
        return services;
    }
    
    public static IServiceCollection AddApplication(
        this IServiceCollection services)
    {
        // MediatR for CQRS
        services.AddMediatR(cfg => 
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
        
        // AutoMapper
        services.AddAutoMapper(Assembly.GetExecutingAssembly());
        
        return services;
    }
}
```
