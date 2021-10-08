# fluffy-umbrella
Created for the course GitHub: Getting Started

## GitHub Getting Started
In this course, you will ean how to use GitHub successfully.

### The basics
- Understand the use of GitHub
- Create repositories
- Work with Git and GitHub locally
- Create and work with issues
- Create a wiki and a GitHub Page



# Introduktion

Log Wrapper projektet udstiller interfacet `IFMTelemetry<T>`, der benyttes til logning i NTI FM 2.0.

`FMSerilogTelemetry<T>` er en implementation af interfacet, der benytter Serilog biblioteket - <https://serilog.net/>. Serilog understøtter struktureret logning, hvilket for eksempel gør søgning og filtrering nemmere på Application Insights. Den kan sættes op til at logge til et eller flere "Sinks" - se afsnittet om opsætning i appsettings.json nedenfor.

Desuden er der implementeret en factory klasse `FMSerilogTelemetryFactory` der implementerer interfacet `Microsoft.Extensions.Logging.ILoggerFactory`. Den genererer instanser af den ikke-generiske `FMSerilogTelemetry`, der pakkes ind i proxy'en `ProxyFromIFMTelemetryToILogger` og dermed implementerer `Microsoft.Extensions.Logging.ILogger`.

_Bemærk:_ Så snart en instans af `ILoggerFactory` er gjort tilgængelig for dependency injection, aktiveres der en masse "gratis" logning fra f.eks. Microsoft.AspNetCore.Mvc.[...].

Instancerne af de nævnte interfaces og klasser genereres via dependency injection - se nedenfor.

# Opsætning

Disse dele af opsætningen er stadig ikke på plads:

* NuGet pakke? .Net Standard 2.0
* Test/QA/Produktion?
* Opsætning af instrumentation key til Application Insights?

# Konfiguration

## appsettings.json

Herunder et eksempel på hvordan log wrapperen kunne konfigureres i f.eks. appsettings.json.

	{
		"NTI.FM.WriteToApplicationInsights": {
		  "Active": "True",
		  "RestrictedToMinimumLevel": "Debug"
		},
		"NTI.FM.LogPropertyNaming": {
		  "CodeContextPropertyName": "CodeContext",
		  "LocalCorrelationIdPropertyName": "LocalCorrelationId"
		},
		"Serilog": {
		  "MinimumLevel": "Debug",
		  "WriteTo": [
			{
			  "Name": "File",
			  "Args": {
				"Path": "Logs/log_.txt",
				"RollingInterval": "Day",
				"FileSizeLimitBytes": "1000000000",
				"RollOnFileSizeLimit": true,
				"RetainedFileCountLimit": "31",
				"Shared": "True",
				"OutputTemplate": "[{Timestamp:o} {Level:u3}] {Message:lj} [ASM:{AssemblyName},VER:{AssemblyVersion}] [USER:{EnvironmentUserName}] [{ClimateChanges.Year}] [PROP:{Properties}] {NewLine}{Exception}"
			  }
			}
		  ],
		  "Filter": [
			{
			  "Name": "ByExcluding",
			  "Args": {
				"expression": "@l = 'Information' and Contains(@mt, 'filtered')"
			  }
			}
		  ],
		  "Enrich": [ "FromLogContext", "WithMachineName", "WithEnvironmentName", "WithEnvironmentUserName", "WithAssemblyName", "WithAssemblyVersion", "WithThreadId", "WithCorrelationId", "WithCorrelationIdHeader" ],
		  "Destructure": [
			{
			  "Name": "ToMaximumDepth",
			  "Args": { "maximumDestructuringDepth": 4 }
			},
			{
			  "Name": "ToMaximumStringLength",
			  "Args": { "maximumStringLength": 100 }
			},
			{
			  "Name": "ToMaximumCollectionCount",
			  "Args": { "maximumCollectionCount": 10 }
			}
		  ],
		  "Properties": {
			"Application": "WebAPIForLogWrapperTest"
		  }
		},
		
		...
		
		"ApplicationInsights": {
		  "InstrumentationKey": "e92c41b8-5d20-439a-9c62-e8835bff2096",
		  "TelemetryConverter": "Serilog.Sinks.ApplicationInsights.Sinks.ApplicationInsights.TelemetryConverters.TraceTelemetryConverter, Serilog.Sinks.ApplicationInsights"
		}
	}

### Application Insights konfiguration

Den generelle konfiguration placeres i `"ApplicationInsights"` sektionen. Sammen med `services.AddApplicationInsightsTelemetry(Configuration)` (se afsnittet om ConfigureServices nedenfor) giver det en `TelemetryConfiguration`, der anvendes via dependency injection. Det gør os i stand til at genbruge den samme instans, hvilket klart anbefales - se også https://github.com/serilog/serilog-sinks-applicationinsights#configuring.

