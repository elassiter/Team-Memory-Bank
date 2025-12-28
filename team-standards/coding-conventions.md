# Team Coding Standards

## SOLID Principles

### Single Responsibility Principle (SRP)
- Each class should have one reason to change
- Keep classes focused on a single responsibility
```csharp
// ❌ Bad - Multiple responsibilities
public class CustomerService
{
    public void SaveCustomer(Customer customer) { }
    public void SendEmail(string email, string message) { }
    public void GenerateReport() { }
}

// ✅ Good - Separated responsibilities
public class CustomerRepository { }
public class EmailService { }
public class ReportGenerator { }
```

### Open/Closed Principle (OCP)
- Open for extension, closed for modification
```csharp
// ✅ Good - Use interfaces and inheritance
public interface IPaymentProcessor
{
    void ProcessPayment(decimal amount);
}

public class CreditCardProcessor : IPaymentProcessor { }
public class PayPalProcessor : IPaymentProcessor { }
```

### Liskov Substitution Principle (LSP)
- Derived classes must be substitutable for base classes
```csharp
// ✅ Good - Derived class honors base class contract
public abstract class Shape
{
    public abstract double CalculateArea();
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    
    public override double CalculateArea() => Width * Height;
}
```

### Interface Segregation Principle (ISP)
- Clients shouldn't depend on interfaces they don't use
```csharp
// ❌ Bad - Fat interface
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
}

// ✅ Good - Segregated interfaces
public interface IWorkable { void Work(); }
public interface IFeedable { void Eat(); }
public interface IRestable { void Sleep(); }
```

### Dependency Inversion Principle (DIP)
- Depend on abstractions, not concretions
```csharp
// ✅ Good - Depend on interface
public class OrderService
{
    private readonly IOrderRepository _repository;
    
    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }
}
```

## Naming Conventions

### Classes and Interfaces
```csharp
// Classes: PascalCase
public class CustomerService { }
public class OrderRepository { }

// Interfaces: I + PascalCase
public interface ICustomerService { }
public interface IOrderRepository { }

// Abstract classes: PascalCase
public abstract class BaseEntity { }
```

### Methods and Properties
```csharp
// Methods: PascalCase, verb-based
public void SaveCustomer() { }
public async Task<Customer> GetCustomerAsync(int id) { }

// Properties: PascalCase, noun-based
public string FirstName { get; set; }
public int TotalCount { get; private set; }
```

### Variables and Parameters
```csharp
// Local variables: camelCase
var customerName = "John";
int orderCount = 5;

// Parameters: camelCase
public void ProcessOrder(int orderId, string customerEmail) { }

// Private fields: _camelCase
private readonly ILogger _logger;
private string _connectionString;

// Constants: UPPER_SNAKE_CASE or PascalCase
private const int MAX_RETRY_COUNT = 3;
public const string DefaultCurrency = "USD";
```

### Async Methods
```csharp
// Always suffix with 'Async'
public async Task<Customer> GetCustomerAsync(int id) { }
public async Task SaveChangesAsync() { }
```

## Code Organization

### File Structure
- One class per file
- File name matches class name
- Group related classes in folders
```
Domain/
├── Entities/
│   ├── Customer.cs
│   └── Order.cs
├── ValueObjects/
│   ├── Address.cs
│   └── Money.cs
└── Interfaces/
    ├── ICustomerRepository.cs
    └── IOrderRepository.cs
```

### Using Statements
```csharp
// System namespaces first, then external, then internal
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.EntityFrameworkCore;
using MyCompany.Domain.Entities;
```

### Class Member Order
```csharp
public class Customer
{
    // 1. Constants
    private const int MaxNameLength = 100;
    
    // 2. Static fields
    private static int _instanceCount;
    
    // 3. Fields
    private readonly string _email;
    private int _age;
    
    // 4. Constructors
    public Customer(string email)
    {
        _email = email;
    }
    
    // 5. Properties
    public string FirstName { get; set; }
    public string LastName { get; set; }
    
    // 6. Methods (public first, then private)
    public void UpdateEmail(string newEmail) { }
    private void ValidateEmail(string email) { }
}
```

## Error Handling

### Use Specific Exceptions
```csharp
// ❌ Bad
throw new Exception("Something went wrong");

// ✅ Good
throw new ArgumentNullException(nameof(customer));
throw new InvalidOperationException("Customer is not active");
```

### Custom Exceptions
```csharp
public class CustomerNotFoundException : Exception
{
    public int CustomerId { get; }
    
    public CustomerNotFoundException(int customerId)
        : base($"Customer with ID {customerId} was not found")
    {
        CustomerId = customerId;
    }
}
```

### Try-Catch Guidelines
```csharp
// Catch specific exceptions
try
{
    await _repository.SaveAsync(customer);
}
catch (DbUpdateException ex)
{
    _logger.LogError(ex, "Database error saving customer");
    throw;
}
catch (ValidationException ex)
{
    _logger.LogWarning(ex, "Validation failed");
    return Result.Failure(ex.Errors);
}
```

