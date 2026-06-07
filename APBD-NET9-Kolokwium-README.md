# APBD .NET 9 — Wielka ściąga na kolokwium (REST API + EF Core)

> Kompletne notatki do rozwiązywania zadań typu „REST Web API + Entity Framework Core”.
> Pokrywa **Code First** (encje, relacje, migracje, seed, DTO, warstwy) oraz **Database First** (scaffold, filtrowanie LIKE, złożone zapytania z okresami czasu).
> Wszystko, co tu jest, wystarcza do napisania całego projektu od zera.

---

## Spis treści

1. [Szybki start — od zera do działającego API](#1-szybki-start)
2. [Pakiety NuGet i narzędzia CLI](#2-pakiety-nuget-i-narzędzia-cli)
3. [Struktura projektu (warstwy)](#3-struktura-projektu-warstwy)
4. [Code First vs Database First — co wybrać](#4-code-first-vs-database-first)
5. [CODE FIRST — encje](#5-code-first--encje)
6. [Data Annotations — pełna lista](#6-data-annotations--pełna-lista)
7. [Fluent API — pełna lista](#7-fluent-api--pełna-lista)
8. [Relacje: 1:1, 1:N, N:M](#8-relacje-11-1n-nm)
9. [Klucze złożone i klucze stringowe](#9-klucze-złożone)
10. [DbContext + connection string + Program.cs](#10-dbcontext--connection-string--programcs)
11. [Migracje — workflow i komendy](#11-migracje)
12. [Seed data (HasData)](#12-seed-data-hasdata)
13. [DATABASE FIRST — scaffold](#13-database-first--scaffold)
14. [DTO — co, po co i jak mapować](#14-dto)
15. [Kontrolery — atrybuty, routing, binding](#15-kontrolery)
16. [Pełne przykłady endpointów (GET/POST/PUT/DELETE)](#16-pełne-przykłady-endpointów)
17. [Warstwa serwisów + Dependency Injection](#17-warstwa-serwisów--di)
18. [Kody HTTP — tabela i jak je zwracać](#18-kody-http)
19. [Async / await](#19-async--await)
20. [LINQ — przewodnik po zapytaniach](#20-linq--zapytania)
21. [Filtrowanie / search / LIKE / wildcardy](#21-filtrowanie--like)
22. [Złożone zapytania — wolny zasób w okresie czasu](#22-złożone-zapytania--okresy-czasu)
23. [Walidacja danych wejściowych](#23-walidacja)
24. [Obsługa błędów / ProblemDetails](#24-obsługa-błędów)
25. [Minimal API (alternatywa dla kontrolerów)](#25-minimal-api)
26. [Transakcje](#26-transakcje)
27. [Częste pułapki i błędy](#27-częste-pułapki)
28. [Ściąga komend CLI](#28-ściąga-komend-cli)
29. [Gotowe szablony do skopiowania](#29-gotowe-szablony)

---

## 1. Szybki start

```bash
# utworzenie projektu Web API (.NET 9), bez Minimal API (z kontrolerami)
dotnet new webapi --use-controllers -n MyApi
cd MyApi

# albo klasycznie (starszy szablon też ma kontrolery):
# dotnet new webapi -controllers -n MyApi

dotnet run
```

> `--use-controllers` daje strukturę z `Controllers/`. Bez tej flagi .NET 9 generuje Minimal API w `Program.cs`.

Sprawdź wersję SDK:
```bash
dotnet --version   # powinno być 9.0.x
```

---

## 2. Pakiety NuGet i narzędzia CLI

**EF Core + SQL Server (najczęstszy zestaw na kolokwium):**
```bash
dotnet add package Microsoft.EntityFrameworkCore --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Design --version 9.0.0   # migracje + scaffold
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 9.0.0    # PMC (Add-Migration itd.)
```

> Dopasuj wersję do zainstalowanego SDK. Jeśli `9.0.0` nie istnieje, użyj `dotnet add package <pkg>` bez `--version` (weźmie najnowszą 9.x).

**Narzędzie `dotnet ef` (globalne) — potrzebne do migracji i scaffold:**
```bash
dotnet tool install --global dotnet-ef          # pierwsza instalacja
dotnet tool update  --global dotnet-ef          # aktualizacja
dotnet ef --version                             # sprawdzenie
```

**Swagger UI (opcjonalnie — w .NET 9 nie ma go domyślnie):**
```bash
dotnet add package Swashbuckle.AspNetCore
```
.NET 9 domyślnie używa `Microsoft.AspNetCore.OpenApi` (`AddOpenApi()` / `MapOpenApi()`), które generuje tylko dokument JSON pod `/openapi/v1.json`, bez UI. Jeśli chcesz klasyczny interfejs Swaggera — dodaj Swashbuckle (patrz §10).

**Inne providery bazodanowe (gdyby nie było SQL Server):**
```bash
# SQLite (lekka, plikowa — wygodna gdy brak SQL Servera):
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
# PostgreSQL:
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
# In-Memory (do testów, NIE używaj na produkcji):
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

---

## 3. Struktura projektu (warstwy)

Wymóg z zadania: **rozdziel logikę biznesową od warstwy API**, nie zwracaj encji bezpośrednio (używaj DTO).

```
MyApi/
├── Controllers/            # warstwa API/prezentacji (cienkie kontrolery)
│   └── PcsController.cs
├── Data/
│   └── AppDbContext.cs     # kontekst EF Core
├── Models/                 # encje (mapowane na tabele)
│   ├── Pc.cs
│   ├── Component.cs
│   ├── PcComponent.cs
│   ├── Manufacturer.cs
│   └── ComponentType.cs
├── DTOs/                   # obiekty transferowe (osobne dla in/out)
│   ├── PcGetAllDto.cs
│   ├── PcDetailsDto.cs
│   ├── CreatePcDto.cs
│   └── UpdatePcDto.cs
├── Services/               # warstwa logiki biznesowej
│   ├── IPcService.cs
│   └── PcService.cs
├── Migrations/             # generowane przez EF (commituj do repo!)
├── appsettings.json        # connection string
└── Program.cs              # konfiguracja DI, pipeline
```

Przepływ żądania:
```
HTTP → Controller → Service (logika) → DbContext (EF Core) → Baza
                 ←  DTO        ←       Encje
```

---

## 4. Code First vs Database First

| | **Code First** | **Database First (scaffold)** |
|---|---|---|
| Punkt startu | Piszesz klasy C# | Masz gotową bazę / skrypt `create.sql` |
| Generowanie | Klasy → migracje → baza | Baza → `scaffold` → klasy + DbContext |
| Komenda kluczowa | `dotnet ef migrations add` + `database update` | `dotnet ef dbcontext scaffold` |
| Zadanie 1 (komputery) | ✅ to jest Code First | |
| Zadanie 2 (szpital) | | ✅ to jest Database First |

**Zasada rozpoznania na kolokwium:**
- „utwórz klasy encji”, „migracje”, „HasData/seed” → **Code First**.
- „dostarczony `create.sql`”, „scaffold”, „DatabaseFirst” → **Database First**.

---

## 5. CODE FIRST — encje

Encja = klasa C# mapowana na tabelę. Konwencje EF Core:
- Właściwość `Id` lub `<Nazwa>Id` → automatycznie klucz główny (PK).
- Typy referencyjne (`string`) są domyślnie `nullable` w EF, ale w C# z włączonym nullable trzeba uważać (patrz niżej).
- Właściwości nawigacyjne tworzą relacje.

Przykład pełnego modelu z **Zadania 1** (komputery + komponenty, N:M z ilością `amount`, plus `Manufacturer` i `ComponentType`):

```csharp
// Models/Pc.cs
public class Pc
{
    public int Id { get; set; }                 // PK (konwencja)
    public string Name { get; set; } = null!;
    public double Weight { get; set; }
    public int Warranty { get; set; }           // okres gwarancji (miesiące)
    public DateTime CreatedAt { get; set; }
    public int Stock { get; set; }              // stan magazynowy

    // nawigacja: tabela łącząca N:M z payloadem (amount)
    public ICollection<PcComponent> PcComponents { get; set; } = new List<PcComponent>();
}
```

```csharp
// Models/Component.cs  (PK to string "code", np. "CPU0000001")
public class Component
{
    public string Code { get; set; } = null!;       // PK stringowy
    public string Name { get; set; } = null!;
    public string? Description { get; set; }

    public int ManufacturerId { get; set; }          // FK
    public Manufacturer Manufacturer { get; set; } = null!;

    public int TypeId { get; set; }                  // FK
    public ComponentType Type { get; set; } = null!;

    public ICollection<PcComponent> PcComponents { get; set; } = new List<PcComponent>();
}
```

```csharp
// Models/PcComponent.cs  (tabela łącząca N:M z dodatkową kolumną amount)
public class PcComponent
{
    public int PcId { get; set; }                    // FK + część PK złożonego
    public Pc Pc { get; set; } = null!;

    public string ComponentCode { get; set; } = null!; // FK + część PK złożonego
    public Component Component { get; set; } = null!;

    public int Amount { get; set; }                  // ilość w zestawie
}
```

```csharp
// Models/Manufacturer.cs
public class Manufacturer
{
    public int Id { get; set; }
    public string Abbreviation { get; set; } = null!;
    public string FullName { get; set; } = null!;
    public DateOnly FoundationDate { get; set; }     // sama data, bez czasu

    public ICollection<Component> Components { get; set; } = new List<Component>();
}
```

```csharp
// Models/ComponentType.cs
public class ComponentType
{
    public int Id { get; set; }
    public string Abbreviation { get; set; } = null!;
    public string Name { get; set; } = null!;

    public ICollection<Component> Components { get; set; } = new List<Component>();
}
```

> `= null!;` ucisza ostrzeżenia nullable dla wymaganych pól. Alternatywa: dodać `required` (C# 11+): `public required string Name { get; set; }`.

---

## 6. Data Annotations — pełna lista

Atrybuty z `using System.ComponentModel.DataAnnotations;` oraz `System.ComponentModel.DataAnnotations.Schema;`.

```csharp
public class Product
{
    [Key]                                   // jawny klucz główny
    public int Id { get; set; }

    [Required]                              // NOT NULL
    [MaxLength(100)]                        // nvarchar(100)
    [MinLength(3)]
    public string Name { get; set; } = null!;

    [StringLength(50, MinimumLength = 2)]   // długość min/max w jednym
    public string Code { get; set; } = null!;

    [Column("price_net", TypeName = "decimal(10,2)")]  // nazwa kolumny + typ SQL
    public decimal Price { get; set; }

    [Range(0, 9999)]                        // walidacja zakresu (działa też w API)
    public int Stock { get; set; }

    [NotMapped]                             // NIE tworzy kolumny w bazie
    public string DisplayName => $"{Code} - {Name}";

    [ForeignKey(nameof(Category))]          // jawny FK
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;

    [Table("Products")]                     // (na klasie) nazwa tabeli
    // [DatabaseGenerated(DatabaseGeneratedOption.Identity)]  // auto-increment
    // [DatabaseGenerated(DatabaseGeneratedOption.None)]      // wyłącz auto-PK
}
```

Najważniejsze (zapamiętaj):
- `[Key]` — PK.
- `[Required]` — NOT NULL.
- `[MaxLength(n)]` / `[StringLength(n)]` — max długość tekstu.
- `[Column(TypeName="...")]` — konkretny typ SQL (np. `decimal(18,2)`).
- `[ForeignKey("Nazwa")]` — klucz obcy.
- `[NotMapped]` — pomiń w bazie.
- `[Table("...")]` — nazwa tabeli.

---

## 7. Fluent API — pełna lista

Konfiguracja w `OnModelCreating` w `DbContext`. **Fluent API ma priorytet nad Data Annotations** i jest bardziej elastyczne (klucze złożone, relacje N:M, kaskady).

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    // --- KLUCZ GŁÓWNY ---
    modelBuilder.Entity<Component>().HasKey(c => c.Code);

    // --- KLUCZ ZŁOŻONY (composite key) ---
    modelBuilder.Entity<PcComponent>().HasKey(pc => new { pc.PcId, pc.ComponentCode });

    // --- WŁAŚCIWOŚCI ---
    modelBuilder.Entity<Pc>(e =>
    {
        e.Property(p => p.Name).IsRequired().HasMaxLength(100);
        e.Property(p => p.Weight).HasColumnType("decimal(6,2)");
        e.Property(p => p.CreatedAt).HasDefaultValueSql("GETDATE()"); // wartość domyślna SQL
        e.HasIndex(p => p.Name).IsUnique();                           // indeks unikalny
        e.ToTable("PCs");                                             // nazwa tabeli
    });

    // --- RELACJA 1:N ---
    modelBuilder.Entity<Component>()
        .HasOne(c => c.Manufacturer)         // Component ma jednego Manufacturer
        .WithMany(m => m.Components)          // Manufacturer ma wiele Component
        .HasForeignKey(c => c.ManufacturerId)
        .OnDelete(DeleteBehavior.Restrict);  // zakaz kaskadowego usuwania

    // --- RELACJA N:M z payloadem (przez encję łączącą) ---
    modelBuilder.Entity<PcComponent>()
        .HasOne(pc => pc.Pc)
        .WithMany(p => p.PcComponents)
        .HasForeignKey(pc => pc.PcId)
        .OnDelete(DeleteBehavior.Cascade);   // usunięcie PC kasuje wpisy łączące

    modelBuilder.Entity<PcComponent>()
        .HasOne(pc => pc.Component)
        .WithMany(c => c.PcComponents)
        .HasForeignKey(pc => pc.ComponentCode);
}
```

`DeleteBehavior`:
- `Cascade` — usuń dzieci razem z rodzicem (potrzebne dla DELETE w zadaniu 1).
- `Restrict` / `NoAction` — zablokuj usunięcie, jeśli są powiązania.
- `SetNull` — ustaw FK na NULL (FK musi być nullable).

---

## 8. Relacje: 1:1, 1:N, N:M

### 1:N (jeden-do-wielu) — najczęstsza
Jeden `Manufacturer` ma wiele `Component`:
```csharp
public class Manufacturer
{
    public int Id { get; set; }
    public ICollection<Component> Components { get; set; } = new List<Component>();
}
public class Component
{
    public string Code { get; set; } = null!;
    public int ManufacturerId { get; set; }            // FK
    public Manufacturer Manufacturer { get; set; } = null!;
}
```
EF rozpozna to po konwencji (FK = `ManufacturerId`), Fluent niepotrzebny — ale można doprecyzować jak w §7.

### 1:1 (jeden-do-jednego)
```csharp
public class User { public int Id; public Profile Profile { get; set; } = null!; }
public class Profile
{
    public int Id { get; set; }
    public int UserId { get; set; }                    // FK + zwykle unikalny
    public User User { get; set; } = null!;
}
```
```csharp
modelBuilder.Entity<User>()
    .HasOne(u => u.Profile)
    .WithOne(p => p.User)
    .HasForeignKey<Profile>(p => p.UserId);
```

### N:M (wiele-do-wielu)

**Wariant A — czysta N:M (bez dodatkowych kolumn).** EF Core 5+ tworzy tabelę łączącą automatycznie:
```csharp
public class Pc        { public ICollection<Component> Components { get; set; } = new List<Component>(); }
public class Component { public ICollection<Pc> Pcs { get; set; } = new List<Pc>(); }
// EF sam stworzy tabelę ComponentPc. Fluent (opcjonalnie):
modelBuilder.Entity<Pc>().HasMany(p => p.Components).WithMany(c => c.Pcs)
    .UsingEntity(j => j.ToTable("PcComponents"));
```

**Wariant B — N:M z payloadem (np. `amount`, ilość).** ⚠️ **To jest przypadek z Zadania 1** — musisz zrobić **jawną encję łączącą** (`PcComponent` z `Amount`), bo czysta N:M nie pomieści dodatkowej kolumny. Wzorzec: dwie relacje 1:N do tabeli łączącej z kluczem złożonym (patrz §5 i §7).

---

## 9. Klucze złożone

Klucz złożony (composite key) konfigurujesz **tylko przez Fluent API** (Data Annotation `[Key]` nie obsługuje wielu kolumn bez kolejności):

```csharp
modelBuilder.Entity<PcComponent>()
    .HasKey(pc => new { pc.PcId, pc.ComponentCode });
```

Klucz stringowy (jak `Component.Code = "CPU0000001"`):
```csharp
modelBuilder.Entity<Component>().HasKey(c => c.Code);
modelBuilder.Entity<Component>().Property(c => c.Code).HasMaxLength(10);
// Jeśli PK nie ma być auto-generowany:
modelBuilder.Entity<Component>().Property(c => c.Code)
    .ValueGeneratedNever();
```

---

## 10. DbContext + connection string + Program.cs

### DbContext
```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // jeden DbSet na encję
    public DbSet<Pc> Pcs => Set<Pc>();
    public DbSet<Component> Components => Set<Component>();
    public DbSet<PcComponent> PcComponents => Set<PcComponent>();
    public DbSet<Manufacturer> Manufacturers => Set<Manufacturer>();
    public DbSet<ComponentType> ComponentTypes => Set<ComponentType>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        // konfiguracja Fluent API + seed (HasData) — patrz §7 i §12
    }
}
```

### appsettings.json — connection string
```jsonc
{
  "ConnectionStrings": {
    // SQL Server (LocalDB — typowo na uczelni / Windows):
    "Default": "Server=(localdb)\\MSSQLLocalDB;Database=ApbdDb;Trusted_Connection=True;TrustServerCertificate=True;",
    // SQL Server pełny / Docker:
    // "Default": "Server=localhost,1433;Database=ApbdDb;User Id=sa;Password=Your_pass123;TrustServerCertificate=True;"
  },
  "Logging": { "LogLevel": { "Default": "Information" } },
  "AllowedHosts": "*"
}
```
> Na macOS/Linux LocalDB nie istnieje — użyj SQL Servera w Dockerze albo SQLite:
> `"Default": "Data Source=app.db"` + `options.UseSqlite(...)`.

### Program.cs (kontrolery + EF + Swagger)
```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// 1) rejestracja DbContext z connection string
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
    // .UseSqlite(...) / .UseNpgsql(...) zależnie od bazy

// 2) kontrolery
builder.Services.AddControllers();

// 3) rejestracja serwisów (DI) — patrz §17
builder.Services.AddScoped<IPcService, PcService>();

// 4) OpenAPI / Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddOpenApi();             // .NET 9 -> /openapi/v1.json
// builder.Services.AddSwaggerGen();       // jeśli używasz Swashbuckle

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();                       // dokument JSON
    // app.UseSwagger(); app.UseSwaggerUI(); // jeśli Swashbuckle
}

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

> **Swagger UI z Swashbuckle:** po `dotnet add package Swashbuckle.AspNetCore` zamień blok OpenAPI na
> `builder.Services.AddSwaggerGen();` oraz `app.UseSwagger(); app.UseSwaggerUI();` — wtedy UI jest pod `/swagger`.

---

## 11. Migracje

Migracja = wersjonowana zmiana schematu bazy generowana z Twoich encji.

### Workflow (CLI — działa wszędzie)
```bash
# 1) utwórz pierwszą migrację
dotnet ef migrations add InitialCreate

# 2) zastosuj migracje do bazy (utworzy/zaktualizuje bazę)
dotnet ef database update

# kolejna zmiana w encjach -> nowa migracja -> update:
dotnet ef migrations add AddStockColumn
dotnet ef database update

# usuń OSTATNIĄ (jeszcze niezaaplikowaną) migrację:
dotnet ef migrations remove

# cofnij bazę do konkretnej migracji:
dotnet ef database update InitialCreate

# wygeneruj skrypt SQL ze wszystkich migracji:
dotnet ef migrations script -o migrations.sql
```

### Workflow (Package Manager Console w Visual Studio)
```powershell
Add-Migration InitialCreate
Update-Database
Remove-Migration
```

> ⚠️ **Commituj cały folder `Migrations/` do repo** (wymóg zadania).
> ⚠️ Jeśli masz wiele projektów, dodaj `--project` / `--startup-project`.
> ⚠️ Po zmianie encji ZAWSZE: nowa migracja → `database update`. Sama zmiana klasy nie zmienia bazy.

---

## 12. Seed data (HasData)

Wymóg: min. 3 rekordy na tabelę z poprawnymi relacjami. Robisz to w `OnModelCreating`. Po dodaniu seedu trzeba utworzyć migrację i zrobić `database update`.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    // klucze/relacje (jak w §7) ...

    // --- SEED: tabele słownikowe najpierw ---
    modelBuilder.Entity<Manufacturer>().HasData(
        new Manufacturer { Id = 1, Abbreviation = "AMD", FullName = "Advanced Micro Devices", FoundationDate = new DateOnly(1969, 5, 1) },
        new Manufacturer { Id = 2, Abbreviation = "NV",  FullName = "NVIDIA Corporation",     FoundationDate = new DateOnly(1993, 4, 5) },
        new Manufacturer { Id = 3, Abbreviation = "COR", FullName = "Corsair Gaming Inc.",    FoundationDate = new DateOnly(1994, 1, 1) }
    );

    modelBuilder.Entity<ComponentType>().HasData(
        new ComponentType { Id = 1, Abbreviation = "CPU", Name = "Processor" },
        new ComponentType { Id = 2, Abbreviation = "GPU", Name = "Graphics Card" },
        new ComponentType { Id = 3, Abbreviation = "RAM", Name = "Memory" }
    );

    modelBuilder.Entity<Component>().HasData(
        new Component { Code = "CPU0000001", Name = "Ryzen 7 7800X3D", Description = "8-core gaming processor", ManufacturerId = 1, TypeId = 1 },
        new Component { Code = "GPU0000001", Name = "RTX 4080 Super",  Description = "High-end GPU",            ManufacturerId = 2, TypeId = 2 },
        new Component { Code = "RAM0000001", Name = "Corsair DDR5 16GB", Description = "DDR5 module",           ManufacturerId = 3, TypeId = 3 }
    );

    modelBuilder.Entity<Pc>().HasData(
        new Pc { Id = 1, Name = "Gaming Beast X", Weight = 12.5, Warranty = 36, CreatedAt = new DateTime(2026, 5, 8, 9, 0, 0), Stock = 5 },
        new Pc { Id = 2, Name = "Office Mini Pro", Weight = 4.2, Warranty = 24, CreatedAt = new DateTime(2026, 4, 15, 13, 30, 0), Stock = 12 },
        new Pc { Id = 3, Name = "Workstation Z",   Weight = 9.0, Warranty = 48, CreatedAt = new DateTime(2026, 3, 1, 8, 0, 0), Stock = 3 }
    );

    // --- SEED tabeli łączącej (klucze złożone, FK muszą wskazywać na istniejące rekordy) ---
    modelBuilder.Entity<PcComponent>().HasData(
        new PcComponent { PcId = 1, ComponentCode = "CPU0000001", Amount = 1 },
        new PcComponent { PcId = 1, ComponentCode = "GPU0000001", Amount = 1 },
        new PcComponent { PcId = 1, ComponentCode = "RAM0000001", Amount = 2 }
    );
}
```

Zasady seedowania:
- **Podawaj jawnie wszystkie PK** (nawet auto-increment) — EF tego wymaga w `HasData`.
- Seeduj **rodziców przed dziećmi** (najpierw słowniki, na końcu tabele łączące).
- Używaj **stałych wartości** (nie `DateTime.Now`, nie `Guid.NewGuid()`) — inaczej każda migracja będzie się różnić.
- Po `HasData`: `dotnet ef migrations add Seed` → `dotnet ef database update`.

---

## 13. DATABASE FIRST — scaffold

Masz gotowy skrypt `create.sql` (np. Zadanie 2 — szpital). Najpierw tworzysz bazę z tego skryptu, potem **reverse-engineering** (scaffold) generuje encje + DbContext.

### Krok 1 — utwórz bazę ze skryptu
- W SQL Server Management Studio / Azure Data Studio: otwórz `create.sql` i wykonaj.
- Albo z CLI (`sqlcmd`): `sqlcmd -S (localdb)\MSSQLLocalDB -i create.sql`

### Krok 2 — scaffold (CLI)
```bash
dotnet ef dbcontext scaffold \
  "Server=(localdb)\MSSQLLocalDB;Database=Hospital;Trusted_Connection=True;TrustServerCertificate=True;" \
  Microsoft.EntityFrameworkCore.SqlServer \
  --output-dir Models \
  --context-dir Data \
  --context HospitalContext \
  --use-database-names \
  --no-onconfiguring \
  --data-annotations
```

Najważniejsze opcje:
| Opcja | Znaczenie |
|---|---|
| `-o, --output-dir` | folder na wygenerowane encje |
| `--context-dir` | folder na klasę kontekstu |
| `-c, --context` | nazwa klasy kontekstu |
| `--use-database-names` | nazwy klas/właściwości jak w bazie |
| `--data-annotations` | użyj atrybutów zamiast całości Fluent |
| `--no-onconfiguring` | NIE wstawiaj connection stringa w kodzie (zamiast tego rejestruj w `Program.cs`) |
| `-t, --table` | scaffold tylko wybranych tabel (`-t Patients -t Beds`) |
| `-f, --force` | nadpisz istniejące pliki (przy ponownym scaffold) |

### Package Manager Console (Visual Studio):
```powershell
Scaffold-DbContext "Server=(localdb)\MSSQLLocalDB;Database=Hospital;Trusted_Connection=True;TrustServerCertificate=True;" Microsoft.EntityFrameworkCore.SqlServer -OutputDir Models -Context HospitalContext
```

### Krok 3 — przenieś connection string do appsettings i Program.cs
Po `--no-onconfiguring` zarejestruj kontekst jak w §10:
```csharp
builder.Services.AddDbContext<HospitalContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

> ⚠️ Po scaffold **nie używasz migracji** (baza już istnieje). Jeśli baza się zmieni — scaffold ponownie z `--force`.
> ⚠️ Bez `--no-onconfiguring` EF wstawi connection string na sztywno w `OnConfiguring` — to ostrzeżenie/zły styl. Usuń to.

---

## 14. DTO

**DTO (Data Transfer Object)** = klasa do przesyłania danych przez API, oddzielona od encji.
Po co: bezpieczeństwo (nie ujawniasz struktury bazy), kontrola pól, brak cykli w serializacji (encje z nawigacjami się zapętlają!).

Wymóg z zadania: osobne DTO dla **danych wychodzących**, **danych wejściowych**, **tworzenia/aktualizacji**.

```csharp
// DTOs/PcGetAllDto.cs — odpowiedź GET /api/pcs (płaska lista)
public class PcGetAllDto
{
    public int Id { get; set; }
    public string Name { get; set; } = null!;
    public double Weight { get; set; }
    public int Warranty { get; set; }
    public DateTime CreatedAt { get; set; }
    public int Stock { get; set; }
}

// DTOs/CreatePcDto.cs — wejście POST (bez Id!)
public class CreatePcDto
{
    [Required, MaxLength(100)]
    public string Name { get; set; } = null!;
    [Range(0, double.MaxValue)]
    public double Weight { get; set; }
    public int Warranty { get; set; }
    public DateTime CreatedAt { get; set; }
    public int Stock { get; set; }
}

// DTOs/UpdatePcDto.cs — wejście PUT
public class UpdatePcDto
{
    [Required, MaxLength(100)]
    public string Name { get; set; } = null!;
    public double Weight { get; set; }
    public int Warranty { get; set; }
    public DateTime CreatedAt { get; set; }
    public int Stock { get; set; }
}

// DTOs/PcDetailsDto.cs — odpowiedź GET /api/pcs/{id}/components (zagnieżdżone)
public class PcComponentDto
{
    public int Amount { get; set; }
    public ComponentDto Component { get; set; } = null!;
}
public class ComponentDto
{
    public string Code { get; set; } = null!;
    public string Name { get; set; } = null!;
    public string? Description { get; set; }
    public ManufacturerDto Manufacturer { get; set; } = null!;
    public TypeDto Type { get; set; } = null!;
}
public class ManufacturerDto
{
    public int Id { get; set; }
    public string Abbreviation { get; set; } = null!;
    public string FullName { get; set; } = null!;
    public DateOnly FoundationDate { get; set; }
}
public class TypeDto
{
    public int Id { get; set; }
    public string Abbreviation { get; set; } = null!;
    public string Name { get; set; } = null!;
}
```

### Mapowanie ręczne (bezpieczne na kolokwium, zero zależności)
Najlepiej mapować od razu w zapytaniu LINQ przez `.Select(...)` (EF wygeneruje optymalny SQL i pobierze tylko potrzebne kolumny):
```csharp
var pcs = await _context.Pcs
    .Select(p => new PcGetAllDto
    {
        Id = p.Id, Name = p.Name, Weight = p.Weight,
        Warranty = p.Warranty, CreatedAt = p.CreatedAt, Stock = p.Stock
    })
    .ToListAsync();
```

> Możesz użyć AutoMappera (`dotnet add package AutoMapper`), ale na kolokwium ręczne mapowanie jest pewniejsze i nie wymaga konfiguracji.

---

## 15. Kontrolery

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]                          // automatyczna walidacja ModelState + lepsze błędy 400
[Route("api/pcs")]                       // bazowa ścieżka (albo [Route("api/[controller]")])
public class PcsController : ControllerBase
{
    private readonly IPcService _service;
    public PcsController(IPcService service) => _service = service;   // DI przez konstruktor

    // GET api/pcs
    [HttpGet]
    public async Task<IActionResult> GetAll() => Ok(await _service.GetAllAsync());

    // GET api/pcs/5/components
    [HttpGet("{id:int}/components")]
    public async Task<IActionResult> GetComponents(int id) { ... }

    // POST api/pcs
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreatePcDto dto) { ... }
}
```

Bindowanie parametrów:
| Źródło | Atrybut | Przykład |
|---|---|---|
| Segment URL | `[FromRoute]` (domyślnie dla nazw z trasy) | `/api/pcs/{id}` → `int id` |
| Query string | `[FromQuery]` | `/api/patients?search=an` → `string? search` |
| Body JSON | `[FromBody]` | `CreatePcDto dto` |
| Nagłówek | `[FromHeader]` | |

Ograniczenia trasy (route constraints): `{id:int}`, `{pesel:length(11)}`, `{code:alpha}`, `{id:guid}`.

Metody zwracające status (z `ControllerBase`):
```csharp
return Ok(data);                       // 200
return Created($"api/pcs/{id}", dto);  // 201 + nagłówek Location
return CreatedAtAction(nameof(GetById), new { id }, dto); // 201
return NoContent();                    // 204
return BadRequest("komunikat");        // 400
return NotFound("Nie znaleziono PC");  // 404
return Conflict("Konflikt");           // 409
return StatusCode(500, "błąd");        // dowolny kod
```

---

## 16. Pełne przykłady endpointów

Kompletny kontroler dla Zadania 1 (logika w serwisie — patrz §17, tu pokazane też „wszystko w kontrolerze” dla zwięzłości; na kolokwium przenieś logikę do serwisu).

```csharp
[ApiController]
[Route("api/pcs")]
public class PcsController : ControllerBase
{
    private readonly AppDbContext _context;
    public PcsController(AppDbContext context) => _context = context;

    // GET api/pcs  -> lista podstawowych danych
    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var pcs = await _context.Pcs
            .Select(p => new PcGetAllDto
            {
                Id = p.Id, Name = p.Name, Weight = p.Weight,
                Warranty = p.Warranty, CreatedAt = p.CreatedAt, Stock = p.Stock
            })
            .ToListAsync();
        return Ok(pcs);                                  // 200
    }

    // GET api/pcs/{id}/components  -> komponenty danego PC (404 jeśli brak PC)
    [HttpGet("{id:int}/components")]
    public async Task<IActionResult> GetComponents(int id)
    {
        var pc = await _context.Pcs
            .Where(p => p.Id == id)
            .Select(p => new
            {
                p.Id, p.Name, p.Weight, p.Warranty, p.CreatedAt, p.Stock,
                Components = p.PcComponents.Select(pc => new PcComponentDto
                {
                    Amount = pc.Amount,
                    Component = new ComponentDto
                    {
                        Code = pc.Component.Code,
                        Name = pc.Component.Name,
                        Description = pc.Component.Description,
                        Manufacturer = new ManufacturerDto
                        {
                            Id = pc.Component.Manufacturer.Id,
                            Abbreviation = pc.Component.Manufacturer.Abbreviation,
                            FullName = pc.Component.Manufacturer.FullName,
                            FoundationDate = pc.Component.Manufacturer.FoundationDate
                        },
                        Type = new TypeDto
                        {
                            Id = pc.Component.Type.Id,
                            Abbreviation = pc.Component.Type.Abbreviation,
                            Name = pc.Component.Type.Name
                        }
                    }
                }).ToList()
            })
            .FirstOrDefaultAsync();

        if (pc is null) return NotFound($"PC with id {id} not found");  // 404
        return Ok(pc);                                                  // 200
    }

    // POST api/pcs  -> utwórz
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreatePcDto dto)
    {
        var pc = new Pc
        {
            Name = dto.Name, Weight = dto.Weight, Warranty = dto.Warranty,
            CreatedAt = dto.CreatedAt, Stock = dto.Stock
        };
        _context.Pcs.Add(pc);
        await _context.SaveChangesAsync();               // tu nadawane jest Id

        var result = new PcGetAllDto
        {
            Id = pc.Id, Name = pc.Name, Weight = pc.Weight,
            Warranty = pc.Warranty, CreatedAt = pc.CreatedAt, Stock = pc.Stock
        };
        return CreatedAtAction(nameof(GetComponents), new { id = pc.Id }, result); // 201
    }

    // PUT api/pcs/{id}  -> aktualizuj
    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdatePcDto dto)
    {
        var pc = await _context.Pcs.FindAsync(id);
        if (pc is null) return NotFound($"PC with id {id} not found");  // 404

        pc.Name = dto.Name; pc.Weight = dto.Weight; pc.Warranty = dto.Warranty;
        pc.CreatedAt = dto.CreatedAt; pc.Stock = dto.Stock;

        await _context.SaveChangesAsync();
        return Ok(pc);                                   // 200 (albo NoContent())
    }

    // DELETE api/pcs/{id}  -> usuń (kaskada na PcComponents), 204
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        var pc = await _context.Pcs.FindAsync(id);
        if (pc is null) return NotFound($"PC with id {id} not found");  // 404

        _context.Pcs.Remove(pc);                         // kaskada usunie PcComponents
        await _context.SaveChangesAsync();
        return NoContent();                              // 204
    }
}
```

---

## 17. Warstwa serwisów + DI

Zalecane przez zadanie: cienki kontroler + logika w serwisie.

```csharp
// Services/IPcService.cs
public interface IPcService
{
    Task<IEnumerable<PcGetAllDto>> GetAllAsync();
    Task<PcDetailsDto?> GetComponentsAsync(int id);
    Task<PcGetAllDto> CreateAsync(CreatePcDto dto);
    Task<bool> UpdateAsync(int id, UpdatePcDto dto);
    Task<bool> DeleteAsync(int id);
}
```

```csharp
// Services/PcService.cs
public class PcService : IPcService
{
    private readonly AppDbContext _context;
    public PcService(AppDbContext context) => _context = context;

    public async Task<IEnumerable<PcGetAllDto>> GetAllAsync() =>
        await _context.Pcs.Select(p => new PcGetAllDto { /* ... */ }).ToListAsync();

    public async Task<bool> DeleteAsync(int id)
    {
        var pc = await _context.Pcs.FindAsync(id);
        if (pc is null) return false;
        _context.Pcs.Remove(pc);
        await _context.SaveChangesAsync();
        return true;
    }
    // ... reszta analogicznie
}
```

Rejestracja w `Program.cs`:
```csharp
builder.Services.AddScoped<IPcService, PcService>();
```

Cykle życia DI:
- `AddScoped` — jedna instancja na żądanie HTTP (**domyślnie dla serwisów z DbContextem**).
- `AddTransient` — nowa instancja przy każdym wstrzyknięciu.
- `AddSingleton` — jedna na całą aplikację (NIE dla DbContextu!).

Kontroler wtedy korzysta z serwisu i tłumaczy wynik na status:
```csharp
[HttpDelete("{id:int}")]
public async Task<IActionResult> Delete(int id) =>
    await _service.DeleteAsync(id) ? NoContent() : NotFound($"PC {id} not found");
```

---

## 18. Kody HTTP

| Kod | Nazwa | Kiedy zwracać | Metoda |
|---|---|---|---|
| 200 | OK | poprawny GET/PUT zwracający dane | `Ok(data)` |
| 201 | Created | utworzono zasób (POST) | `CreatedAtAction(...)` / `Created(...)` |
| 204 | No Content | sukces bez body (zwykle DELETE) | `NoContent()` |
| 400 | Bad Request | błędne dane wejściowe / walidacja | `BadRequest(msg)` |
| 401 | Unauthorized | brak uwierzytelnienia | `Unauthorized()` |
| 403 | Forbidden | brak uprawnień | `Forbid()` |
| 404 | Not Found | zasób nie istnieje | `NotFound(msg)` |
| 409 | Conflict | konflikt (np. duplikat) | `Conflict(msg)` |
| 500 | Internal Server Error | nieobsłużony błąd serwera | `StatusCode(500, msg)` |

Mapowanie operacji → kody (z zadań):
- `POST` sukces → **201**, błędne dane → **400**.
- `PUT` sukces → **200**, brak zasobu → **404**.
- `DELETE` sukces → **204**, brak zasobu → **404**.
- `GET by id` brak → **404**.
- „nie ma wolnego łóżka” → **404** z **czytelnym** komunikatem (Zadanie 2 wymaga jednoznacznych komunikatów, nie jednego ogólnego).

---

## 19. Async / await

Wymóg zadania: operacje na bazie **asynchronicznie**.

Reguły:
- Metoda kontrolera/serwisu: `public async Task<IActionResult> ...` lub `Task<T>`.
- Każde zapytanie EF kończące materializacją ma wariant `...Async` — używaj go z `await`.
- Nigdy nie blokuj asynchronicznego kodu przez `.Result` / `.Wait()` (deadlocki).

Najważniejsze metody async EF Core:
```csharp
await _context.Pcs.ToListAsync();
await _context.Pcs.FirstOrDefaultAsync(p => p.Id == id);
await _context.Pcs.SingleOrDefaultAsync(p => p.Id == id);
await _context.Pcs.FindAsync(id);          // szuka po PK, sprawdza cache
await _context.Pcs.AnyAsync(p => p.Stock > 0);
await _context.Pcs.CountAsync();
await _context.SaveChangesAsync();
```

---

## 20. LINQ — zapytania

```csharp
// FILTROWANIE
_context.Pcs.Where(p => p.Stock > 0)

// PROJEKCJA do DTO (pobiera tylko potrzebne kolumny)
_context.Pcs.Select(p => new PcGetAllDto { Id = p.Id, Name = p.Name })

// SORTOWANIE
_context.Pcs.OrderBy(p => p.Name).ThenByDescending(p => p.CreatedAt)

// PAGINACJA
_context.Pcs.Skip((page - 1) * size).Take(size)

// EAGER LOADING (dołączanie powiązanych — gdy NIE używasz Select)
_context.Pcs
    .Include(p => p.PcComponents)
        .ThenInclude(pc => pc.Component)
            .ThenInclude(c => c.Manufacturer)

// AGREGACJE
_context.Pcs.CountAsync();   .SumAsync(p => p.Stock);   .MaxAsync(p => p.Weight);

// GRUPOWANIE
_context.Components.GroupBy(c => c.TypeId)
    .Select(g => new { TypeId = g.Key, Count = g.Count() })

// ISTNIENIE
await _context.Pcs.AnyAsync(p => p.Id == id)

// POBRANIE JEDNEGO
.FirstOrDefaultAsync()   // pierwszy lub null
.SingleOrDefaultAsync()  // dokładnie jeden lub null (rzuci gdy >1)
.FindAsync(pk)           // po kluczu głównym
```

> `Include` vs `Select`: jeśli zwracasz DTO, użyj `Select` (mniej danych, brak cykli). `Include` używaj gdy potrzebujesz pełnych encji w pamięci.
> `AsNoTracking()` — przy odczytach tylko-do-zwrotu przyspiesza (brak śledzenia zmian): `_context.Pcs.AsNoTracking()...`.

---

## 21. Filtrowanie / search / LIKE / wildcardy

To dokładnie **Zadanie 2** (`/api/patients?search=an` filtrujące po `FirstName`/`LastName`).

### Sposób 1 — `Contains` (tłumaczy się na `LIKE '%search%'`)
```csharp
[HttpGet("/api/patients")]
public async Task<IActionResult> Get([FromQuery] string? search)
{
    var query = _context.Patients.AsQueryable();

    if (!string.IsNullOrWhiteSpace(search))
        query = query.Where(p =>
            p.FirstName.Contains(search) || p.LastName.Contains(search));

    var result = await query
        .Select(p => new PatientDto { Id = p.IdPatient, FirstName = p.FirstName, LastName = p.LastName })
        .ToListAsync();

    return Ok(result);
}
```

### Sposób 2 — `EF.Functions.Like` (jawny LIKE + wildcardy, zgodnie z podpowiedzią w zadaniu)
```csharp
if (!string.IsNullOrWhiteSpace(search))
{
    var pattern = $"%{search}%";
    query = query.Where(p =>
        EF.Functions.Like(p.FirstName, pattern) ||
        EF.Functions.Like(p.LastName, pattern));
}
```

Wildcardy w LIKE:
- `%` — dowolny ciąg znaków (`%an%` = zawiera „an”).
- `_` — dokładnie jeden dowolny znak.

> Parametr opcjonalny: `string? search` + sprawdzenie `IsNullOrWhiteSpace` — gdy brak parametru, zwróć wszystkich.
> Wielkość liter: w SQL Server domyślnie collation jest case-insensitive (`an` znajdzie `Jan`). W razie potrzeby case-insensitive jawnie: porównuj `ToLower()`.

---

## 22. Złożone zapytania — okresy czasu

To **Zadanie 2 Zad3**: znajdź łóżko danego typu, w danym oddziale, **wolne** w zadanym okresie `[from, to]`. Jeśli brak — 404 z czytelnym komunikatem.

### Logika nakładania się okresów (KLUCZOWE)
Dwa okresy `[from, to]` i `[existingFrom, existingTo]` **nachodzą na siebie** (kolizja), gdy:
```
existingFrom < to   AND   existingTo > from
```
Czyli łóżko jest **wolne**, gdy **NIE istnieje** żadne przypisanie spełniające powyższe.

```csharp
[HttpPost("/api/patients/{pesel}/bedassignments")]
public async Task<IActionResult> AssignBed(string pesel, [FromBody] BedAssignmentRequest dto)
{
    // 1) pacjent istnieje?
    var patient = await _context.Patients.FirstOrDefaultAsync(p => p.Pesel == pesel);
    if (patient is null)
        return NotFound($"Patient with PESEL {pesel} not found.");

    // 2) oddział istnieje?
    var ward = await _context.Wards.FirstOrDefaultAsync(w => w.IdWard == dto.WardId);
    if (ward is null)
        return NotFound($"Ward with id {dto.WardId} not found.");

    // 3) typ łóżka istnieje?
    var bedType = await _context.BedTypes.FirstOrDefaultAsync(t => t.IdBedType == dto.BedTypeId);
    if (bedType is null)
        return NotFound($"Bed type with id {dto.BedTypeId} not found.");

    // 4) znajdź WOLNE łóżko danego typu w danym oddziale w okresie [From, To]
    var freeBed = await _context.Beds
        .Where(b => b.WardId == dto.WardId && b.BedTypeId == dto.BedTypeId)
        .Where(b => !b.BedAssignments.Any(a =>
            a.From < dto.To && a.To > dto.From))      // negacja nakładania
        .FirstOrDefaultAsync();

    if (freeBed is null)
        return NotFound(
            $"No free bed of type {dto.BedTypeId} in ward {dto.WardId} for the period {dto.From:d} - {dto.To:d}.");

    // 5) utwórz przypisanie
    var assignment = new BedAssignment
    {
        BedId = freeBed.IdBed,
        PatientId = patient.IdPatient,
        From = dto.From,
        To = dto.To
    };
    _context.BedAssignments.Add(assignment);
    await _context.SaveChangesAsync();

    return Created($"/api/beds/{freeBed.IdBed}", new { freeBed.IdBed, patient.Pesel, dto.From, dto.To });
}
```

DTO żądania:
```csharp
public class BedAssignmentRequest
{
    public int WardId { get; set; }
    public int BedTypeId { get; set; }
    public DateTime From { get; set; }
    public DateTime To { get; set; }
}
```

> **Czytelne komunikaty 404** (wymóg zadania): osobny komunikat dla „brak pacjenta”, „brak oddziału”, „brak typu”, „brak wolnego łóżka” — NIE jeden ogólny string.
> Walidacja zakresu dat: dodaj `if (dto.From >= dto.To) return BadRequest("From must be before To.");`.

---

## 23. Walidacja

`[ApiController]` automatycznie zwraca **400** z `ModelState`, gdy DTO nie spełnia atrybutów walidacji.

```csharp
public class CreatePcDto
{
    [Required(ErrorMessage = "Name is required")]
    [MaxLength(100)]
    public string Name { get; set; } = null!;

    [Range(0, 1000, ErrorMessage = "Weight must be 0..1000")]
    public double Weight { get; set; }

    [Range(0, int.MaxValue)]
    public int Stock { get; set; }

    [EmailAddress] public string? Email { get; set; }
    [RegularExpression(@"^\d{11}$", ErrorMessage = "PESEL must be 11 digits")]
    public string? Pesel { get; set; }
}
```

Ręczne sprawdzenie (gdy `[ApiController]` wyłączone lub własna logika):
```csharp
if (!ModelState.IsValid) return BadRequest(ModelState);
```

Typowe atrybuty walidacji: `[Required]`, `[MaxLength]`, `[MinLength]`, `[StringLength]`, `[Range]`, `[EmailAddress]`, `[RegularExpression]`, `[Compare]`, `[Phone]`, `[Url]`.

---

## 24. Obsługa błędów

`ControllerBase` ma gotowe helpery zwracające **ProblemDetails** (standard RFC 7807):
```csharp
return NotFound("...");          // 404
return BadRequest("...");        // 400
return Problem("opis", statusCode: 500);
return ValidationProblem(ModelState);
```

Globalny handler (opcjonalnie, .NET 8+):
```csharp
builder.Services.AddProblemDetails();
app.UseExceptionHandler();   // łapie nieobsłużone wyjątki -> 500 ProblemDetails
```

Wzorzec try/catch w endpoincie (gdy chcesz kontrolować 500):
```csharp
try
{
    await _context.SaveChangesAsync();
    return Ok();
}
catch (DbUpdateException ex)
{
    return StatusCode(500, $"Database error: {ex.Message}");
}
```

---

## 25. Minimal API (alternatywa)

Jeśli wolisz/zostaniesz poproszony o Minimal API — całość w `Program.cs`, bez kontrolerów:
```csharp
var app = builder.Build();

app.MapGet("/api/pcs", async (AppDbContext db) =>
    Results.Ok(await db.Pcs.Select(p => new PcGetAllDto { /*...*/ }).ToListAsync()));

app.MapGet("/api/pcs/{id:int}/components", async (int id, AppDbContext db) =>
{
    var pc = await db.Pcs.Where(p => p.Id == id).Select(/*...*/).FirstOrDefaultAsync();
    return pc is null ? Results.NotFound($"PC {id} not found") : Results.Ok(pc);
});

app.MapPost("/api/pcs", async (CreatePcDto dto, AppDbContext db) =>
{
    var pc = new Pc { Name = dto.Name, /*...*/ };
    db.Pcs.Add(pc);
    await db.SaveChangesAsync();
    return Results.Created($"/api/pcs/{pc.Id}", pc);
});

app.MapPut("/api/pcs/{id:int}", async (int id, UpdatePcDto dto, AppDbContext db) =>
{
    var pc = await db.Pcs.FindAsync(id);
    if (pc is null) return Results.NotFound();
    pc.Name = dto.Name; /*...*/
    await db.SaveChangesAsync();
    return Results.Ok(pc);
});

app.MapDelete("/api/pcs/{id:int}", async (int id, AppDbContext db) =>
{
    var pc = await db.Pcs.FindAsync(id);
    if (pc is null) return Results.NotFound();
    db.Pcs.Remove(pc);
    await db.SaveChangesAsync();
    return Results.NoContent();
});

app.Run();
```
Odpowiedniki `Results`: `Results.Ok`, `Results.Created`, `Results.NoContent`, `Results.NotFound`, `Results.BadRequest`, `Results.Problem`.

---

## 26. Transakcje

Gdy kilka operacji musi się wykonać atomowo:
```csharp
using var tx = await _context.Database.BeginTransactionAsync();
try
{
    _context.A.Add(a);
    await _context.SaveChangesAsync();
    _context.B.Add(b);
    await _context.SaveChangesAsync();
    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
    throw;
}
```
> Pojedyncze `SaveChangesAsync()` samo w sobie jest transakcyjne — jawnej transakcji potrzebujesz tylko przy wielu zapisach zależnych.

---

## 27. Częste pułapki

- **Zmieniłeś encję, ale nie zrobiłeś migracji** → baza się nie zmienia. Zawsze: `migrations add` → `database update`.
- **Cykl w serializacji JSON** (encja A → B → A) → zwracaj **DTO**, nie encje. (Awaryjnie: `builder.Services.AddControllers().AddJsonOptions(o => o.JsonSerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles);`)
- **N:M z dodatkową kolumną** (np. `amount`) → potrzebna **jawna encja łącząca** z kluczem złożonym.
- **`HasData` z `DateTime.Now`/`Guid.NewGuid()`** → migracje „dryfują”. Używaj stałych.
- **`HasData` bez jawnych PK** → błąd. Podaj `Id` ręcznie.
- **DELETE blokowane przez FK** → ustaw `OnDelete(DeleteBehavior.Cascade)` w relacji.
- **`Include` + `Select` jednocześnie** → `Include` jest ignorowane gdy robisz projekcję `Select`. Wybierz jedno.
- **LocalDB na macOS/Linux nie działa** → użyj SQL Server w Dockerze lub SQLite.
- **Connection string wbity w `OnConfiguring` po scaffold** → przenieś do `appsettings.json`, scaffolduj z `--no-onconfiguring`.
- **Zapomniany `await`** → zapytanie nie wykona się / zwróci `Task`. Zawsze `await`.
- **`Single` zamiast `First`** gdy może być >1 rekord → wyjątek. Używaj `FirstOrDefault` gdy nie masz pewności unikalności.
- **PUT bez sprawdzenia istnienia** → zwróć 404 zanim zaczniesz modyfikować.
- **Brak `[ApiController]`** → brak automatycznej walidacji i bindowania `[FromBody]`.

---

## 28. Ściąga komend CLI

```bash
# --- PROJEKT ---
dotnet new webapi --use-controllers -n MyApi
dotnet run
dotnet build
dotnet watch run                       # hot reload

# --- PAKIETY ---
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet restore

# --- NARZĘDZIE EF ---
dotnet tool install --global dotnet-ef
dotnet tool update  --global dotnet-ef

# --- CODE FIRST (migracje) ---
dotnet ef migrations add InitialCreate
dotnet ef database update
dotnet ef migrations remove
dotnet ef migrations list
dotnet ef database update <NazwaMigracji>     # cofnięcie do migracji
dotnet ef migrations script -o out.sql

# --- DATABASE FIRST (scaffold) ---
dotnet ef dbcontext scaffold "ConnString" Microsoft.EntityFrameworkCore.SqlServer \
  -o Models -c MyContext --no-onconfiguring --use-database-names -f

# --- DIAGNOSTYKA ---
dotnet ef dbcontext info
dotnet ef dbcontext list
```

PMC (Visual Studio):
```powershell
Add-Migration Init
Update-Database
Remove-Migration
Scaffold-DbContext "ConnString" Microsoft.EntityFrameworkCore.SqlServer -OutputDir Models
```

---

## 29. Gotowe szablony

### A) Minimalny `Program.cs` (kontrolery + EF + serwis)
```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddScoped<IPcService, PcService>();
builder.Services.AddControllers();
builder.Services.AddOpenApi();

var app = builder.Build();
if (app.Environment.IsDevelopment()) app.MapOpenApi();
app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

### B) Szkielet kontrolera CRUD (skopiuj i podmień nazwy)
```csharp
[ApiController]
[Route("api/items")]
public class ItemsController : ControllerBase
{
    private readonly AppDbContext _db;
    public ItemsController(AppDbContext db) => _db = db;

    [HttpGet]
    public async Task<IActionResult> GetAll() =>
        Ok(await _db.Items.Select(x => new ItemDto { /*...*/ }).ToListAsync());

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        var item = await _db.Items.FindAsync(id);
        return item is null ? NotFound($"Item {id} not found") : Ok(item);
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateItemDto dto)
    {
        var item = new Item { /* mapowanie z dto */ };
        _db.Items.Add(item);
        await _db.SaveChangesAsync();
        return CreatedAtAction(nameof(GetById), new { id = item.Id }, item);
    }

    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateItemDto dto)
    {
        var item = await _db.Items.FindAsync(id);
        if (item is null) return NotFound($"Item {id} not found");
        // mapowanie z dto
        await _db.SaveChangesAsync();
        return Ok(item);
    }

    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        var item = await _db.Items.FindAsync(id);
        if (item is null) return NotFound($"Item {id} not found");
        _db.Items.Remove(item);
        await _db.SaveChangesAsync();
        return NoContent();
    }
}
```

### C) Plik `.http` do testowania endpointów (VS / Rider / VS Code REST Client)
```http
@host = https://localhost:5001

### GET all
GET {{host}}/api/pcs

### GET components
GET {{host}}/api/pcs/1/components

### POST
POST {{host}}/api/pcs
Content-Type: application/json

{ "name": "Gaming Beast X", "weight": 12.5, "warranty": 36, "createdAt": "2026-05-08T09:00:00", "stock": 5 }

### PUT
PUT {{host}}/api/pcs/1
Content-Type: application/json

{ "name": "Updated", "weight": 10, "warranty": 24, "createdAt": "2026-05-08T09:00:00", "stock": 3 }

### DELETE
DELETE {{host}}/api/pcs/1

### Search (Zadanie 2)
GET {{host}}/api/patients?search=an
```

---

## Checklisty na kolokwium

### Code First (typ Zadania 1)
- [ ] Pakiety: `EntityFrameworkCore.SqlServer`, `.Design`, `.Tools`.
- [ ] Encje z PK/FK, `[MaxLength]`, `[Required]`, nawigacje.
- [ ] N:M z payloadem → encja łącząca + klucz złożony (Fluent `HasKey(new {...})`).
- [ ] `AppDbContext` z `DbSet`-ami + Fluent w `OnModelCreating`.
- [ ] Connection string w `appsettings.json`, rejestracja w `Program.cs`.
- [ ] `HasData` ≥ 3 rekordy/tabelę, stałe wartości, jawne PK.
- [ ] `migrations add` → `database update`; folder `Migrations/` w repo.
- [ ] DTO osobno dla in/out/create/update; mapowanie w `Select`.
- [ ] Endpointy: GET, GET/{id}/sub, POST→201, PUT→200/404, DELETE→204/404.
- [ ] Wszystko `async/await`.
- [ ] Logika w serwisie, cienki kontroler.

### Database First (typ Zadania 2)
- [ ] Wykonaj `create.sql` (utwórz bazę).
- [ ] `dotnet ef dbcontext scaffold` z `--no-onconfiguring`, `-o Models`, `-c Context`.
- [ ] Connection string do `appsettings.json` + `AddDbContext` w `Program.cs`.
- [ ] GET z opcjonalnym `?search=` → `Contains`/`EF.Functions.Like` z `%...%` po `FirstName`/`LastName`.
- [ ] Złożone zapytanie (wolny zasób w okresie) → negacja nakładania: `!Any(a => a.From < to && a.To > from)`.
- [ ] Czytelne, rozróżnialne komunikaty 404.
- [ ] Format odpowiedzi zgodny z `GET.json` / `POST.json` (DTO!).
```

