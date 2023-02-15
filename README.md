# MINIMAL API’s
> Menos é mais?
## O que são APIs Minimal
“APIs mínimas são arquitetadas para criar APIs HTTP com dependências mínimas. Eles são ideais para microsserviços e aplicativos que desejam incluir apenas os arquivos, recursos e dependências mínimas no ASP.NET Core.”

## O Padrão MVC
### Model - View - Controller
![image](https://user-images.githubusercontent.com/58392536/219026512-8c1d69e1-0936-4040-8efb-3d00e6df2a3f.png)

## ASP.NET MVC API Architecture
![image](https://user-images.githubusercontent.com/58392536/219026591-4acbdc5b-48e6-4b54-b2ac-f8dd00caf59f.png)

O que isso tem a ver com APIs mínimas?
# Isso significa*...
* Não suporta validação de modelo
* Não suporte para JSONPatch
* Não suporta filtros
* Não suporta vinculação de modelo personalizado (suporte para IModelBinder)
![image](https://user-images.githubusercontent.com/58392536/219027337-11e0a811-717b-4884-adef-8a11c9e7ea01.png)

## FLUXO
### Let’s Code!
```console
dotnet new webapi -minimal -n SixMinAPI
```

```console
code -r SixMinAPI
```

```console
dotnet add package Microsoft.EntityFrameworkCore
```

```console
dotnet add package Microsoft.EntityFrameworkCore.Design
```

```console
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

```console
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
```

```console
dotnet user-secrets init
```

## Docker-Compose
Utilizado para subir o servidor do SQL Express
```console
docker --version
```

```console
docker ps
```

docker-compose.yaml
```c#
version: '2.13.0'
services:
  sqlserer:
    image: "mcr.microsoft.com/mssql/server:2019-latest"
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "Pa55W0rd!"
      MSSQL_PID: "Express"
    ports:
      - "1433:1433"
```

```console
docker-compose up -d
```

```console
docker-compose stop
```

## MODELS 
```c#
using System.ComponentModel.DataAnnotations;

namespace SixMinAPI.Models
{
    public class Command
    {
        [Key]
        public int Id { get; set; }
        [Required]
        public string? HowTo { get; set; }
        [Required]
        [MaxLength(5)]
        public string? Platform { get; set; }
        [Required]
        public string? CommandLine { get; set; }
    }
}
```

## DTOs
 - Modelos de Transferência 
```c#
using System.ComponentModel.DataAnnotations;

namespace SixMinAPI.Dtos
{
    public class CommandCreateDto
    {
        [Required]
        public string? HowTo { get; set; }
        [Required]
        [MaxLength(5)]
        public string? Platform { get; set; }
        [Required]
        public string? CommandLine { get; set; }
    }
}
```

```c#
namespace SixMinAPI.Dtos
{
    public class CommandReadDto
    {
        public int Id { get; set; }
        public string? HowTo { get; set; }
        public string? Platform { get; set; }
        public string? CommandLine { get; set; }
    }
}
```

```c#
using System.ComponentModel.DataAnnotations;

namespace SixMinAPI.Dtos
{
    public class CommandUpdateDto
    {
        [Required]
        public string? HowTo { get; set; }
        [Required]
        [MaxLength(5)]
        public string? Platform { get; set; }
        [Required]
        public string? CommandLine { get; set; }
    }
}
```

## Data Access (DB Context)
Configuração do banco (usuario, senha sql) 
```c#
using Microsoft.EntityFrameworkCore;
using SixMinAPI.Models;

namespace SixMinAPI.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
        {
           
        }
        public DbSet<Command> Commands => Set<Command>();
    }
}
```

```c#
appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings":
  {
    "SQLDbConnection": "Server=localhost,1433;Initial Catalog=CommandDb;TrustServerCertificate=true"
  }
}
```

## UserSecrets
```console
dotnet user-secrets set “UserId” “sa”
```

```console
>> dotnet user-secrets set "Password" "Pa55W0rd!" 
```

Em Program
```c#
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using SixMinAPI.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var sqlConBuilder = new SqlConnectionStringBuilder();

sqlConBuilder.ConnectionString = builder.Configuration.GetConnectionString("SQLDbConnection");
sqlConBuilder.UserID = builder.Configuration["UserId"];
sqlConBuilder.Password = builder.Configuration["Password"];

builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(sqlConBuilder.ConnectionString));

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.Run();
```

## Migrations
```console
dotnet ef migrations add initialmigration
```

Caso precise: ```console dotnet tool install –global dotnet-ef ```
```console
dotnet ef database update
```

Nome do Banco de Dados que será criado
```c#
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings":
  {
    "SQLDbConnection": "Server=localhost,1433;Initial Catalog=CommandDb;TrustServerCertificate=true"
  }
}
```

## Repository
### Interface
```c#
using SixMinAPI.Models;

namespace SixMinAPI.Data
{
    public interface ICommandRepo
    {
        Task SaveChanges();
        Task<Command?> GetCommandById(int id);
        Task<IEnumerable<Command>> GetAllCommands();
        Task CreateCommand(Command cmd);
        void DeleteCommand(Command cmd);
    }
}
```

### Implementação
```c#
using Microsoft.EntityFrameworkCore;
using SixMinAPI.Models;

