# Creating an OAuth Server with aspnet core 2.0 identity and IdentityServer4
## Step by step guide

In this guide we will learn to:
- Use aspnet core 2.0 identity
- Integrate IdentityServer4
- Use a 'real' database (Postgres)
- Use a secrets.json file to safely keep credentials out of source control
- Implement 2FA with QR codes and a mobile authenticator app
- Setup some different clients to take advantage of our OAuth server
- Plugin twitter as an authentication provider

### Pre-requisites:
Requires windows 10 or an environment that supports docker compose v2
1. install dotnet core 2.0
2. install docker for | platform |


### Creating the project
1. 
```cmd
mkdir c:\src\project
mkdir c:\src\project\Business.Identity.Host
cd c:\src\project\Business.Identity.Host
```
4. run some setup commands
```cmd
 dotnet new mvc --auth Individual 
 dotnet add package IdentityServer4.AspNetIdentity
 dotnet add package IdentityServer4.EntityFramework
 dotnet restore
 dotnet build
```
5. Edit Startup.cs
```csharp
//after services.AddMvc();
services.AddIdentityServer().AddAspNetIdentity<ApplicationUser>();
```
6. dotnet run to check your work compiles and runs

### Hooking up a 'real' database
1. Using postgresql
```
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```
2. Edit Startup.cs to use postgres
```csharp
var connectionString = Configuration["IdentityConnection"];
services.AddDbContext<ApplicationDbContext>(
    options => options.UseNpgsql(connectionString)
);
```
3. Edit Startup.cs to use persistent stores in ID4
```csharp
string migrationsAssembly = Assembly.GetExecutingAssembly().GetName().Name;
services
    .AddIdentityServer()
    .AddConfigurationStore(options =>
    {
        options.ConfigureDbContext = builder =>
            builder.UseNpgsql(connectionString, b => b.MigrationsAssembly(migrationsAssembly));
    })
    .AddOperationalStore(options =>
    {
        options.ConfigureDbContext = builder =>
            builder.UseNpgsql(connectionString, b => b.MigrationsAssembly(migrationsAssembly));

        options.EnableTokenCleanup = true;
        options.TokenCleanupInterval = 30;
    })
    .AddAspNetIdentity<ApplicationUser>();
```
4. Ensure csproj file has the correct tooling to support user secrets
- You'll need to add a property to the projects property group
```xml
<PropertyGroup>
    <TargetFramework>netcoreapp2.0</TargetFramework>
    <UserSecretsId>User-Secret-ID</UserSecretsId>
  </PropertyGroup>
```
- Next add a reference to the secret manager tool
```xml
 <DotNetCliToolReference Include="Microsoft.Extensions.SecretManager.Tools" Version="2.0.0" />
```
- Restore packages
```cmd
dotnet restore
```
- You'll need to ensure you have a secrets file in the correct location
#### Windows: %APPDATA%\microsoft\UserSecrets\{userSecretsId}\secrets.json
#### Linux: ~/.microsoft/usersecrets/{userSecretsId}/secrets.json
#### Mac: ~/.microsoft/usersecrets/{userSecretsId}/secrets.json
- Put your connection string in the secrets.json file:
```json
{
    "IdentityConnection": "User ID=identity_user;Password=identity_password;Host=localhost;Port=5433;Database=identity;Pooling=true;"
}
```
- Ensure your Startup::ctor uses secrets in development
```csharp
public Startup(IHostingEnvironment env)
{
    var builder = new ConfigurationBuilder();
    if (env.IsDevelopment())
    {
        builder.AddUserSecrets<Startup>();
    } else {
        builder.AddEnvironmentVariables();
    }

    Configuration = builder.Build();
}
```
5. Create migrations for the identity server stores.
```cmd
dotnet ef migrations add initial-id4-persisted-grants --context PersistedGrantDbContext -o Data/Migrations/IdentityServer/PersistedGrantDb

dotnet ef migrations add initial-id4-server-config --context ConfigurationDbContext -o Data/Migrations/IdentityServer/ConfigurationDb

dotnet ef database update --context PersistedGrantDbContext
dotnet ef database update --context ConfigurationDbContext
```


