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



# All of the below is test!

# Introduktion

Log Wrapper projektet udstiller interfacet `IFMTelemetry<T>`, der benyttes til logning i NTI FM 2.0.

`FMSerilogTelemetry<T>` er en implementation af interfacet, der benytter Serilog biblioteket <https://serilog.net/>. Serilog understøtter struktureret logning, hvilket for eksempel gør søgning og filtrering nemmere på Application Insights. Den kan sættes op til at logge til et eller flere "Sinks" - se afsnittet om opsætning i appsettings.json nedenfor.

# Opsætning

## NuGet pakke? .Net Standard 2.0

## appsettings.json

Herunder et eksempel på hvordan `FMSerilogTelemetry<T>` kan konfigureres.
* `"Using"`: Angiver hvilke sinks der skal logges til - i dette tilfælde logges der både til Application Insights og til fil.
* `"Enrich"`: Angiver hvilke af Serilogs enrichers der skal anvendes, f.eks.:
	* `"FromLogContext"`: Bør altid inkluderes, da den er nødvendig for at kunne påtrykke correlation ids og lignende properties.
	* `"WithCorrelationId"` og `"WithCorrelationIdHeader"`: Serilogs indbyggede understøttelse af correlation id. Correlation id bæres med på tværs af http request/response.
	* `"With..."`: De resterende enrichers påtrykker typisk enkelte properties der kan være nyttige til udrede hvilket miljø der er kørt under.
* `"Destructure"`: Angiver hvordan objekter i de strukturerede logs skal destruktureres til tekst når de er indlejret i log beskeder.


	  "Serilog": {
		"Using": [ "Serilog.Sinks.ApplicationInsights", "Serilog.Sinks.File" ],
		"MinimumLevel": "Debug",
		"WriteTo": [
		  {
			"Name": "ApplicationInsights",
			"Args": {
			  "InstrumentationKey": "e92c41b8-5d20-439a-9c62-e8835bff2096",
			  "telemetryConverter": "Serilog.Sinks.ApplicationInsights.Sinks.ApplicationInsights.TelemetryConverters.TraceTelemetryConverter, Serilog.Sinks.ApplicationInsights"
			}
		  },
		  {
			"Name": "File",
			"Args": {
			  "path": "Logs/log.txt",
			  "restrictedToMinimumLevel": "Information",
			  "outputTemplate": "[{Timestamp:o} {Level:u3}] {Message:lj} [ASM:{AssemblyName},VER:{AssemblyVersion}] [USER:{EnvironmentUserName}] [{ClimateChanges.Year}] [PROP:{Properties}] {NewLine}{Exception}"
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


## Startup.cs

### `ConfigureServices`

Det følgende er et eksempel på opsætning af en `FMSerilogTelemetry<T>` log wrapper i et Web API. Linjerne skal indsættes i `ConfigureServices` metoden.

        public void ConfigureServices(IServiceCollection services)
        {
		
			...
		
			services.AddApplicationInsightsTelemetry(Configuration);

			services.AddSingleton(typeof(IFMTelemetry<>), typeof(FMSerilogTelemetry<>));

			// To make logging able to read/write correlation ids from/to the http request/response headers
			services.AddHttpContextAccessor();
        }


## Dependency injection

Den konkrete instans af log wrapperen kan genereres via dependency injection, hvor det også angives hvilken (generisk) type der anvendes - i dette tilfælde `WeatherForecastController`.

        public WeatherForecastController(IFMTelemetry<WeatherForecastController> telemetryLogger)
        {
            _telemetryLogger = telemetryLogger;
        }

# Logning i praksis

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
