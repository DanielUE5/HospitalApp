# HospitalApp

A hospital web application built with ASP.NET Core MVC and C#.

The project is being developed incrementally, with small commits for each completed change.

## Current status

The application currently contains the initial MVC scaffold generated with `dotnet new mvc`:

- Home and Privacy pages with a shared Razor layout.
- Static assets, including Bootstrap, jQuery, and client-side validation libraries.
- Default error handling and development launch profiles.

Database integration, authentication, appointment booking, and API endpoints are planned and are not implemented yet.

## Technology

- .NET 10 and ASP.NET Core MVC
- C# with nullable reference types enabled
- Razor views, HTML, CSS, and JavaScript
- Bootstrap

## Getting started

Install the .NET 10 SDK, then run these commands from the project directory:

```sh
dotnet restore
dotnet build
dotnet run --launch-profile http
```

Open <http://localhost:5022> in your browser. Stop the application with `Ctrl+C`.

For local HTTPS development, trust the development certificate and use the HTTPS profile:

```sh
dotnet dev-certs https --trust
dotnet run --launch-profile https
```

The HTTPS profile uses <https://localhost:7042>.

No database or external service configuration is required at this stage.

## Planned development

- [x] Initial MVC application and repository documentation
- [ ] Define requirements, roles, and the data model
- [ ] Add a database with Entity Framework Core and migrations
- [ ] Add registration, sign-in, and patient, doctor, and administrator roles
- [ ] Build department and doctor pages with administration features
- [ ] Implement schedules, appointment booking, and cancellation
- [ ] Expose REST API endpoints with validation and access control
- [ ] Test key workflows, including conflicting appointment requests

The database provider and detailed architecture will be selected when the project requirements are finalized.

## Project structure

```text
Controllers/   MVC controllers
Models/        Application models
Views/         Razor views and shared layouts
wwwroot/       Static files and bundled frontend libraries
Properties/    Local development launch profiles
Program.cs     Application setup and request pipeline
```

## Configuration

Tracked `appsettings` files contain non-secret settings only. Keep future database passwords, API keys, and other credentials in .NET User Secrets for local development or environment variables for deployment.

Build output and local development files are excluded by `.gitignore`.

## License

A project license has not been selected yet. Bundled third-party libraries retain their own licenses in `wwwroot/lib/`.
