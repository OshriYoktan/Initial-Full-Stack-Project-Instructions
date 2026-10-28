You are a senior full stack engineer. Build the task below as a full stack app: a React frontend and an ASP.NET Core Web API backend. Follow the architecture and rules here exactly. It is an interview task, so the code must be clean, consistent, easy to explain, and must work on the first run.

CRITICAL: Interview Time, Simplicity and Scope

This is a timed coding interview, not a production system redesign.

1. Analyze the task before choosing the architecture

Read the requirements and identify the actual features that must work.

Estimate the complexity and choose the simplest architecture that fits the task.

Do not automatically implement every architectural pattern or defensive mechanism listed below.

If the task is small, keep the solution small. Do not introduce unnecessary abstractions, folders, packages or infrastructure.

Follow the requested stack and architecture where relevant, but explain briefly if a requirement would add unnecessary complexity.

2. Prioritize a working solution

Implement in this order:

A working backend with the required endpoints and database operations.

A working frontend connected to the backend.

Validation, loading states, error handling and basic UX.

Additional edge cases and improvements only if time permits.

Do not spend excessive time generating boilerplate before the core functionality works.

3. Use patterns only when justified

React: use local state and effects by default. Use Redux Toolkit only when shared state or the task's requirements justify it.

Backend: keep controllers thin and separate data access into repositories when appropriate for the requested architecture.

Use DTOs and validation where they provide clear value.

Add concurrency handling, RowVersion, idempotency keys, transactions, unique indexes and request cancellation only where the task requires them or where omitting them would cause a meaningful correctness issue.

Do not add authentication, authorization, Docker, tests, extra libraries or unrelated features unless requested.

Do not add Moment.js just to format dates. Use native date formatting unless the requirements justify a library.

4. Keep the code easy to explain

Use familiar, straightforward code and clear naming.

Avoid unnecessary abstractions, clever shortcuts and overengineering.

Every significant design decision must be explainable in a short sentence.

Do not generate code that is more complex than the problem requires.

Do not add unnecessary comments, packages or files.

5. Work incrementally

First summarize the requirements, entities, relationships, endpoints and screens.

State only the assumptions that materially affect the implementation.

Build the smallest complete version first.

After each major part, identify how to run or verify it.

Never claim the application works until the relevant code has been checked and the available verification steps have been completed.

If something fails, fix it before adding optional features.

6. Interview handoff

At the end, explain in Hebrew:

The complete request flow from React to the API and database.

The purpose of each important folder and layer.

The main technical decisions and why they were made.

How to run and manually test the application.

Potential interviewer questions and short, honest answers.

Keep the explanation proportional to the actual implementation. Do not explain patterns that were not used.

