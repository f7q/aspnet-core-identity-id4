# Creating an OAuth Server with aspnet core 2.0 identity and IdentityServer4
## Step by step guide

In this guide we will learn to:
- Create an aspnet core 2.0 identity front end
- Integrate this with IdentityServer4
- Use a 'real' database (Postgres)
- Use a secrets.json file to safely keep credentials out of source control
- Implement 2FA with QR codes and a mobile authenticator app
- Setup some different clients to take advantage of our OAuth server
- Plugin twitter as an authentication provider

### Pre-requisites:
Requires windows 10 or an environment that supports docker compose v2
1. install dotnet core 2.0
2. install docker for | platform |

Note - shell examples are given in powershell, but they all have bash equivalents

### Creating the project environment
1. 
```cmd
mkdir c:\src\project
mkdir c:\src\project\Business.Identity.Host
cd c:\src\project\Business.Identity.Host
```
2. Create some secrets that we can use in our configuration files. To keep these secrets out of source control you use a .gitignore for the .env file you'll create. This needs to be in the same directory as your docker-compose.yml

_.env_
```bash
PGADMIN_DEFAULT_EMAIL=your@email.com
PGADMIN_DEFAULT_PASSWORD=p@ssw0rd!
POSTGRES_USER=identity_user
POSTGRES_PASSWORD=identity_password
POSTGRES_DB=identity
```
The syntax rules should be heeded when creating your .env file! https://docs.docker.com/compose/env-file/

3. configure your development environment by creating a docker-compose.yml file:
_docker-compose.yml_
```yml
version: '3'
services:
  dbadmin:
    image: dpage/pgadmin4
    restart: always
    env_file: .env.example
    ports: 
    - "5555:80"
  identity_db:
    image: postgres:alpine
    restart: always
    env_file: .env.example
    ports:
    - "5432:5432"
```

This creates a couple of containers, one for a web application listening on 5555 to manage your database and run queries, the other is a postgres database listening on 5432.

4. Up your development stack
```powershell
#in the same directory as docker-compose.yml
#run in background
docker-compose up -d 
# OR run in foreground
# docker-compose up 
```

You can now check if everything is running by going to http://localhost:5555 and logging into pgadmin with the credentials you setup.

To determine what IP address pgadmin should connect on, you could use the following to inspect the db container for it's internal IP address:
```powershell
docker ps
#output
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                           NAMES
2ecf91e552e6        dpage/pgadmin4      "/bin/bash /entry.sh"    3 minutes ago       Up 3 minutes        443/tcp, 0.0.0.0:5555->80/tcp   id4aspnetcore_dbadmin_1
217a2da8a303        postgres:alpine     "docker-entrypoint..."   3 minutes ago       Up 3 minutes        0.0.0.0:5432->5432/tcp          id4aspnetcore_identity_db_1
#end output
docker exec -it 217a2da8a303 /sbin/ifconfig eth0 #each OS is different here

#output
eth0      Link encap:Ethernet  HWaddr 02:42:AC:15:00:02
          inet addr:172.21.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:44 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2536 (2.4 KiB)  TX bytes:0 (0.0 B)
```
This shows us we can determine that the IP Address is 172.21.0.2 so to connect pgadmin to our db - we use: 172.21.0.2 port 5432 (refer to pgadmin documentation to help with this)

### Create our identity server application

1. run some setup commands
```cmd
 dotnet new mvc --auth Individual 
 dotnet add package IdentityServer4.AspNetIdentity
 dotnet add package IdentityServer4.EntityFramework
 dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
 dotnet restore
 dotnet build
```
2. Edit Startup.cs
```csharp
//after services.AddMvc();
services.AddIdentityServer().AddAspNetIdentity<ApplicationUser>();

//in Configure method - replace app.UseAuthentication with
app.UseIdentityServer();
```
3. dotnet run to check your work compiles and runs

### Hooking up our 'real' database

1. Edit Startup.cs to use postgres
```csharp
var connectionString = Configuration["IdentityConnection"];
services.AddDbContext<ApplicationDbContext>(
    options => options.UseNpgsql(connectionString)
);
```
2. Edit Startup.cs to use persistent stores in ID4
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
3. Ensure csproj file has the correct tooling to support user secrets
- You'll need to ensure there is a property to the projects property group
```xml
<PropertyGroup>
    <TargetFramework>netcoreapp2.0</TargetFramework>
    <UserSecretsId>Change-This-ID =)</UserSecretsId>
  </PropertyGroup>
```
- Next ensure there is a reference to the secret manager tool
```xml
 <DotNetCliToolReference Include="Microsoft.Extensions.SecretManager.Tools" Version="2.0.0" />
```
- You'll need to ensure you have a secrets file in the correct location
#### Windows: %APPDATA%\microsoft\UserSecrets\{userSecretsId}\secrets.json
#### Linux: ~/.microsoft/usersecrets/{userSecretsId}/secrets.json
#### Mac: ~/.microsoft/usersecrets/{userSecretsId}/secrets.json

Visual studio users can right clikc on the project, and select 'Manage user secrets' to create / edit this file.
- Put your connection string in the secrets.json file:
```json
{
    "IdentityConnection": "User ID=identity_user;Password=Ch@ng3 me =);Host=localhost;Port=5433;Database=identity;Pooling=true;"
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
4. Update your database with the aspnet identity tables
```cmd
dotnet ef database update --context ApplicationDbContext
```
5. Create migrations for the ID4 identity server stores.
```cmd
dotnet ef migrations add initial-id4-persisted-grants --context PersistedGrantDbContext -o Data/Migrations/IdentityServer/PersistedGrantDb

dotnet ef migrations add initial-id4-server-config --context ConfigurationDbContext -o Data/Migrations/IdentityServer/ConfigurationDb