Konfiguration af Application Insights til log wrapperen placeres i `"NTI.FM.WriteToApplicationInsights"` sektionen. Kun disse konfigurationer er tilgængelige:

* `Active`: Slår logning til Application Insights til/fra.
* `restrictedToMinimumLevel`: Angiver det minimale log level af logs der bliver sendt til Application Insights.

### Konfiguration af øvrige sinks

Øvrige sinks kan konfigureres som normalt i `"Serilog"` sektionen, men kræver typisk at der inkluderes `Serilog.Sinks.[target]` NuGet pakker.

Konfigurationen ovenfor har et eksempel med logning til `"File"`. Læs mere om konfigurationen af Serilogs fil logning her: <https://github.com/serilog/serilog-sinks-file>.

### Generel Serilog konfiguration

Placeres i "Serilog" sektionen. Herunder:

* `"Enrich"`: Angiver hvilke af Serilogs enrichers der skal anvendes, f.eks.:
	* `"FromLogContext"`: Bør altid inkluderes, da den er nødvendig for at kunne påtrykke correlation ids og lignende properties.
	* `"WithCorrelationId"` og `"WithCorrelationIdHeader"`: Serilogs indbyggede understøttelse af correlation id. Correlation id bæres med på tværs af http request/response.
	* `"With..."`: De resterende enrichers påtrykker typisk enkelte properties der kan være nyttige til udrede hvilket miljø der er kørt under.
* `"Destructure"`: Angiver hvordan objekter i de strukturerede logs skal destruktureres til tekst når de er indlejret i log beskeder.

### Øvrig log konfiguration

Da vi dependency injecter en ILoggerFactory, får vi (som nævnt indledningsvist) noget logning forærende. Log levels for dette kan konfigureres i `"Logging"` sektionen, som her:

		"Logging": {
		  "ApplicationInsights": {
			"LogLevel": {
			  "Default": "Debug",
			  "Microsoft": "Error"
			}
		  },
		  "LogLevel": {
			"Default": "Information",
			"Microsoft": "Warning",
			"Microsoft.Hosting.Lifetime": "Information"
		  }
		},

Se også <https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-5.0>.

### Property Naming konfiguration

I `"NTI.FM.LogPropertyNaming"` sektionen kan man ændre property navnet på de log properties, som udskrives direkte af log wrapperen. Det vil dog typisk ikke være nødvendigt - det er ikke navnet, men derimod værdien, af disse properties der er vigtig.

* `"CodeContextPropertyName"`: Navnet på den property der angiver hvilken kontekst en given `FMTelemetry` instans logger fra - f.eks. klassenavnet fra de generisk ´<T>´ instanser, eller det `categoryName` der gives i kald til `ILoggerFactory.CreateLogger`.
	* Default værdi (hvis intet er konfigureret): "CodeContext"
* `"LocalCorrelationIdPropertyName"`: Navnet på den property der bruges til at angive lokale correlation ids (se kaldet til `_telemetryLogger.GetCorrelationIdObject` nedenfor)
	* Default værdi (hvis intet er konfigureret): "LocalCorrelationId"

# Opsætning af logning i C# koden

## Web API controllers

Det følgende er et eksempel på opsætning af logning i et Web API projekt med en controller `WeatherForecastController`. Der opsættes følgende instanser:

* En `FMSerilogTelemetry<T>` log wrapper
* En `FMSerilogTelemetryFactory` (som `ILoggerFactory`), der genererer ILogger instanser ved hjælp af en proxy.

### Tilføjelser til `ConfigureServices` i Startup.cs

Denne linje indsættes i `ConfigureServices` metoden for at gøre de nødvendige instanser af log klasserne tilgængelige via dependency injection.

        public void ConfigureServices(IServiceCollection services)
        {
		
            ...
		
            services.AddFMTelemetry(Configuration);
        }

### Brug af instanserne via dependency injection 

De konkrete instanser af den generisk log klasse `FMSerilogTelemetry<T>` (hvor det også angives hvilken (generisk) type `T` der anvendes - i dette tilfælde `WeatherForecastController`) og af log instans factory'en `FMSerilogTelemetryFactory` kan nu trækkes ud via dependency injection.

Det kunne se sådan ud:

        public WeatherForecastController(IFMTelemetry<WeatherForecastController> telemetryLogger, ILoggerFactory loggerFactory)
        {
            _telemetryLogger = telemetryLogger;
            _loggerFactory = loggerFactory;
        }

Motivation for hver af de udtrukkede instanser:

