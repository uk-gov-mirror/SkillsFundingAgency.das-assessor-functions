# Developer Setup - Assessor Functions
<img src="https://avatars.githubusercontent.com/u/9841374?s=200&v=4" align="right" alt="UK Government logo">

[![Build Status](https://sfa-gov-uk.visualstudio.com/Digital%20Apprenticeship%20Service/_apis/build/status/das-assessor-functions?repoName=SkillsFundingAgency%2Fdas-assessor-functions&branchName=master)](https://sfa-gov-uk.visualstudio.com/Digital%20Apprenticeship%20Service/_build/latest?definitionId=2539&repoName=SkillsFundingAgency%2Fdas-assessor-functions&branchName=master)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=SkillsFundingAgency_das-assessor-functions&metric=alert_status)](https://sonarcloud.io/project/overview?id=SkillsFundingAgency_das-assessor-functions)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg?longCache=true&style=flat-square)](https://en.wikipedia.org/wiki/MIT_License)

## Requirements

In order to run this solution locally you will need:

* .NET 8 SDK
* Azure Functions Core Tools v4
* Azurite
* Azure Storage Explorer
* A local or accessible instance of `SFA.DAS.AssessorService.Application.Api`

## Environment Setup

### local.settings.json

Create a `local.settings.json` file in the `SFA.DAS.Assessor.Functions` project.

Set **Copy to Output Directory** to **Copy always**.

For local development, use the following as a starting point:

```json
{
  "IsEncrypted": false,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "AppName": "das-assessor-functions",
    "EnvironmentName": "LOCAL",
    "ConfigNames": "SFA.DAS.AssessorFunctions",
    "ConfigurationStorageConnectionString": "UseDevelopmentStorage=true",

    "DatabaseMaintenanceTimerSchedule": "0 0 0 * * *",
    "AssessmentsSummaryUpdateSchedule": "0 0 0 * * *",
    "RebuildExternalApiSandboxTimerSchedule": "0 0 0 * * *",
    "EnqueueProvidersTimerSchedule": "0 0 0 * * *",
    "ImportLearnersTimerSchedule": "0 0 0 * * *",
    "OfqualImportTimerSchedule": "0 0 0 * * *",
    "OfsImportTimerSchedule": "0 0 0 * * *",
    "BlobSasTokenGeneratorTimerSchedule": "0 0 0 * * *",
    "BlobStorageSamplesTimerSchedule": "0 0 0 * * *",
    "CertificateDeliveryNotificationTimerSchedule": "0 0 0 * * *",
    "CertificatePrintRequestTimerSchedule": "0 0 0 * * *",
    "CertificatePrintResponseTimerSchedule": "0 0 0 * * *",
    "RefreshProvidersTimerSchedule": "0 0 0 * * *",
    "StandardImportTimerSchedule": "0 0 0 * * *",
    "StandardSummaryUpdateTimerSchedule": "0 0 0 * * *"
  }
}
```

The timer schedule values are required locally because timer trigger attributes are resolved by the Azure Functions host before application configuration is loaded from Azure Table Storage.

The cron expression uses the Azure Functions six-part format:

```text
{second} {minute} {hour} {day} {month} {day-of-week}
```

For example:

```json
"ImportLearnersTimerSchedule": "0 0 0 * * *"
```

runs at midnight every day.

### Running selected functions locally

The full function app contains a number of timer, queue and durable functions. To avoid starting every function in the app, use the function allow list in `local.settings.json`.

For example:

```json
"AzureFunctionsJobHost__functions__0": "RefreshIlrsEnqueueProviders",
"AzureFunctionsJobHost__functions__1": "RefreshIlrsDequeueProviders",
"AzureFunctionsJobHost__functions__2": "ImportLearners"
```

This reduces local startup noise and prevents unrelated functions from running while debugging.

### Azure Table Storage configuration

Add the following configuration row to local Azure Table Storage:

```text
Partition Key: LOCAL
Row Key: SFA.DAS.AssessorFunctions_1.0
```

The configuration data can be copied from:

```text
https://github.com/SkillsFundingAgency/das-employer-config/blob/master/das-assessor-functions/SFA.DAS.AssessorFunctions.json
```

Ensure the local configuration contains values suitable for your machine, including database connection strings and any local/mock API settings.

### Mock Data Collection API data

For local development, mock Data Collection API data can be enabled in the Azure Table Storage configuration.

Example:

```json
"DataCollectionMock": {
  "Enabled": true,
  "AcademicYear": "1920",
  "ProviderCount": 10,
  "LearnerCount": 4,
  "LearningDeliveryCount": 1
}
```

When `DataCollectionMock.Enabled` is `true`, the functions use mock provider, learner and learning delivery data instead of the configured real Data Collection API.

The mock data volume is controlled by:

```text
ProviderCount
LearnerCount
LearningDeliveryCount
```

The academic year used by the mock data is controlled by:

```json
"AcademicYear": "1920"
```

Set `DataCollectionMock.Enabled` to `false` to use the configured real Data Collection API.

### Azure Storage Queues

Some queue-trigger functions require queues to exist in local storage.

For the ILR refresh flow, the queue name is defined in:

```text
src\SFA.DAS.Assessor.Functions\Infrastructure\QueueNames.cs
```

The queue used by `RefreshIlrsDequeueProviders` is currently:

```text
sfa-das-assessor-refresh-ilrs
```

Create required queues in Azurite using Azure Storage Explorer, or create them with the Azure CLI:

```powershell
az storage queue create `
  --name sfa-das-assessor-refresh-ilrs `
  --connection-string "UseDevelopmentStorage=true"
```

## Running

Ensure that an instance of the `SFA.DAS.AssessorService.Application.Api` project is running before starting the functions app.

This can be either local or remote, but it must expose a database connection suitable for local development.

The `BaseAddress` value in the local configuration table must point to the running instance of `SFA.DAS.AssessorService.Application.Api`.

The DC API requires a `ClientSecret`, which can be obtained for a specific test environment from the DC team.

## Triggering timer functions manually

Timer functions can be triggered manually through the Functions admin endpoint.

Start the function host, then call the admin endpoint with a `POST`.

If running from Visual Studio, use the port shown in the Functions host output. For example, if the host is running on port `8081`:

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "http://localhost:8081/admin/functions/RefreshIlrsEnqueueProviders" `
  -ContentType "application/json" `
  -Body '{ "input": "" }'
```

To force a fixed port from Visual Studio, add the port to `Properties\launchSettings.json`.

Example:

```json
{
  "profiles": {
    "SFA.DAS.Assessor.Functions": {
      "commandName": "Project",
      "commandLineArgs": "--port 8081",
      "launchBrowser": false
    }
  }
}
```

Alternatively, start the function app from the command line:

```powershell
func start --port 8081
```

## Refresh ILRs

The ILR refresh flow uses two functions:

```text
RefreshIlrsEnqueueProviders
RefreshIlrsDequeueProviders
```

`RefreshIlrsEnqueueProviders` is a timer-triggered function. It identifies providers that need to be refreshed and adds messages to the ILR refresh queue.

`RefreshIlrsDequeueProviders` is a queue-triggered function. It processes messages from the ILR refresh queue.

The functions connect to the Data Collection API. For local development, the configuration can be updated to use mock Data Collection API data instead.

The current value of `RefreshIlrsLastRunDate` in Assessor Settings controls the date from which providers are considered for refresh.

## Opportunity Finder DataSync

No specific local configuration is required.

## Testing

This codebase includes unit tests and integration tests.

### Unit Tests

The solution contains the following unit test projects:

```text
SFA.DAS.Assessor.Functions.ExternalApis.UnitTests
SFA.DAS.Assessor.Functions.UnitTests
```

The unit tests use:

```text
C#
.NET
FluentAssertions
Moq
NUnit
AutoFixture
```

### Integration Tests

The solution contains the following integration test project:

```text
SFA.DAS.Assessor.Functions.ExternalApis.IntegrationTests
```
