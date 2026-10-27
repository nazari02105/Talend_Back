# Talend Back End (ETL Web API)

The back end of a small **ETL tool**, a simple Talend-like web app. It is a REST API written in **C# / ASP.NET Core 5** with **SQL Server**.

Users upload **datasets** (CSV files or SQL Server tables) and build **pipelines** that read a dataset, change the data, and write the result into another dataset. The pipelines run with the [ETLBox](https://www.etlbox.net/) library.

The front end (Angular) is a separate project: **[Talend_Front](https://github.com/nazari02105/Talend_Front)**.

## Features

- Sign up, log in and log out (token based)
- **CSV datasets:** upload a CSV file with its column delimiter, row delimiter and header setting; view, download and delete it
- **SQL Server datasets:** save a connection to a table in a SQL Server database; list the tables of a database; view and delete it
- **Pipelines:** create, view and delete pipelines. A pipeline has a **source**, a **destination**, and steps in between:
  - **Filter:** keep rows that match conditions (`>`, `<`, `=`), combined with **and** / **or**
  - **Join:** join with a second dataset on a left key and a right key (always an inner join)
  - **Aggregate:** `sum`, `count`, `min`, `max` or `average` of a column, with group-by columns
- **Run** a pipeline in the background, check its **status** (`Running`, `Finished`, `Failed`, `NotRunning`) and **cancel** it. Each user can run one pipeline at a time.
- Download a pipeline as a **YAML** file, or create one by uploading a YAML file

## Requirements

- **Windows**, because the app connects to SQL Server with Windows login (see below)
- **.NET 5 SDK** (all projects target `net5.0`). .NET 5 is no longer supported by Microsoft, but you can still download it from the .NET website. Visual Studio 2019 or JetBrains Rider also works.
- **SQL Server** installed as the **default instance** on your computer (for example SQL Server Express or Developer edition)

### About the database connection

The database connection string is fixed in the code, in `ETLLibrary/Database/EtlContext.cs`:

```
Server=.; Database=ETLDB; Trusted_Connection=True;
```

This means: SQL Server on **this computer** (default instance), database **`ETLDB`**, and your **Windows account** for login. If your SQL Server is different (for example a named instance like `.\SQLEXPRESS`, or a username and password), change this line before you run the app.

## How to run

All commands are run from the project folder (the folder with `ETLProject.sln`).

1. **Install the Entity Framework tool** (version 5, to match the project):

   ```bash
   dotnet tool install --global dotnet-ef --version 5.0.9
   ```

2. **Create the database.** This creates the `ETLDB` database and its tables (`Users`, `CsvFiles`, `DbConnections`, `DbPipelines`) from the migration in `ETLLibrary/Migrations`:

   ```bash
   dotnet ef database update --project ETLLibrary --startup-project ETLLibrary
   ```

   (In Visual Studio you can do the same with **Package Manager Console**: choose `ETLLibrary` as the default project and as the startup project, then run `Update-Database`.)

3. **Trust the HTTPS development certificate** (once per computer). The front end talks to the back end over `https://localhost:5001`, so the browser must trust this certificate:

   ```bash
   dotnet dev-certs https --trust
   ```

4. **Start the API:**

   ```bash
   cd ETLWebApp
   dotnet run --launch-profile ETLWebApp
   ```

   The API runs on **<https://localhost:5001>** (and <http://localhost:5000>). Open **<https://localhost:5001/swagger>** to see and try all the endpoints.

   In Visual Studio, choose the **ETLWebApp** profile (not *IIS Express*) and press Run. The *IIS Express* profile uses other ports (`44352`), and the front end would not find the API.

5. **Start the front end** ([Talend_Front](https://github.com/nazari02105/Talend_Front)) and open <http://localhost:4200>. The API only accepts browser requests (CORS) from `http://localhost:4200` and `https://localhost:5001`.

**Note:** I could not build or run this project while writing this README, because the package server (NuGet) was not reachable. The steps above come from reading the project files.

## API

Requests that need a login take the token as a **query parameter**: `?token=<token>`. You get the token from `users/login`.

**Users** (`users/`)

| Method | Address | What it does |
|---|---|---|
| POST | `users/signup` | Body: `{"username", "email", "password", "firstName", "lastName"}` |
| POST | `users/login` | Body: `{"username", "password"}`. You can use the username or the email. Returns `{"token": "..."}` |
| POST | `users/logout` | Body: `{"token"}` |
| GET | `users/{token}` | The logged-in user |
| GET | `users/{username}/csvs` | Names of the user's CSV datasets |
| GET | `users/{username}/sqlservers` | Names of the user's SQL Server datasets |
| GET | `users/{username}/datasets` | All datasets, with their type (`csv` or `sqlserver`) |
| GET | `users/{username}/pipelines` | The user's pipelines |

**CSV datasets** (`dataset/csv/`)

| Method | Address | What it does |
|---|---|---|
| POST | `dataset/csv/create?token=` | Form data: `File` (the CSV file) and `Details`, a JSON text: `{"Name", "ColDelimiter", "RowDelimiter", "HasHeader"}` |
| GET | `dataset/csv/{name}?token=` | The content of the CSV |
| GET | `dataset/csv/download/{name}?token=` | Download the CSV file |
| DELETE | `dataset/csv/delete/{name}?token=` | Delete it |

**SQL Server datasets** (`dataset/sqlserver/`)

| Method | Address | What it does |
|---|---|---|
| POST | `dataset/sqlserver/create?token=` | Body: `{"name", "dbName", "dbUsername", "dbPassword", "url", "table"}` |
| GET | `dataset/sqlserver/tables?dbName=&dbUsername=&dbPassword=&url=` | List the tables of a database |
| GET | `dataset/sqlserver/{name}?token=` | The content of the table |
| DELETE | `dataset/sqlserver/delete/{name}?token=` | Delete it |

**Pipelines** (`pipeline/`)

| Method | Address | What it does |
|---|---|---|
| POST | `pipeline/create?token=` | Body: `{"name", "content"}`. `content` is the pipeline as JSON text (see below) |
| GET | `pipeline/{name}?token=` | The pipeline content |
| DELETE | `pipeline/delete/{name}?token=` | Delete it |
| POST | `pipeline/run?token=` | Body: `{"name"}`. Starts the pipeline |
| GET | `pipeline/{name}/status?token=` | `Running`, `Finished`, `Failed` or `NotRunning` |
| POST | `pipeline/cancel?token=` | Cancel your running pipeline |
| GET | `pipeline/yml/download/{name}?token=` | Download the pipeline as YAML |
| POST | `pipeline/yml/upload?token=` | Form data: `File` (YAML file) and `Name` |

### Pipeline format

A pipeline is saved as JSON with `nodes` and `edges` (the graph drawn in the front end). Each node has an `id` and a `data.type`:

- `common` with id `source` or `destination`: `data.name` is the name of a dataset
- `filter`: `data.filterTree` is a small tree of `and` / `or` / `condition` nodes; each condition has `column`, `operation` and `value`
- `join`: `data.dataset` (the second dataset), `data.leftKey`, `data.rightKey`
- `aggregate`: `data.operation`, `data.column`, `data.outputName`, `data.groupColumns`

The front end builds this JSON for you. The **destination must be an existing dataset**: the result is written into that CSV file or SQL Server table.

## Where data is stored

- Users, dataset info and pipelines are saved in the `ETLDB` database.
- Uploaded CSV files are saved in a `csvFiles/<username>/` folder, inside the folder the API is started from (`ETLWebApp/` if you follow the steps above).
- Login tokens are kept **in memory**, so everyone is logged out when the API restarts.

## Known issues

- **Passwords are stored as plain text** in the database, and `GET users/{token}` returns the user, including the password.
- **The `users/{username}/...` addresses don't need a token**, so anyone can see the dataset and pipeline names of any user.
- **The database connection string is fixed in the code** (not in `appsettings.json`).
- **SQL Server datasets use Windows login.** The connection string for a user's database includes the username and password, but also `Integrated Security=True`, so SQL Server uses the Windows account of the computer running the API instead.
- **Join is always an inner join**, even though the code has other join types.
- **The tests can't run as they are.** `TestETLLibrary/PipelineTests.cs` is marked as "just for my local testing". It needs `demo.csv`, `join_1.csv` and `join_2.csv`, which are not in the repo.

## Project structure

| Folder | What it is |
|---|---|
| `ETLWebApp/` | The web API: controllers, request models, `Startup.cs` (CORS, Swagger), `Properties/launchSettings.json` (ports) |
| `ETLLibrary/Database/` | Database context, tables (models), and managers for CSV files, SQL Server datasets and pipelines |
| `ETLLibrary/Authentication/` | Login, logout and tokens |
| `ETLLibrary/Migrations/` | Database migration that creates the tables |
| `ETLLibrary/Model/Pipeline/` | Pipeline nodes (source, filter, join, aggregate, destination) and the JSON-to-pipeline convertor |
| `ETLLibrary/Processing/` | Runs pipelines in the background and keeps their status |
| `ETLLibrary/Convertors/` | JSON ↔ YAML convertors |
| `TestETLLibrary/` | A test project for local testing (see known issues) |

## Tech

- ASP.NET Core 5 (Web API), Swagger (Swashbuckle 5.6)
- Entity Framework Core 5 with SQL Server
- ETLBox 2.4 (CSV and SQL Server connectors)
- YamlDotNet
