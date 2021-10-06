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

Log Wrapper projektet udstiller interfacet `IFMTelemetry<T>`, der benyttes til logning, event tracking og lignende i NTI FM 2.0.

`FMSerilogTelemetry<T>` er en implementation af interfacet, der benytter Serilog biblioteket - <https://serilog.net/>. Serilog understøtter struktureret logning, hvilket for eksempel gør søgning og filtrering nemmere på Application Insights. Den kan sættes op til at logge til et eller flere "Sinks" - se afsnittet om opsætning i appsettings.json nedenfor.

## Øvrig telemetri

Hvis log wrapperen er sat op til at logge til Application Insights, så vil øvrig telemetri (f.eks. event tracking) blive registreret via application insights API'et: <https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics>

Er log wrapperen derimod ikke sat op til at logge til Application Insights, så vil øvrig telemetri blive logget som almindelige logs med log level `Information` - se f.eks. `FMSerilogTelemetry.TrackEvent`.

# Opsætning

## NuGet pakke? .Net Standard 2.0

## Test/QA/Produktion?

### Application Insights opsætning og instrumentation key

## appsettings.json

Herunder et eksempel på hvordan `FMSerilogTelemetry<T>` kan konfigureres i f.eks. appsettings.json.

	{
		"NTI.FM.WriteToApplicationInsights": {
		  "Active": "True",
		  "restrictedToMinimumLevel": "Information"
		},
		"Serilog": {
		  "MinimumLevel": "Debug",
		  "WriteTo": [
			{
			  "Name": "File",
			  "Args": {
				"path": "Logs/log.txt",
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
		
		...
		
		"ApplicationInsights": {
		  "InstrumentationKey": "e92c41b8-5d20-439a-9c62-e8835bff2096",
		  "telemetryConverter": "Serilog.Sinks.ApplicationInsights.Sinks.ApplicationInsights.TelemetryConverters.TraceTelemetryConverter, Serilog.Sinks.ApplicationInsights"
		}
	}

### Application Insights opsætning

Den generelle opsætning placeres i `"ApplicationInsights"` sektionen. Sammen med `services.AddApplicationInsightsTelemetry(Configuration)` (se afsnittet om ConfigureServices nedenfor) giver det en `TelemetryConfiguration`, der anvendes via dependency injection. Det gør os i stand til at genbruge den samme instans, hvilket klart anbefales - se også https://github.com/serilog/serilog-sinks-applicationinsights#configuring.

Opsætning af Application Insights til log wrapperen placeres i `"NTI.FM.WriteToApplicationInsights"` sektionen. Kun disse konfigurationer er tilgængelige:

* `Active`: Slår logning til Application Insights til/fra.
* `restrictedToMinimumLevel`: Angiver det minimale log level af logs der bliver sendt til Application Insights.

### Opsætning af øvrige sinks

Øvrige sinks kan konfigureres som normalt i `"Serilog"` sektionen, men kræver typisk at der inkluderes `Serilog.Sinks.[target]` NuGet pakker. Opsætningen ovenfor har et eksempel med logning til `"File"`.

### Generel Serilog opsætning

Placeres i "Serilog" sektionen. Herunder:

* `"Enrich"`: Angiver hvilke af Serilogs enrichers der skal anvendes, f.eks.:
	* `"FromLogContext"`: Bør altid inkluderes, da den er nødvendig for at kunne påtrykke correlation ids og lignende properties.
	* `"WithCorrelationId"` og `"WithCorrelationIdHeader"`: Serilogs indbyggede understøttelse af correlation id. Correlation id bæres med på tværs af http request/response.
	* `"With..."`: De resterende enrichers påtrykker typisk enkelte properties der kan være nyttige til udrede hvilket miljø der er kørt under.
* `"Destructure"`: Angiver hvordan objekter i de strukturerede logs skal destruktureres til tekst når de er indlejret i log beskeder.

## Startup.cs

Det følgende er et eksempel på opsætning af en `FMSerilogTelemetry<T>` log wrapper i et Web API.

### Tilføjelser til `ConfigureServices`

Disse linjer indsættes i `ConfigureServices` metoden.

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

        public WeatherForecastController(IFMTelemetry<WeatherForecastController> telemetryLogger, TelemetryConfiguration telemetryConfiguration)
        {
            _telemetryLogger = telemetryLogger;
            _telemetryConfiguration = telemetryConfiguration;
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

## Eksempel på tracking af et event

		_telemetryLogger.TrackEvent("Test TrackEvent basic");
		_telemetryLogger.TrackEvent("Test TrackEvent properties", 
			new Dictionary<string, string>(){ {"PropName1", "PropValue1"}, { "PropName2", "PropValue2" } }, 
			null);
		_telemetryLogger.TrackEvent("Test TrackEvent metrics", 
			null,
			new Dictionary<string, double>(){ {"MetricName1", 1.0 }, { "MetricName2", 7.0 } });
		_telemetryLogger.TrackEvent("Test TrackEvent both properties and metrics",
			new Dictionary<string, string>() { { "PropName1", "PropValue1" }, { "PropName2", "PropValue2" } },
			new Dictionary<string, double>(){ {"MetricName1", 3.0 }, { "MetricName2", 11.0 } });
