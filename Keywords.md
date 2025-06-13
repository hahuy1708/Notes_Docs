# Common Keywords and Methods 

## 1. AsNoTracking()
- **Purpose**: Disable EF Core’s change tracking for entities returned by a query.
- **When to use**:
    - You only need to **read** data (no insert/update/delete).
    - You want better performance and lower memory usage when loading large result sets.
- **Key effect**:
    - The returned entities are **not** tracked in the `DbContext`’s change tracker.
    - Subsequent calls like `SaveChanges()` will ignore these entities (cannot update without re‐attaching).
### Example
```csharp
var product = await _context.Products
    .AsNoTracking()                       // Disable tracking
    .FirstOrDefaultAsync(p => p.Id == id);

if (product != null)
{
    // product is read-only; EF won’t watch for changes
    Console.WriteLine(product.Name);
}
``````
## 2. Include()
- **Purpose**: Eagerly load related navigation properties (e.g., parent/child tables) in a single query.
- **When to use:**:
  - You plan to read data from one entity and also need data from its related entity (e.g., load Category when querying Product).
  - You want to avoid separate (lazy) queries or the “N+1” problem.
- **Key effect**:
  - EF Core generates a SQL JOIN under the hood to fetch both tables together. 
### Example
```csharp
// Load all products in a category, including Category details:
var products = await _context.Products
    .AsNoTracking()
    .Include(p => p.Category)             // Eagerly load the Category navigation
    .Where(p => p.CategoryId == someCategoryId)
    .ToListAsync();

foreach (var p in products)
{
    // Now p.Category is already populated (no extra query needed)
    Console.WriteLine($"{p.Name} belongs to {p.Category.Name}");
}
``````

### 2.1. ThenInclude()
- **Purpose**: Loads related data from nested navigation properties after an Include().
- **When to use**: When you need to load a property of a related entity that has already been included via Include().
### Example
```csharp
var orders = await _context.Orders
    .Include(o => o.Customer)
    .ThenInclude(c => c.Membership)
    .ToListAsync();
// This loads: All orders - Each order’s customer - And each customer’s membership
``````


## 3. FindAsync()
- **Purpose**: Asynchronously retrieves an entity by its primary key.
- **When to use**:
    - You know the primary key value (e.g., id) and want to load that exact entity.
    - You plan to modify or delete this entity (tracking is needed).
- **Key behavior**:
    - Checks the DbContext’s local cache first (if it’s already tracked).
    - If not found in cache, queries the database.
    - Always returns a tracked entity (so you can update fields and save).

### Example
```csharp
var product = await _context.Products.FindAsync(id);

``````

## 4. FirstOrDefaultAsync()
- **Purpose**: Return the first entity in a sequence that matches a filter, or null if none match.
- **When to use**:
    - You need an entity based on any arbitrary condition, not just primary key. 
    - You want to retrieve it for read-only or update (depending on tracking).
- Tracking behavior:
    - By default, returns a tracked entity—unless you append .AsNoTracking(), which returns an untracked entity.
### Example (tracked)
```csharp
// Retrieve a single active product by name, tracking it for update:
var product = await _context.Products
    .FirstOrDefaultAsync(p => p.Name == "Widget" && p.IsAvailable);

if (product != null)
{
    product.StockQuantity -= 1;         // EF knows to UPDATE this entity
    await _context.SaveChangesAsync();
}
``````
### Example (untracked)
```csharp
// Retrieve read-only product details without tracking:
var product = await _context.Products
    .AsNoTracking()
    .FirstOrDefaultAsync(p => p.Name == "Gadget");

if (product != null)
{
    Console.WriteLine($"Price: {product.Price}");
    // Cannot update directly, since this entity is not tracked.
}
``````

## 5. ToListAsync() 
- **Purpose**: Execute a query and materialize the results as a List<T> asynchronously.
- **When to use**:
    - You expect multiple results and want them in a List<T>.
    - You are at the end of a LINQ query chain (after Where, Select, Include, etc.).