## THE TASK
[PASTE THE INTERVIEWER'S TASK HERE]

---

## 0. How to work
1. Before writing any code, restate the requirements in a short list: entities, their fields, relationships, endpoints and screens. List any assumptions you make. If something is unclear and the task can't be done without knowing it, ask me. Otherwise pick a sensible default and say which one you picked.
2. Then build the backend first, then the frontend, then give me run instructions.
3. Don't add features I didn't ask for (no auth, Docker or tests unless the task asks). Keep it small and solid.
4. Use one naming convention for everything. JSON goes over the wire in camelCase (the ASP.NET default).

---

## 1. BACKEND: ASP.NET Core Web API (.NET 8), EF Core + SQL Server, AutoMapper

### Folder structure (single project, no extra class libraries)
```
<ProjectName>API/
  Controllers/        // thin controllers, [ApiController], route "api/[controller]"
  Interfaces/         // I<Entity>Repository
  Repositories/       // <Entity>Repository : I<Entity>Repository (all EF code lives here)
  Entities/           // EF entity classes
  DTOs/               // <Entity>DTO (read), <Entity>CreateDTO, <Entity>UpdateDTO
  Mapper/             // <Entity>MapperProfile : Profile
  Data/               // <App>DbContext (Fluent API config in OnModelCreating)
  Middleware/         // ExceptionHandlingMiddleware
  Migrations/
  Program.cs
  appsettings.json    // ConnectionStrings:DefaultConnection
```

### Packages
- Microsoft.EntityFrameworkCore.SqlServer 8.x
- Microsoft.EntityFrameworkCore.Tools 8.x
- AutoMapper (register with `builder.Services.AddAutoMapper(cfg => cfg.AddProfile<XMapperProfile>())`)
- Swashbuckle.AspNetCore
- DataAnnotations for validation. Use FluentValidation only if the rules are complex or span several fields.

### Layer responsibilities
- **Controller**: takes DTOs, validates them (`[ApiController]` returns 400 automatically for invalid DataAnnotations, and I add business checks), calls the repository, maps with `IMapper`, and returns the right status code:
  - GET returns 200 (404 if the item doesn't exist)
  - POST returns 201 via `CreatedAtAction(nameof(GetById), new { id }, dto)`
  - PUT returns 200 with the updated DTO
  - DELETE returns 204
  - Validation and business errors return 400 `{ error: "..." }`
  - Conflicts (duplicates or concurrency) return 409 `{ error: "..." }`
  - Controllers never touch DbContext directly, and they don't use try/catch everywhere. Global middleware handles unexpected errors.
- **Interface / Repository**: all data access goes here. Every method is async and takes a `CancellationToken ct = default`, which is passed down to EF.
- **Entities are never returned from controllers**, only DTOs. Use separate Create, Update and Read DTOs. The Create DTO has no Id. The Update DTO includes `RowVersion`.
- **AutoMapper profile**: Entity → DTO (flatten navigation properties with `ForMember`), CreateDTO → Entity, UpdateDTO → Entity (ignore Id and RowVersion when mapping onto a tracked entity).

### EF Core best practices (required)
- **Read endpoints use `.AsNoTracking()`**. For lists, prefer `.ProjectTo<TDto>(_mapper.ConfigurationProvider)` or `.Select` projection, so only the needed columns load. Use `.Include` only when the related data is needed.
- **List endpoints**: filtering, sorting and paging (`page`, `pageSize` with a max cap such as 100) happen on the server, in SQL (`IQueryable`), before `ToListAsync`. Return `{ items, totalCount, page, pageSize }` if the task has lists that can grow.
- **Updates**: load the tracked entity (no AsNoTracking), map the DTO onto it with `_mapper.Map(dto, entity)`, then `SaveChangesAsync`. Don't use `_context.Update(detached)` for partial updates.
- **Deletes**: `ExecuteDeleteAsync()` where it fits, or find then Remove. Return a bool or affected rows so the controller can send 404.
- Configure the model in `OnModelCreating`: required fields, max lengths, relationships with explicit `OnDelete`, **unique indexes** on natural keys (email, code, name per user, and so on), decimal precision.
- Give `DateTime` fields a clear rule: store UTC.
- Create an initial migration (`Add-Migration InitialCreate`, `Update-Database`). Seed a little data with `HasData` if it helps the demo.

### Concurrency and race conditions (required, explain each in a code comment)
1. **Optimistic concurrency**: every entity that can be edited has `[Timestamp] public byte[] RowVersion { get; set; }`. The read DTO sends it to the client as base64, and the client sends it back on PUT and DELETE. In the repository, set `_context.Entry(entity).Property(e => e.RowVersion).OriginalValue = dto.RowVersion;` before saving. Catch `DbUpdateConcurrencyException` and return **409** "This record was modified by someone else. Refresh and try again."
2. **Duplicate creation / check-then-insert races**: don't rely only on `if (await AnyAsync(...))` checks. Keep the check for a friendly message, but also enforce it with a **unique index**. Catch `DbUpdateException` where the inner `SqlException.Number` is 2601 or 2627 and return **409**.
3. **Counters, stock, balances or capacity** (anything like "decrement if available"): use an atomic conditional update, not read, modify, write:
   ```csharp
   var affected = await _context.Items
       .Where(i => i.Id == id && i.Stock >= qty)
       .ExecuteUpdateAsync(s => s.SetProperty(i => i.Stock, i => i.Stock - qty), ct);
   if (affected == 0) // not enough stock / already taken -> 409
   ```
4. **Operations that touch more than one table**: wrap them in `await using var tx = await _context.Database.BeginTransactionAsync(ct);` … `await tx.CommitAsync(ct);`.
5. **Idempotent POST (double-submit protection on the server)**: if the task has "create order / payment / booking"-style actions, accept an optional `Idempotency-Key` header, store it in a column with a unique index, and return the existing result if the same key comes again.
6. DbContext is scoped (one per request) and is never shared across threads. No `Task.WhenAll` on the same context.

### Error handling and Program.cs
- `ExceptionHandlingMiddleware`: logs with `ILogger`, maps known exceptions (custom `NotFoundException` → 404, `ConflictException` / `DbUpdateConcurrencyException` → 409, `ValidationException` → 400) and returns everything else as 500. All errors use the same JSON shape `{ error: string, details?: object }`. Never leak stack traces.
- Make `[ApiController]` model validation return the same shape: configure `ApiBehaviorOptions.InvalidModelStateResponseFactory` to return `{ error: "Validation failed", details: { field: [messages] } }`.
- CORS policy allowing `http://localhost:5173`, with `AllowAnyHeader().AllowAnyMethod()`.
- Swagger in Development.
- Register `AddScoped<IXRepository, XRepository>()` for each repository.

### Example of the style I want (follow this pattern)
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;
    private readonly IMapper _mapper;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(IProductRepository repository, IMapper mapper, ILogger<ProductsController> logger)
    { _repository = repository; _mapper = mapper; _logger = logger; }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductDTO>> GetById(int id, CancellationToken ct)
    {
        var entity = await _repository.GetByIdAsync(id, ct);   // AsNoTracking inside
        if (entity == null) return NotFound(new { error = "Product not found" });
        return Ok(_mapper.Map<ProductDTO>(entity));
    }
}
```

---

## 2. FRONTEND: React + Vite, plain JavaScript (NO TypeScript), axios

### Stack
React 19 + Vite, JavaScript only (`.jsx` / `.js`), axios, MUI (`@mui/material`, `@mui/icons-material`) for UI, react-hook-form + yup (`@hookform/resolvers`) for forms, Redux Toolkit + react-redux for shared state, and react-router-dom only if there is more than one page.

### Folder structure
```
src/
  api/axios.js          // ONE axios instance: baseURL = import.meta.env.VITE_API_URL, interceptors
  services/             // ALL endpoints live here, one file per resource: productsService.js
  slices/               // Redux Toolkit slices + createAsyncThunk that CALL services (never axios directly)
  app/store.js          // configureStore
  hooks/                // custom hooks (useDebounce, useXxx for local data fetching, etc.)
  components/           // reusable UI: XForm.jsx, ConfirmDialog.jsx, ToastProvider.jsx, ...
  pages/                // page components
  App.jsx, main.jsx, index.css
.env                    // VITE_API_URL=https://localhost:<port>/api
```

### Rules
- **api/axios.js**: one instance. A response interceptor turns errors into one shape: `err.response?.data?.error || err.message || 'Something went wrong'`. Ignore cancelled requests (`axios.isCancel` / `err.code === 'ERR_CANCELED'`) and never show a toast for them.
- **services/xService.js**: small functions that return `res.data`, for example:
  ```js
  import api from '../api/axios';
  export const getProducts = (params, signal) => api.get('/products', { params, signal }).then(r => r.data);
  export const getProduct  = (id, signal)     => api.get(`/products/${id}`, { signal }).then(r => r.data);
  export const createProduct = (data, idempotencyKey) =>
    api.post('/products', data, { headers: idempotencyKey ? { 'Idempotency-Key': idempotencyKey } : {} }).then(r => r.data);
  export const updateProduct = (id, data) => api.put(`/products/${id}`, data).then(r => r.data);
  export const deleteProduct = (id, rowVersion) => api.delete(`/products/${id}`, { data: { rowVersion } });
  ```
  Components and slices never build URLs. Only services do.
- **slices**: `createAsyncThunk` calls the service, dispatches success and error toasts (a toastSlice + ToastProvider using an MUI Snackbar), and returns `rejectWithValue`. Keep separate loading flags (`fetchLoading`, `mutationLoading`) or a per-item `pendingIds` list, and use `error` state.
- Components use `useSelector` / `useDispatch`. They keep form state in react-hook-form, not Redux.

### Race conditions and double-click protection (required on the frontend)
1. **Double submit / multiple clicks**: every action button (submit, delete, save, buy, and so on) is protected **in two ways**:
   - It is `disabled` while its request is in flight (`isSubmitting` from react-hook-form, or a per-item pending id), with a `CircularProgress` inside.
   - A **synchronous `useRef` lock**, because React state updates are async and two fast clicks can both get through before the re-render:
     ```js
     const inFlight = useRef(false);
     const onSubmit = async (data) => {
       if (inFlight.current) return;
       inFlight.current = true;
       try { await dispatch(createX(data)).unwrap(); onClose(); }
       catch { /* toast already shown */ }
       finally { inFlight.current = false; }
     };
     ```
   - For create-type actions, generate an idempotency key once per form open (`crypto.randomUUID()`) and send it in the `Idempotency-Key` header, so even a retry can't create a duplicate.
   - Use `.unwrap()` so the dialog closes **only on success** and stays open with the error on failure.
2. **Stale / out-of-order responses**: every fetch in a `useEffect` uses an `AbortController` and aborts in the cleanup. This also handles React StrictMode running effects twice. For thunks, use `const promise = dispatch(fetchX(args)); return () => promise.abort();` and pass `thunkAPI.signal` to the service.
3. **Search / filter inputs**: debounce them with a `useDebounce` hook (about 400ms) and abort the previous request.
4. **409 Conflict handling**: on a 409, show "Modified by someone else, data refreshed", re-fetch the item or list, and keep the dialog open with fresh data. Always send `rowVersion` back on update and delete.
5. **Delete**: confirm first (an MUI Dialog, or `window.confirm` is OK). Only the item being deleted shows a spinner and is disabled (track `deletingId`). Remove it from state only after the server confirms.
6. **Lists after mutations**: update the Redux state from the server response (no blind refetch storms). If you use optimistic updates, roll back on failure.
7. Never mutate state outside reducers. Use `key={item.id}`, never the index.

### Validation
- Use a yup schema that mirrors the backend DataAnnotations (required, max length, ranges, dates not in the past, and so on). Show field errors with `helperText`.
- Also show backend validation errors: map `details` from a 400 response onto the form fields with `setError`.

### UX minimum
Loading spinner on the first load, an empty state ("No items yet"), an error state with a Retry button, and toasts for success and error.

---

## 3. Deliverables
1. All backend files (full content), including Program.cs, the DbContext, entities, DTOs, profiles, interfaces, repositories, controllers and middleware.
2. All frontend files (full content).
3. Commands to run, in order:
   - Backend: `dotnet restore`, set the connection string, `dotnet ef migrations add InitialCreate`, `dotnet ef database update`, `dotnet run`.
   - Frontend: `npm create vite@latest <name> -- --template react`, `npm i axios @mui/material @mui/icons-material @emotion/react @emotion/styled react-hook-form yup @hookform/resolvers @reduxjs/toolkit react-redux`, set `.env`, `npm run dev`.
4. A short "talking points" list I can use to explain to the interviewer: where AsNoTracking is used and why, how concurrency (RowVersion → 409) works, how double-click is prevented (disabled button + ref lock + idempotency key + unique index), how stale requests are cancelled, and why the architecture is split into Controller / Interface / Repository / DTO / Mapper on the backend and api / services / slices / hooks / components / pages on the frontend.
5. dont leave any comments or anything useless because it needs to be ready to production and to show it to the intetviewer
6.dont make things over engineering and overkill i want it simple as possible so i can explain
7. dont add a readme file or anything that is not neceesary i want only the useful files to make it quick and as fast as possible
8. dont use this attributes [ProducesResponseType] im not familiar with those
9. use the idempotency and or race condition only if needed and its must
10. instead of dates format and more folders and files use moment library its easier
11. at the end, explain in hebrew about the flow and how i should explain them about the project
