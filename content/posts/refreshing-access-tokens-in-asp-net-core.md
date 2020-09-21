---
title: "Refreshing Access Tokens in ASP.NET Core"
date: 2020-09-21T00:15:57+02:00
draft: false
tags: ["ASP.NET Core", "Auth0", "Authentication", "OAuth", "Client Credential Flow", "Auth0", "Machine to Machine"]
---

![ClientCredentialFlowAuth0](/img/ClientCredentialFlowAuth0.jpg)

## Client Credential Flow
1. `Client` acquires `Access Token` from `Authorization Server` using
   - ClientId
   - Client Secret
   - Audience
   - GrantType
2. `Client` sends `Access Token` to `Resource Server`
3. `Resource Server` retrieves `jwks.json`
   - `ASP.NET Core` takes care of caching the `jwks.json`. So only the first API request will be slow.
4. `Resource Server` validates `JWT Signature`
5. `Resource Server` checks expiration, permissions and so on
6. `Client` receives `Protected Resources`


## Setup Auth0 for Machine-to-Machine Authentication
[https://auth0.com/docs/applications/set-up-an-application/register-machine-to-machine-applications](https://auth0.com/docs/applications/set-up-an-application/register-machine-to-machine-applications)

## Taking care of Access Token expiration
Per default Auth0 returns access tokens with a lifetime of 24 hours. After that you have to acquire a new token.
Auth0 also has several access token limits/quotas e.g. requests/per second or overall amount of tokens per month. So you definately need to think about access token management.

### Building a Microsoft Dependency Injection container integrated solution by enhancing the extension AddHttpClient

As described in [Issues with the original HttpClient class available in .NET Core](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests#issues-with-the-original-httpclient-class-available-in-net-core), it is recommended to use the provided Microsoft DI container facilities that use `IHttpClientFactory` under the hood. 

My idea is to enhance the already given extension for [Typed Http Clients](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests#how-to-use-typed-clients-with-ihttpclientfactory)

```csharp 
public static IHttpClientBuilder AddHttpClient<TClient, TImplementation>(
    this IServiceCollection services, Action<HttpClient> configureClient)
```

with a new extention
```csharp
public static IHttpClientBuilder AddAuthenticatedHttpClient<TClient, TImplementation>(
    this IServiceCollection serviceCollection, AuthenticationConfig authenticationConfig, 
    Action<HttpClient> configure)
```

in order register an `HttpClient` that takes care of (re-)acquiring access tokens transparently.

**Configuration (e.g. Startup.cs):**
```csharp
var authenticationConfig = new AuthenticationConfig
{
    // Auth0 tenant token endpoint
    Url = ".../oauth/token",
    Parameters = new Parameters
    {
        Audience = "...",
        ClientId = "...",
        ClientSecret = "...",
        GrantType = "client_credentials"
    }
};
serviceCollection.AddAuthenticatedHttpClient<IService, Service>(authenticationConfig,
    x => x.BaseAddress = new Uri("https://apiAddress/"));
```
As a result of returning an `IHttpClientBuilder` like it is already done with `AddHttpClient(..)` we can add additional functionality with `AddHttpMessageHandler` or using `Microsoft.Extensions.DependencyInjection.PollyHttpClientBuilderExtensions.AddPolicyHandler` extension to add Http transient error handling.

```csharp
serviceCollection
    // Takes care of access tokens
    .AddAuthenticatedHttpClient<IService, Service>(authenticationConfig, 
        x => x.BaseAddress = new Uri("https://localhost:5001/"))
    // Takes care of transient Http failures
    .AddPolicyHandler((provider, message) => GetPolicy(provider));

private static IAsyncPolicy<HttpResponseMessage> GetPolicy(IServiceProvider serviceProvider)
{
    var jitterer = new Random();
    var retryWithJitterPolicy = HttpPolicyExtensions
        .HandleTransientHttpError()
        .OrResult(msg => msg.StatusCode == System.Net.HttpStatusCode.NotFound)
        .WaitAndRetryAsync(6, 
            (retryAttempt) => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))
                              + TimeSpan.FromMilliseconds(jitterer.Next(0, 100)),
            (result, timespan, context) =>
            {
                var logger = serviceProvider.GetService<ILogger<Program>>();
                logger.LogWarning($"retrying in {timespan.TotalSeconds}seconds ...");
            });

    return retryWithJitterPolicy;
}

```


**Solution implementation**

```csharp
public static class ServiceCollectionExtensions
{
    public static IHttpClientBuilder AddAuthenticatedHttpClient<TClient, TImplementation>(
        this IServiceCollection serviceCollection,
        AuthenticationConfig authenticationConfig, Action<HttpClient> configure)
        where TImplementation : class, TClient
        where TClient : class
    {
        serviceCollection.AddSingleton(authenticationConfig);
        serviceCollection.AddTransient<AuthenticationHandler>();
        serviceCollection.AddMemoryCache();
        return serviceCollection
            .AddHttpClient<TClient, TImplementation>(configure)
            .AddHttpMessageHandler(provider => provider.GetRequiredService<AuthenticationHandler>());
    }
}

public class AuthenticationHandler : DelegatingHandler
{
    private readonly AuthenticationConfig _authenticationConfig;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IMemoryCache _memoryCache;
    private readonly ILogger<AuthenticationHandler> _logger;
    private static readonly SemaphoreSlim SemaphoreSlim = new SemaphoreSlim(1);
    private const string AccessTokenCacheKey = "AccessTokenCacheKey";
    public AuthenticationHandler(AuthenticationConfig authenticationConfig, IHttpClientFactory httpClientFactory, IMemoryCache memoryCache, ILogger<AuthenticationHandler> logger)
    {
        _authenticationConfig = authenticationConfig;
        _httpClientFactory = httpClientFactory;
        _memoryCache = memoryCache;
        _logger = logger;
    }
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (_memoryCache.TryGetValue(AccessTokenCacheKey, out JwtSecurityToken cachedToken))
        {
            request.Headers.Authorization = new AuthenticationHeaderValue("bearer", cachedToken.RawData);
            return await base.SendAsync(request, cancellationToken);
        }
        await SemaphoreSlim.WaitAsync(cancellationToken);
        try
        {
            if (_memoryCache.TryGetValue(AccessTokenCacheKey, out cachedToken))
            {
                request.Headers.Authorization = new AuthenticationHeaderValue("bearer", cachedToken.RawData);
                return await base.SendAsync(request, cancellationToken);
            }
            _logger.LogInformation("Token expired.");
            var accessToken = await AcquireAccessToken(_authenticationConfig);
            var absoluteExpiration = new DateTimeOffset(accessToken.ValidTo, TimeSpan.Zero);
            _logger.LogInformation($"New token is valid until {absoluteExpiration}.");
            _memoryCache.Set(AccessTokenCacheKey, accessToken, absoluteExpiration.AddSeconds(-5));
            request.Headers.Authorization = new AuthenticationHeaderValue("bearer", accessToken.RawData);
            return await base.SendAsync(request, cancellationToken);
        }
        finally
        {
            SemaphoreSlim.Release();
        }
    }

    private async Task<JwtSecurityToken> AcquireAccessToken(AuthenticationConfig config)
    {
        var httpClient = _httpClientFactory.CreateClient(config.Url);
        httpClient.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
        var body = JsonConvert.SerializeObject(config.Parameters, Formatting.Indented);
        var responseMessage = await httpClient.PostAsync(new Uri(config.Url), new StringContent(body, Encoding.UTF8, "application/json"));
        var content = await responseMessage.Content.ReadAsStringAsync();
        var accessToken = JObject.Parse(content)["access_token"].Value<string>();
        var tokenHandler = new JwtSecurityTokenHandler();
        var jwtSecurityToken = tokenHandler.ReadJwtToken(accessToken);
        return jwtSecurityToken;
    }
}
```

 **Usage:**

```csharp
public interface IService
{
    Task<string> GetTest();
}

public class Service : IService
{
    private readonly HttpClient _client;

    public Service(HttpClient client)
    {
        _client = client;
    }

    public async Task<string> GetTest()
    {
        // we can simply use HttpClient and don't have to take care of any authentication 
        // or retry code
        var httpResponseMessage = await _client.GetAsync("/test");
        var content = await httpResponseMessage.Content.ReadAsStringAsync();
        return content;
    }
}
```