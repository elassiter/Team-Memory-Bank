# EF Core with SQL Server - Team Guidelines

## Connection Configuration

### Connection Strings (appsettings.json)
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyAppDb;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=true",
    "Production": "Server=prod-server;Database=MyAppDb;User Id=appuser;Password=***;Encrypt=True;TrustServerCertificate=False"
  }
}
```

### DbContext Registration (Program.cs)
```csharp
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30);
            sqlOptions.MigrationsAssembly("Infrastructure");
        }
    ));
```

## DbContext Pattern

### Standard DbContext
```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    // DbSets
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Apply all entity configurations from assembly
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
        
        // Global query filters (soft delete example)
        modelBuilder.Entity<Customer>().HasQueryFilter(c => !c.IsDeleted);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Automatically set audit fields
        var entries = ChangeTracker.Entries()
            .Where(e => e.Entity is BaseEntity && 
                       (e.State == EntityState.Added || e.State == EntityState.Modified));

        foreach (var entry in entries)
        {
            var entity = (BaseEntity)entry.Entity;
            
            if (entry.State == EntityState.Added)
            {
                entity.CreatedAt = DateTime.UtcNow;
            }
            else
            {
                entity.UpdatedAt = DateTime.UtcNow;
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

## Entity Configuration

### Use IEntityTypeConfiguration
```csharp
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        // Table mapping
        builder.ToTable("Customers");
        
        // Primary key
        builder.HasKey(c => c.Id);
        
        // Properties
        builder.Property(c => c.FirstName)
            .IsRequired()
            .HasMaxLength(100)
            .IsUnicode(true);
            
        builder.Property(c => c.Email)
            .IsRequired()
            .HasMaxLength(255)
            .HasColumnType("varchar(255)"); // Non-unicode for performance
            
        // Decimal precision for money
        builder.Property(c => c.CreditLimit)
            .HasPrecision(18, 2);
            
        // Computed column
        builder.Property(c => c.FullName)
            .HasComputedColumnSql("[FirstName] + ' ' + [LastName]", stored: true);
            
        // Indexes
        builder.HasIndex(c => c.Email)
            .IsUnique()
            .HasDatabaseName("IX_Customers_Email");
            
        builder.HasIndex(c => new { c.LastName, c.FirstName })
            .HasDatabaseName("IX_Customers_Name");
            
        // Relationships
        builder.HasMany(c => c.Orders)
            .WithOne(o => o.Customer)
            .HasForeignKey(o => o.CustomerId)
            .OnDelete(DeleteBehavior.Cascade)
            .HasConstraintName("FK_Orders_Customers");
            
        // Value objects (owned types)
        builder.OwnsOne(c => c.Address, address =>
        {
            address.Property(a => a.Street).HasMaxLength(200);
            address.Property(a => a.City).HasMaxLength(100);
            address.Property(a => a.ZipCode).HasMaxLength(10);
        });
    }
}
```

### Many-to-Many Configuration
```csharp
public class OrderProductConfiguration : IEntityTypeConfiguration<OrderProduct>
{
    public void Configure(EntityTypeBuilder<OrderProduct> builder)
    {
        builder.ToTable("OrderProducts");
        
        // Composite key
        builder.HasKey(op => new { op.OrderId, op.ProductId });
        
        builder.HasOne(op => op.Order)
            .WithMany(o => o.OrderProducts)
            .HasForeignKey(op => op.OrderId);
            
        builder.HasOne(op => op.Product)
            .WithMany(p => p.OrderProducts)
            .HasForeignKey(op => op.ProductId);
            
        builder.Property(op => op.Quantity)
            .IsRequired();
            
        builder.Property(op => op.UnitPrice)
            .HasPrecision(18, 2);
    }
}
```

## Migrations

### CLI Commands
```bash
# Add new migration
dotnet ef migrations add InitialCreate --project Infrastructure --startup-project API

# Update database
dotnet ef database update --project Infrastructure --startup-project API

# Rollback to specific migration
dotnet ef database update PreviousMigration --project Infrastructure --startup-project API

# Remove last migration (if not applied)
dotnet ef migrations remove --project Infrastructure --startup-project API

# Generate SQL script
dotnet ef migrations script --project Infrastructure --output migration.sql

# Generate script for specific range
dotnet ef migrations script AddCustomers AddOrders --output updates.sql
```

### Migration Best Practices
- One migration per logical database change
- Use descriptive names (AddCustomerTable, AddEmailIndexToCustomers)
- Review generated migration before applying
- Never edit applied migrations
- Test migrations on dev database first
- Keep migrations in source control

## Query Patterns

### Efficient Queries
```csharp
// ✅ Use AsNoTracking for read-only queries
public async Task<IEnumerable<CustomerDto>> GetAllCustomersAsync()
{
    return await _context.Customers
        .AsNoTracking()
        .Select(c => new CustomerDto
        {
            Id = c.Id,
            FullName = c.FirstName + " " + c.LastName,
            Email = c.Email
        })
        .ToListAsync();
}

// ✅ Projection to DTO
public async Task<CustomerDto?> GetCustomerByIdAsync(int id)
{
    return await _context.Customers
        .Where(c => c.Id == id)
        .Select(c => new CustomerDto
        {
            Id = c.Id,
            FullName = c.FullName,
            Email = c.Email
        })
        .FirstOrDefaultAsync();
}

// ✅ Explicit loading with Include
public async Task<Order?> GetOrderWithDetailsAsync(int orderId)
{
    return await _context.Orders
        .Include(o => o.Customer)
        .Include(o => o.OrderProducts)
            .ThenInclude(op => op.Product)
        .FirstOrDefaultAsync(o => o.Id == orderId);
}

// ✅ Pagination
public async Task<IEnumerable<Customer>> GetCustomersPagedAsync(int page, int pageSize)
{
    return await _context.Customers
        .AsNoTracking()
        .OrderBy(c => c.LastName)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
}
```

### Avoid N+1 Problems
```csharp
// ❌ Bad - N+1 query problem
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
{
    // Separate query for each order!
    order.Customer = await _context.Customers.FindAsync(order.CustomerId);
}

// ✅ Good - Single query with Include
var orders = await _context.Orders
    .Include(o => o.Customer)
    .ToListAsync();
```

### Raw SQL (When Needed)
```csharp
// Complex queries or stored procedures
public async Task<IEnumerable<CustomerStats>> GetCustomerStatsAsync()
{
    return await _context.Database
        .SqlQueryRaw<CustomerStats>(@"
            SELECT 
                c.Id,
                c.FirstName + ' ' + c.LastName AS FullName,
                COUNT(o.Id) AS OrderCount,
                SUM(o.TotalAmount) AS TotalSpent
            FROM Customers c
            LEFT JOIN Orders o ON c.Id = o.CustomerId
            GROUP BY c.Id, c.FirstName, c.LastName")
        .ToListAsync();
}

// Parameterized queries
public async Task<Customer?> GetByEmailAsync(string email)
{
    return await _context.Customers
        .FromSqlInterpolated($"SELECT * FROM Customers WHERE Email = {email}")
        .FirstOrDefaultAsync();
}
```

## Transaction Management

### Transaction Pattern
```csharp
public async Task<Result> TransferOrderAsync(int orderId, int newCustomerId)
{
    using var transaction = await _context.Database.BeginTransactionAsync();
    
    try
    {
        var order = await _context.Orders.FindAsync(orderId);
        if (order == null)
            return Result.Failure("Order not found");
            
        var customer = await _context.Customers.FindAsync(newCustomerId);
        if (customer == null)
            return Result.Failure("Customer not found");
            
        order.CustomerId = newCustomerId;
        await _context.SaveChangesAsync();
        
        // Additional operations...
        
        await transaction.CommitAsync();
        return Result.Success();
    }
    catch (Exception ex)
    {
        await transaction.RollbackAsync();
        return Result.Failure(ex.Message);
    }
}
```

## Performance Optimization

### Key Strategies
1. **Use AsNoTracking** for read-only queries
2. **Project to DTOs** instead of loading full entities
3. **Include only what you need** - avoid over-fetching
4. **Implement pagination** for large result sets
5. **Add indexes** on foreign keys and frequently queried columns
6. **Use compiled queries** for hot paths
7. **Batch operations** when possible

### Compiled Queries
```csharp
private static readonly Func<ApplicationDbContext, int, Task<Customer?>> _getCustomerById =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
        context.Customers.FirstOrDefault(c => c.Id == id));

public async Task<Customer?> GetCustomerByIdAsync(int id)
{
    return await _getCustomerById(_context, id);
}
```

### Bulk Operations (EF Core 7+)
```csharp
// Bulk update
await _context.Customers
    .Where(c => c.LastOrderDate < cutoffDate)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(c => c.IsActive, false)
        .SetProperty(c => c.UpdatedAt, DateTime.UtcNow));

// Bulk delete
await _context.Customers
    .Where(c => !c.IsActive && c.IsDeleted)
    .ExecuteDeleteAsync();
```

## Common Pitfalls

### ❌ DON'T
- Call `SaveChanges()` in loops
- Load entire tables without filtering
- Use `.Include()` unnecessarily
- Forget to use async methods
- Hardcode connection strings
- Edit applied migrations

### ✅ DO
- Use async/await consistently
- Filter before loading (`.Where()` then `.ToList()`)
- Use projections for DTOs
- Batch operations when possible
- Use migrations for schema changes
- Enable connection pooling (default)
- Log SQL in development

## Index Strategy for SQL Server

```csharp
// Foreign keys (always)
builder.HasIndex(o => o.CustomerId);

// Unique constraints
builder.HasIndex(c => c.Email).IsUnique();

// Composite indexes for common queries
builder.HasIndex(o => new { o.CustomerId, o.OrderDate });

// Filtered indexes (EF Core)
builder.HasIndex(c => c.Email)
    .HasFilter("[IsActive] = 1");
```

## Testing with EF Core

### In-Memory Database
```csharp
var options = new DbContextOptionsBuilder<ApplicationDbContext>()
    .UseInMemoryDatabase(databaseName: "TestDb")
    .Options;

using var context = new ApplicationDbContext(options);
var repository = new CustomerRepository(context);

// Seed test data
context.Customers.Add(new Customer { FirstName = "Test", LastName = "User" });
await context.SaveChangesAsync();

// Test repository methods
var customer = await repository.GetByIdAsync(1);
Assert.NotNull(customer);
```