## Comments and Documentation

### XML Documentation
```csharp
/// <summary>
/// Retrieves a customer by their unique identifier.
/// </summary>
/// <param name="id">The customer's unique identifier.</param>
/// <returns>The customer if found; otherwise, null.</returns>
/// <exception cref="ArgumentException">Thrown when id is less than 1.</exception>
public async Task<Customer?> GetCustomerByIdAsync(int id)
{
    if (id < 1)
        throw new ArgumentException("ID must be greater than 0", nameof(id));
        
    return await _repository.GetByIdAsync(id);
}
```

### Inline Comments
```csharp
// Use comments to explain "why", not "what"

// ❌ Bad - Obvious what the code does
// Increment counter by 1
counter++;

// ✅ Good - Explains why
// Retry connection due to transient network issues
for (int i = 0; i < MAX_RETRY_COUNT; i++)
{
    try
    {
        await ConnectAsync();
        break;
    }
    catch (NetworkException)
    {
        if (i == MAX_RETRY_COUNT - 1) throw;
        await Task.Delay(1000);
    }
}
```

## Formatting

### Braces
```csharp
// Always use braces, even for single-line statements
if (customer == null)
{
    return NotFound();
}

// Exception: Short property getters/setters
public string Name { get; set; }
```

### Line Length
- Max 120 characters per line
- Break long method chains

```csharp
// ✅ Good - Readable method chain
var result = await _context.Customers
    .Where(c => c.IsActive)
    .OrderBy(c => c.LastName)
    .ThenBy(c => c.FirstName)
    .ToListAsync();
```

### Spacing
```csharp
// Space after control keywords
if (condition) { }
for (int i = 0; i < count; i++) { }

// No space for method calls
MethodCall();
DoSomething(parameter);

// Space around operators
int sum = a + b;
bool isValid = value == 10;
```

## Best Practices

### Use var Appropriately
```csharp
// ✅ Good - Type is obvious
var customer = new Customer();
var count = GetCount();

// ❌ Bad - Type is not obvious
var result = ProcessData();  // What type is result?

// ✅ Better - Explicit type when unclear
CustomerDto result = ProcessData();
```

### String Handling
```csharp
// Use string interpolation
string message = $"Hello, {name}!";

// Use StringBuilder for concatenation in loops
var sb = new StringBuilder();
foreach (var item in items)
{
    sb.AppendLine(item.ToString());
}
```

### Null Checking
```csharp
// Use null-conditional operators
var length = customer?.Name?.Length ?? 0;

// Use null-coalescing
string name = customer?.Name ?? "Unknown";

// C# 11+ required parameters
public void ProcessOrder(required string orderId) { }
```

### Async/Await
```csharp
// Always await async calls
await SaveCustomerAsync(customer);

// Don't mix sync and async
// ❌ Bad
Task.Run(() => SaveCustomer(customer)).Wait();

// ✅ Good
await Task.Run(() => SaveCustomer(customer));

// Use ConfigureAwait(false) in library code
await SaveAsync().ConfigureAwait(false);
```

### LINQ Usage
```csharp
// Use LINQ for clarity
var activeCustomers = customers.Where(c => c.IsActive).ToList();

// Avoid complex LINQ - break into steps
// ❌ Bad - Hard to read
var result = orders
    .Where(o => o.Status == "Active")
    .SelectMany(o => o.Items)
    .Where(i => i.Price > 100)
    .GroupBy(i => i.Category)
    .Select(g => new { Category = g.Key, Total = g.Sum(i => i.Price) })
    .ToList();

// ✅ Good - Broken into readable steps
var activeOrders = orders.Where(o => o.Status == "Active");
var expensiveItems = activeOrders
    .SelectMany(o => o.Items)
    .Where(i => i.Price > 100);
var categorySummary = expensiveItems
    .GroupBy(i => i.Category)
    .Select(g => new CategoryTotal(g.Key, g.Sum(i => i.Price)))
    .ToList();
```

## Testing

### Test Naming
```csharp
// Pattern: MethodName_Scenario_ExpectedBehavior
[Fact]
public void GetCustomerById_ValidId_ReturnsCustomer() { }

[Fact]
public void GetCustomerById_InvalidId_ThrowsException() { }

[Fact]
public void SaveCustomer_NullCustomer_ThrowsArgumentNullException() { }
```

### AAA Pattern
```csharp
[Fact]
public async Task CreateCustomer_ValidData_ReturnsSuccess()
{
    // Arrange
    var command = new CreateCustomerCommand("John", "Doe", "john@example.com");
    var handler = new CreateCustomerCommandHandler(_mockRepository.Object);
    
    // Act
    var result = await handler.Handle(command, CancellationToken.None);
    
    // Assert
    Assert.True(result.IsSuccess);
    Assert.True(result.Value > 0);
}
```