* `_telemetryLogger`: Vores log wrapper, der benyttes til logning - se eksempler på brugen nedenfor.
* `_loggerFactory`: Bruges til logning fra Entity Framework Core - se nedenfor.

## Brug af `FMSerilogTelemetryFactory` som `ILoggerFactory` til logning fra Entity Framework Core

Først skal dependency injection være på plads som ovenfor, så vi har adgang til `ILoggerFactory` instansen.

Nu kan logning fra Entity Framework Core sættes op fra DbContext subklassen (her `SamuraiContext`) i `OnConfiguring` med et kald til `UseLoggerFactory`:

		public class SamuraiContext : DbContext
		{
			private readonly ILoggerFactory _loggerFactory;

			public DbSet<Samurai> Samurais { get; set; }
			public DbSet<Quote> Quotes { get; set; }
			public DbSet<Battle> Battles { get; set; }

			public SamuraiContext(ILoggerFactory loggerFactory)
			{
				_loggerFactory = loggerFactory;
			}

			protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
			{
				optionsBuilder
					.UseSqlServer("Data Source= (localdb)\\MSSQLLocalDB; Initial Catalog=SamuraiAppData")
					.UseLoggerFactory(_loggerFactory)
					.EnableSensitiveDataLogging()
					;
			}

			...
		}

Når dette er gjort vil der automatisk blive logget f.eks. SQL queries og query execution tider.

## Øvrige afhængigheder af `Microsoft.Exception.Logging.ILogger`

Her skal vi igen så vidt muligt benytte dependency injectede `FMSerilogTelemetryFactory` instans af `ILoggerFactory` interfacet som ovenfor.

Hvis det (mod forventning) bliver nødvendigt at tilgå en enkelt ILogger instans i stedet for at bruge factory'en, så kan den tilføjes til dependency injection poolen i `FMTelemetryWebHostBuilderExtensions.AddFMTelemetry`:

		services.AddSingleton<ILogger, ProxyFromIFMTelemetryToILogger>();

# Logning i praksis i C# koden

I de følgende log eksempler er `_telemetryLogger` en instans af `IFMTelemetry<T>` (se ovenfor om Startup.cs og Dependency injection).

## Eksempler på logning i forskellige levels

		_telemetryLogger.LogDebug("Get() LogDebug called at {Now}", DateTime.Now.ToShortTimeString());
		_telemetryLogger.LogInformation("Get() LogInformation called at {Now}", DateTime.Now.ToShortTimeString());
		_telemetryLogger.LogWarning("Get() LogWarning called at {Now}", DateTime.Now.ToShortTimeString());
		_telemetryLogger.LogError("Get() LogError called at {Now}", DateTime.Now.ToShortTimeString());
		_telemetryLogger.LogCritical("Get() LogCritical called at {Now}", DateTime.Now.ToShortTimeString());

## Eksempel på logning af et objekt med en ikke-triviel struktur

		var climateChanges = new
		{
			TemperatureRise = 1.7,
			Year = 2076,
			Cause = "Coffee Outage 2 - long lasting emission 0.........1.........2.........3.........4.........5.........6.........7.........8.........9.........!",
			WeatherTypes = new[]
			{
				"Sleet", "Hail", "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
			},
			DeepOcean = new
			{
				Name = "Vadehavet",
				DeeperOcean = new
				{
					Name = "Middelhavet",
					EvenDeeperOcean = new
					{
						Name = "Atlanterhavet",
						EvenABitDeeperOcean = new
						{
							Name = "Stillehavet",
							DeepestPlace = new { Name = "Marianergraven" }
						}
					}
				}
			}
		};
		_telemetryLogger.Log(FMLogLevel.Error, "Unhandled climate changes {@ClimateChanges}", climateChanges);

## Eksempel på logning af en Exception

		try
		{
			throw new Exception("TestLogException");
		}
		catch (Exception e)
		{
			_telemetryLogger.LogError(e, "Caught an exception");
		}

## Eksempel på påtrykning af en property

		using (_telemetryLogger.PushProperty("BuildingId", "17"))
		{
			_telemetryLogger.LogWarning("Component type unknown");
		}

Logs indenfor `using` blokken (inklusiv kaldte metoder) vil alle indeholde property'en "BuildingId"/"17".

## Eksempel på påtrykning af et lokalt correlation id

		using (_telemetryLogger.GetCorrelationIdObject())
		{
			_telemetryLogger.LogInformation("Testing correlationId as GetCorrelationIdObject");
			new LogWrapperTesting().SomeTestMethod01();
		}

Logs indenfor `using` blokken (inklusiv kaldte metoder) vil alle blive påtrykt et lokalt correlation id, der *ikke* overskriver correlation id fra Serilog enricheren - de ligger i forskellige properties.