namespace SixMinAPI.Data
{
    public class CommandRepo : ICommandRepo
    {
        private readonly AppDbContext _context;


        public CommandRepo(AppDbContext context)
        {
            _context = context;
        }
        public async Task CreateCommand(Command cmd)
        {
            if (cmd == null)
            {
                throw new ArgumentNullException(nameof(cmd));
            }
            await _context.AddAsync(cmd);
        }


        public void DeleteCommand(Command cmd)
        {
            if (cmd == null)
            {
                throw new ArgumentNullException(nameof(cmd));
            }
            _context.Commands.Remove(cmd);
        }


        public async Task<IEnumerable<Command>> GetAllCommands()
        {
            return await _context.Commands!.ToListAsync();
        }


        public async Task<Command?> GetCommandById(int id)
        {
            return await _context.Commands.FirstOrDefaultAsync(c => c.Id == id);
        }


        public async Task SaveChanges()
        {
            await _context.SaveChangesAsync();
        }
    }
}
```

## Program
```c#
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using SixMinAPI.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var sqlConBuilder = new SqlConnectionStringBuilder();

sqlConBuilder.ConnectionString = builder.Configuration.GetConnectionString("SQLDbConnection");
sqlConBuilder.UserID = builder.Configuration["UserId"];
sqlConBuilder.Password = builder.Configuration["Password"];

builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(sqlConBuilder.ConnectionString));
builder.Services.AddScoped<ICommandRepo, CommandRepo>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.Run();
```

## Automapper (Profiles) - mapeamento models x dtos
```c#
using AutoMapper;
using SixMinAPI.Dtos;
using SixMinAPI.Models;

namespace SixMinAPI.Profiles
{
    public class CommandsProfile : Profile
    {
        public CommandsProfile()
        {
            // Source -> Target
            CreateMap<Command, CommandReadDto>();
            CreateMap<CommandCreateDto, Command>();
            CreateMap<CommandUpdateDto, Command>();
        }
    }
}
```

## Program
```c#
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using SixMinAPI.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var sqlConBuilder = new SqlConnectionStringBuilder();

sqlConBuilder.ConnectionString = builder.Configuration.GetConnectionString("SQLDbConnection");
sqlConBuilder.UserID = builder.Configuration["UserId"];
sqlConBuilder.Password = builder.Configuration["Password"];

builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(sqlConBuilder.ConnectionString));
builder.Services.AddScoped<ICommandRepo, CommandRepo>();
builder.Services.AddAutoMapper(AppDomain.CurrentDomain.GetAssemblies());

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.Run();
```

## Minimal API
```c#
using AutoMapper;
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using SixMinAPI.Data;
using SixMinAPI.Dtos;
using SixMinAPI.Models;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var sqlConBuilder = new SqlConnectionStringBuilder();

sqlConBuilder.ConnectionString = builder.Configuration.GetConnectionString("SQLDbConnection");
sqlConBuilder.UserID = builder.Configuration["UserId"];
sqlConBuilder.Password = builder.Configuration["Password"];

builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(sqlConBuilder.ConnectionString));
builder.Services.AddScoped<ICommandRepo, CommandRepo>();
builder.Services.AddAutoMapper(AppDomain.CurrentDomain.GetAssemblies());

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.MapGet("api/v1/commands", async (ICommandRepo repo, IMapper mapper) => {
    var commands = await repo.GetAllCommands();
    return Results.Ok(mapper.Map<IEnumerable<CommandReadDto>>(commands));
});

app.MapGet("api/v1/commands/{id}", async (ICommandRepo repo, IMapper mapper, int id) => {
    var command = await repo.GetCommandById(id);
    if (command != null)
    {
        return Results.Ok(mapper.Map<CommandReadDto>(command));
    }
    return Results.NotFound();
});

app.MapPost("api/v1/commands", async (ICommandRepo repo, IMapper mapper, CommandCreateDto cmdCreateDto) => {
    var commandModel = mapper.Map<Command>(cmdCreateDto);

    await repo.CreateCommand(commandModel);
    await repo.SaveChangesAsync();

    var cmdReadDto = mapper.Map<CommandReadDto>(commandModel);

    return Results.Created($"api/v1/commands/{cmdReadDto.Id}", cmdReadDto);
});

app.MapPut("api/v1/commands/{id}", async (ICommandRepo repo, IMapper mapper, int id, CommandUpdateDto cmdUpdateDto) => {
    var command = await repo.GetCommandById(id);
    if (command == null)
    {
        return Results.NotFound();
    }

    mapper.Map(cmdUpdateDto, command);

    await repo.SaveChangesAsync();

    return Results.NoContent();
});

app.MapDelete("api/v1/commands/{id}", async (ICommandRepo repo, IMapper mapper, int id) => {
    var command = await repo.GetCommandById(id);
    if (command == null)
    {
        return Results.NotFound();
    }

    repo.DeleteCommand(command);
   
    await repo.SaveChangesAsync();

    return Results.NoContent();
});

app.Run();
```