- **Key effect**:
    - Sends the final SQL to the database, retrieves all matching rows, and returns them as a List<T>.
### Example
```csharp
// Get all available products in category 2 as a list:
var productList = await _context.Products
    .AsNoTracking()
    .Where(p => p.CategoryId == 2 && p.IsAvailable)
    .ToListAsync();

// productList is now a List<Product> containing all matching rows
Console.WriteLine($"Found {productList.Count} products.");
``````

## 6. Select()
- **Purpose**: Project or transform each element of a query into a new form (often a DTO).
- **When to use**:
    - You want to shape the result, for example mapping an entity to a lightweight DTO.
    - You only need a subset of fields, rather than loading entire entities.
- **Key effect**:
    - Translates into a SQL SELECT that returns only the specified columns.
    - Improves performance by reducing data transferred.
### Example
```csharp
var productDtos = await _context.Products
    .AsNoTracking()
    .Where(p => p.IsAvailable)
    .Select(p => new ProductResponseDTO
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price,
        StockQuantity = p.StockQuantity
    })
    .ToListAsync();

// productDtos is List<ProductResponseDTO>, containing only the selected fields
``````

## 7. Where()
- **Purpose**: Filter a query based on a boolean predicate.
- **When to use**:
    - You only want entities that meet certain conditions (e.g., IsAvailable == true, Price > 50).
    - Place it before ToListAsync(), FirstOrDefaultAsync(), etc., to limit results.
### Example
```csharp
// Get all active categories whose names start with “Elec”
var categories = await _context.Categories
    .AsNoTracking()
    .Where(c => c.IsActive && c.Name.StartsWith("Elec"))
    .ToListAsync();

foreach (var c in categories)
    Console.WriteLine(c.Name);

``````
    
## 8. AnyAsync()
- **Purpose**: Check if any element in the database satisfies a given predicate.
- **When to use**:
    - You need a simple true/false existence check (e.g., “Does this email already exist?”).
    - You do not need to load the entire entity—only check existence.
- **Key effect**:
    - Translates into SELECT CASE WHEN EXISTS(...) THEN 1 ELSE 0 END SQL, which is very fast.
    - Returns a bool (true if at least one match is found, otherwise false).
### Example
```csharp
// Check if a product with this SKU exists:
bool exists = await _context.Products
    .AnyAsync(p => p.SKU == inputSku);

if (exists)
{
    Console.WriteLine("SKU already in use.");
}
``````



## Putting It All Together – Sample Flow
```csharp
public async Task<bool> IsProductNameTakenAsync(string name)
{
    // 1. Check existence quickly (no entity load):
    return await _context.Products
        .AsNoTracking()
        .AnyAsync(p => p.Name.ToLower() == name.ToLower());
}

public async Task<ProductResponseDTO> GetProductDetailsAsync(int id)
{
    // 2. Load product by ID, include Category, read-only:
    var p = await _context.Products
        .AsNoTracking()
        .Include(p => p.Category)
        .FirstOrDefaultAsync(p => p.Id == id);

    if (p == null) return null;

    // 3. Project to DTO (only fields needed):
    return new ProductResponseDTO
    {
        Id               = p.Id,
        Name             = p.Name,
        Price            = p.Price,
        StockQuantity    = p.StockQuantity,
        CategoryName     = p.Category.Name
    };
}

public async Task<List<ProductResponseDTO>> ListAvailableProductsInCategoryAsync(int categoryId)
{
    // 4. Filter, include, project to DTO, return list:
    return await _context.Products
        .AsNoTracking()
        .Include(p => p.Category)
        .Where(p => p.CategoryId == categoryId && p.IsAvailable)
        .Select(p => new ProductResponseDTO
        {
            Id            = p.Id,
            Name          = p.Name,
            Price         = p.Price,
            StockQuantity = p.StockQuantity,
            CategoryName  = p.Category.Name
        })
        .ToListAsync();
}

``````