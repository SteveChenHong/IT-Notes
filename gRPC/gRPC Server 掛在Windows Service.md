Here is an example of the minimal application created by the default template with a few modifications (see code comments)

The project file

```
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <Protobuf Include="Protos\greet.proto" GrpcServices="Server" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Grpc.AspNetCore" Version="2.40.0" />
    <!-- Install additional packages -->
    <PackageReference Include="Grpc.AspNetCore.Server.Reflection" Version="2.40.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting.WindowsServices" Version="6.0.0" />
  </ItemGroup>

</Project>
```
```
The Program.cs

using GrpcService1.Services;
using Microsoft.Extensions.Hosting.WindowsServices;

// Use WebApplicationOptions to set the ContentRootPath
var builder = WebApplication.CreateBuilder(new WebApplicationOptions
{
    Args = args,
    ContentRootPath = WindowsServiceHelpers.IsWindowsService()
        ? AppContext.BaseDirectory
        : default
});

// Set WindowsServiceLifetime
builder.Host.UseWindowsService();

builder.Services.AddGrpc();

// Add reflection services
builder.Services.AddGrpcReflection();

var app = builder.Build();

app.MapGrpcService<GreeterService>();

// Map reflection endpoint
app.MapGrpcReflectionService();

app.Run();
````
Now open cmd in the project folder and execute

dotnet publish
The publish command will produce the exe file (I assume you are working on a Windows machine, otherwise how would you test Windows Service? 😁). On my machine the path to exe is
```
"C:\github\junk\GrpcService1\bin\Debug\net6.0\publish\GrpcService1.exe"
```
Now open cmd as Administrator and run the following command
```
sc.exe create "GrpcService" binpath="C:\github\junk\GrpcService1\bin\Debug\net6.0\publish\GrpcService1.exe"
```
Open Services and start the GrpcService. Now install and run  [grpcui tool](https://github.com/fullstorydev/grpcui "grpcui tool"):
```
grpcui -plaintext localhost:5000
```
The grpcui tool will open the UI where you should be able to see the Greeter serviceenter image description here

Notes:

I Used WebApplication.CreateBuilder(new WebApplicationOptions()) because without that the service won't start. The Windows Event Viewer shows the error:
Description: The process was terminated due to an unhandled exception. Exception Info: System.NotSupportedException: The content root changed from "C:\Windows\system32" to
"C:\github\junk\GrpcService1\bin\Debug\net6.0\publish". Changing the host configuration using WebApplicationBuilder.Host is not supported. Use WebApplication.CreateBuilder(WebApplicationOptions) instead.

Found the solution here.

By default, the application will start on port 5000. If you wish to use another port add --urls argument when creating the service
```
sc.exe create "GrpcService" binpath="C:\github\junk\GrpcService1\bin\Debug\net6.0\publish\GrpcService1.exe --urls \"http://localhost:6276\""
```

reference:https://stackoverflow.com/questions/73323344/how-create-windows-service-from-vs-2022-created-grpc-server/73409556#73409556
