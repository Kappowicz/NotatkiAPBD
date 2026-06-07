Markdown
# 🚀 .NET 9 & EF Core Web API - Kompendium Wiedzy (Cheat Sheet)

Ten dokument stanowi kompletną ściągawkę z zakresu tworzenia aplikacji Web API przy użyciu platformy ASP.NET Core oraz Entity Framework Core w podejściach Code First i Database First. Zawiera wszystkie niezbędne pojęcia, atrybuty, przykłady implementacji i dobre praktyki wymagane na kolokwiach i w komercyjnych projektach.

---

## 🛠️ 1. Podejścia do Bazy Danych

W Entity Framework Core wyróżniamy dwa główne podejścia do synchronizacji modeli w kodzie z bazą danych:

### Podejście Database First (Scaffolding)
[cite_start]Generowanie klas encji i kontekstu bazy danych na podstawie już istniejącej struktury w bazie[cite: 46].

* **Komenda .NET CLI:**
  ```bash
  dotnet ef dbcontext scaffold "Server=myServerAddress;Database=myDataBase;User Id=myUsername;Password=myPassword;" Microsoft.EntityFrameworkCore.SqlServer -o Models -c AppDbContext
Flaga -o określa folder wyjściowy dla modeli.

Flaga -c określa nazwę generowanej klasy kontekstu (DbContext).

Podejście Code First
Modele aplikacji (klasy w C#) są głównym źródłem prawdy. Na ich podstawie generowana jest struktura bazy danych za pomocą migracji.  
PDF
+ 1


Tworzenie nowej migracji: dotnet ef migrations add InitialCreate   
PDF


Aktualizacja bazy danych: dotnet ef database update   
PDF

Usuwanie ostatniej migracji: dotnet ef migrations remove

📦 2. Modelowanie Danych (Data Annotations vs Fluent API)
Wymagane jest poprawne definiowanie kluczy głównych, obcych, wymaganych pól i typów danych. Można to robić za pomocą atrybutów (Data Annotations) lub konfiguracji w OnModelCreating (Fluent API).  
PDF

Przydatne Atrybuty (Data Annotations)
Atrybut	Opis
[Key]	Oznacza właściwość jako klucz główny (PK).
[Required]	Oznacza pole jako wymagane (NOT NULL w bazie danych).
[MaxLength(255)]	Ogranicza maksymalną długość znaków np. dla stringa.
[Column(TypeName = "varchar(10)")]	Wymusza specyficzny typ danych kolumny w bazie.
[ForeignKey("NazwaWlasciwosci")]	Definiuje klucz obcy (FK) powiązany z właściwością nawigacyjną.
[Table("NazwaTabeli")]	Nadpisuje domyślną nazwę tabeli.
[NotMapped]	Wyklucza właściwość z mapowania do bazy danych (istnieje tylko w kodzie).
Przykład Encji (Code First)
C#
[Table("Patients")]
public class Patient
{
    [Key]
    [Column(TypeName = "char(11)")]
    public string Pesel { get; set; } = null!;

    [Required]
    [MaxLength(50)]
    public string FirstName { get; set; } = null!;

    [Required]
    [MaxLength(100)]
    public string LastName { get; set; } = null!;

    public ICollection<BedAssignment> BedAssignments { get; set; } = new List<BedAssignment>();
}
Konfiguracja za pomocą Fluent API i Seedowanie Danych
Fluent API jest niezbędne do definiowania kluczy złożonych oraz seedowania danych.  
PDF
+ 1

C#
public class AppDbContext : DbContext
{
    public DbSet<Pc> Pcs { get; set; }
    public DbSet<PcComponent> PcComponents { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Klucz złożony dla tabeli łączącej
        modelBuilder.Entity<PcComponent>()
            .HasKey(pc => new { pc.PcId, pc.ComponentCode });

        // Seedowanie danych
        modelBuilder.Entity<Pc>().HasData(
            new Pc { Id = 1, Name = "Gaming Beast X", Weight = 12.5f, Warranty = 36, Stock = 5 }
        );
    }
}
🏛️ 3. Architektura i Dobre Praktyki
Podczas pisania API należy bezwzględnie przestrzegać zasad czystego kodu i separacji warstw.

Zastosuj architekturę wielowarstwową oddzielającą warstwę logiki biznesowej od API/Prezentacji.  
PDF

Nigdy nie udostępniaj encji bazy danych bezpośrednio w endpointach API.  
PDF

Używaj obiektów DTO (Data Transfer Object) do przyjmowania i zwracania danych.  
PDF

Wszelkie operacje bazodanowe muszą być asynchroniczne (async/await).  
PDF

Przykładowy wzorzec DTO
C#
// Wysłanie danych do klienta
public class GetPcResponseDto
{
    public int Id { get; set; }
    public string Name { get; set; } = null!;
}

// Przyjęcie danych z zewnątrz (np. przy tworzeniu)
public class CreatePcRequestDto
{
    [Required]
    [MaxLength(50)]
    public string Name { get; set; } = null!;
    public float Weight { get; set; }
}
🚦 4. Kody Statusów HTTP
API musi odpowiadać poprawnymi kodami w zależności od rezultatu żądania.  
PDF

Kod HTTP	Opis	Kiedy używać w ASP.NET Core
200 OK	
Sukces  
PDF

Przy poprawnym wykonaniu GET, PUT (jeśli zwracamy obiekt). Użyj: return Ok(data);
201 Created	
Utworzono zasób  
PDF

Po poprawnym wykonaniu POST. Użyj: return CreatedAtAction(nameof(Get), new { id = obj.Id }, obj);
204 No Content	
Usunięto/Zmodyfikowano  
PDF

Zazwyczaj po poprawnym DELETE lub PUT. Użyj: return NoContent();
400 Bad Request	
Niepoprawne wejście  
PDF

Gdy walidacja DTO (model state) zawiedzie. Użyj: return BadRequest("Powód");
404 Not Found	
Brak zasobu  
PDF

Gdy encja (np. ID) nie istnieje w bazie. Użyj: return NotFound("Brak zasobu");
500 Error	
Błąd serwera  
PDF

Zazwyczaj rzucany automatycznie w przypadku nieobsłużonych wyjątków.
🔌 5. Kontrolery i Bindowanie Danych (Atrybuty Wejściowe)
ASP.NET Core oferuje różne atrybuty do pobierania danych w zależności od typu żądania.

[FromRoute] - Pobiera wartość ze ścieżki adresu URL (np. api/pcs/{id}).

[FromQuery] - Pobiera wartość z parametrów po znaku zapytania (np. api/pcs?search=intel).

[FromBody] - Pobiera dane z ciała (payload) żądania, najczęściej w formacie JSON (używane w POST/PUT).

Implementacja Kontrolera CRUD
C#
[Route("api/[controller]")]
[ApiController]
public class PcsController : ControllerBase
{
    private readonly AppDbContext _context;

    public PcsController(AppDbContext context)
    {
        _context = context;
    }

    // 1. POBIERANIE Z OPCJONALNYM FILTREM (GET)
    [HttpGet]
    public async Task<IActionResult> GetAllPcs([FromQuery] string? search)
    {
        var query = _context.Pcs.AsQueryable();

        [cite_start]// Użycie funkcji LIKE dla filtrowania [cite: 57]
        if (!string.IsNullOrWhiteSpace(search))
        {
            var searchTerm = $"%{search}%";
            query = query.Where(p => EF.Functions.Like(p.Name, searchTerm));
        }

        var results = await query
            .Select(p => new GetPcResponseDto { Id = p.Id, Name = p.Name })
            .ToListAsync();

        return Ok(results); // Zwraca 200 OK
    }

    // 2. TWORZENIE ZASOBU (POST)
    [HttpPost]
    public async Task<IActionResult> CreatePc([FromBody] CreatePcRequestDto dto)
    {
        var newPc = new Pc { Name = dto.Name, Weight = dto.Weight };
        
        await _context.Pcs.AddAsync(newPc);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetAllPcs), new { id = newPc.Id }, newPc); // Zwraca 201 Created
    }

    // 3. USUWANIE (DELETE)
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeletePc([FromRoute] int id)
    {
        var pc = await _context.Pcs.FindAsync(id);
        if (pc == null)
            return NotFound("Podany komputer nie istnieje."); // Zwraca 404 Not Found

        _context.Pcs.Remove(pc);
        await _context.SaveChangesAsync();

        return NoContent(); // Zwraca 204 No Content
    }
}
🧠 6. Złożone Problemy Logiczne i LINQ
Ładowanie powiązanych danych (Eager Loading)
Jeśli chcesz pobrać encję nadrzędną razem z jej danymi powiązanymi (relacje), używaj Include() i ThenInclude(). Zawsze optymalizuj zapytania rzutując (mapując) na DTO przez Select() zanim pobierzesz dane z bazy za pomocą asynchronicznych metod np. .ToListAsync().

Walidacja Przedziałów Czasowych (np. Rezerwacje Łóżek)
Gdy zadanie wymaga sprawdzenia, czy dany zasób (łóżko, sala) jest wolny we wskazanym okresie.  
PDF

Logika nakładania się dat:
Dwie daty (Istniejąca vs Nowa) nakładają się na siebie, jeśli spełniony jest warunek:
ObecnaRezerwacja.Od < NowaRezerwacja.Do && ObecnaRezerwacja.Do > NowaRezerwacja.Od

C#
// Wyszukiwanie wolnego łóżka
var requestedFrom = request.DateFrom;
var requestedTo = request.DateTo;

var availableBed = await _context.Beds
    .Where(b => b.BedTypeId == request.BedTypeId && b.WardId == request.WardId)
    .Where(b => !b.BedAssignments.Any(assignment => 
        // Sprawdzenie nachodzenia na siebie dat - odrzucamy te łóżka, które mają konflikt
        assignment.From < requestedTo && 
        (assignment.To == null || assignment.To > requestedFrom)))
    .FirstOrDefaultAsync();

if (availableBed == null)
{
    [cite_start]// Komunikat błędu musi być jasny i jednoznaczny [cite: 62]
    return NotFound(new { message = "Brak wolnych łóżek dla wybranego oddziału i typu w danym terminie." });
}


dotnet new tool-manifest
dotnet tool install dotnet-ef
dotnet tool run dotnet-ef