dotnet ef database update --context PersistedGrantDbContext
dotnet ef database update --context ConfigurationDbContext
```

At this point you could run your app and add a user to check that everything on the aspnet identity side is operating as expected.

### Data seeding
This step is entirely optional but you might want to establish a base line of data - especially if you are just getting started and don't have time to make a configuration UI for all of these tables that ID4 and Microsoft just created for you!

You can do this is in a variety of ways but this is an opportunity to demonstrate aspnet core's inbuilt DI framework: 

```csharp
    // In Startup::ConfigureServices, lets add an as yet uncreated class and interface for seeding
    services.AddScoped<IDataSeed, DataSeed>();

    //change the Startup::Configure's signature to look like this:
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, IDataSeed dataSeed) //notice the dataSeed parameter - this will be automatically injected

    //You could now call 
    dataSeed.Init(); 
```
Implementation of a dataseed for ID4 might look like this:
```csharp
public interface IDataSeed
    {
        void Init();
    }

    public class DataSeed : IDataSeed
    {
        readonly ConfigurationDbContext configCtx;
        readonly ApplicationDbContext appCtx;

        public DataSeed(ConfigurationDbContext configCtx, ApplicationDbContext appCtx)
        {
            this.appCtx = appCtx;
            this.configCtx = configCtx;
        }

        public void Init()
        {
            if (!configCtx.Clients.Any()) CreateClients();

            if (!configCtx.ApiResources.Any()) CreateApiResources();
        }

        private void CreateApiResources()
        {
            if (!configCtx.ApiResources.Any())
            {
                var gameApi = new ApiResource("game_api", "Game API");

                configCtx.ApiResources.Add(gameApi.ToEntity());
                configCtx.SaveChanges();
            }
        }

        private void CreateClients()
        {
            var gameClient = new Client()
            {
                ClientId = "game_client",
                AllowedGrantTypes = GrantTypes.ClientCredentials,
                ClientSecrets =
                {
                    new Secret("Ch@ng3 me too!".Sha256())
                },
                AllowedScopes = { "game_api" }
            };
            var e = gameClient.ToEntity();
            e.Id = 1;
            configCtx.Clients.Add(e);
            configCtx.SaveChanges();
        }
    }
```
What I've done here is setup a very basic client and API relationship - which demands that a 'GrantType' of 'ClientCredential' be used. This means that whenever the a client tried to access an API - it will need to provide some credentials.

_Use this an opportunity to berate myself for checking in credentials into source control -- show people how we might inject the Configuration to access it from secrets.json instead_

### Creating an API Resource

```powershell
cd ..
mkdir Business.Game.API
cd Business.Game.API
dotnet new webapi
# if you're using solution files...

dotnet sln {path\to\your.sln} add .\Business.Game.API.csproj 

```

Set this project up on localhost:5001 by adding a launchsettings.json file

```json
{
  "profiles": {
    "Business.Game.API": {
      "commandName": "Project",
      "launchBrowser": true,
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      },
      "applicationUrl": "http://localhost:5001/"
    }
  }
}
```

Modify the values controller.

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Linq;

namespace X.Game.API.Controllers
{
    [Route("api/[controller]")]
    [Authorize]
    public class ValuesController : Controller
    {
        [HttpGet]
        public IActionResult GetValues()
        {
            return Json(User.Claims.Select(c => new { c.Type, c.Value }));
        }
    }
}
```
Bring in the ID4 dependency

```cmd
dotnet add package IdentityServer4.AccessTokenValidation
```

Configure our appsettings.development.json (and prod equiv)
```json
{
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  },
  "Settings": {
    "api_name": "game_api",
    "authority": "http://localhost:5000"
  }
}
```

Setup our startup to use our identity server as an access token provider:

```csharp
public class Startup
    {
        IConfigurationRoot Configuration;

        public Startup(IHostingEnvironment env)
        {
            var builder = new ConfigurationBuilder();
            if(env.IsDevelopment())
            {
                builder.AddUserSecrets<Startup>();
            } else
            {
                builder.AddEnvironmentVariables();
            }

            Configuration = builder.Build();
        }

        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvcCore()
                .AddAuthorization()
                .AddJsonFormatters();

            services.AddAuthentication("Bearer")
                .AddIdentityServerAuthentication(options =>
                {
                    options.Authority = Configuration["settings:authority"];
                    options.RequireHttpsMetadata = false;

                    options.ApiName = Configuration["settings:api_name"];
                });
        }

        public void Configure(IApplicationBuilder app)
        {
            app.UseAuthentication();

            app.UseMvc();
        }
    }
```
*NOTE* Because we've added the user secrets in here as an option - we need to update the csproj and add a UserSecretsId tag.

### Simple API Consumer
First, we'll explore the basic mechanics of requesting an access token

```cmd
mkdir ..\Business.Game.Client
cd ..\Business.Game.Client
dotnet new console
dotnet add package IdentityModel
```

Program.cs
```csharp
static void Main(string[] args)
        {
            Console.WriteLine("Do you want to play a game?");
            var answer = Console.ReadLine();
            if (answer != "Yes") return;

            var disco = DiscoveryClient.GetAsync("http://localhost:5000").Result;
            if (disco.IsError)
            {
                Console.WriteLine(disco.Error);
                return;
            }

            var tokenClient = new TokenClient(disco.TokenEndpoint, "game_client", "Ch@ng3 me too!");
            var tokenResponse = tokenClient.RequestClientCredentialsAsync("game_api").Result;

            if (tokenResponse.IsError)
            {
                Console.WriteLine(tokenResponse.Error);
                return;
            }

            Console.WriteLine(tokenResponse.Json);
        }
```