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

`FMSerilogTelemetry<T>` er en implementation af interfacet, der benytter Serilog biblioteket (https://serilog.net/). Den kan sættes op til at logge til et eller flere "Sinks" - se nedenfor.

# Opsætning

## NuGet pakke? .Net Standard 2.0

## appsettings.json

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

TODO: Guide users through getting your code up and running on their own system. In this section you can talk about:
1.	Installation process
2.	Software dependencies
3.	Latest releases
4.	API references

# Logning i praksis








TODO: Explain how other users and developers can contribute to make your code better. 

If you want to learn more about creating good readme files then refer the following [guidelines](https://docs.microsoft.com/en-us/azure/devops/repos/git/create-a-readme?view=azure-devops). You can also seek inspiration from the below readme files:
- [ASP.NET Core](https://github.com/aspnet/Home)
- [Visual Studio Code](https://github.com/Microsoft/vscode)
- [Chakra Core](https://github.com/Microsoft/ChakraCore)
