# Caching in .NET 8.0

Boosting Your ASP.NET Core App with Multi-Layer Caching: A Comprehensive Guide

https://medium.com/@michaelmaurice410/boosting-your-asp-net-core-app-with-multi-layer-caching-a-comprehensive-guide-63cc5267425a

# Multi-layer Cache:
+ In-Memory Cache (fastest): Store data locally in the server’s memory for ultra-fast access.
+ Redis Cache (distributed): Store data in a distributed cache like Redis, which is shared across different instances of your application.

## CacheManager.cs
```
public class CacheManager(IEnumerable<ICacheProvider> cacheProviders, ILogger<CacheManager> logger)
{
    public async Task<T> GetOrAddAsync<T>(string key, Func<Task<T>> getFromDbFunction, TimeSpan expiry)
    {
        foreach (var cacheProvider in cacheProviders)
        {
            logger.LogInformation("Try to get value from cache: {key} in {cacheProvider}", key, cacheProvider.GetType().Name);
            var cachedValue = await cacheProvider.GetAsync<T>(key);
if (cachedValue != null)
            {
                logger.LogInformation("*****> HIT for {key} in {cacheProvider}", key, cacheProvider.GetType().Name);
                return cachedValue;
            }
            else
                logger.LogInformation("----->Cache wasn't hit for {key} in {cacheProvider}", key, cacheProvider.GetType().Name);
        }
        logger.LogInformation("====> Not found in any cache: {key}", key);
        var result = await getFromDbFunction();
        var providerList = cacheProviders.ToList();
        for (int i = 0; i < providerList.Count; i++)
        {
            var expirySeconds = (i * 2 == 0 ? 1 : i * 2) * expiry.TotalSeconds;
            await providerList[i].SaveAsync(key, result, TimeSpan.FromSeconds(expirySeconds));
        }
        return result;
    }
    public async Task DeleteAsync(string key)
    {
        foreach (var cacheProvider in cacheProviders)
        {
            await cacheProvider.DeleteAsync(key);
        }
    }
}
```

### InMemoryCacheProvider.cs
```
public class InMemoryCacheProvider(IMemoryCache memCache, ILogger<InMemoryCacheProvider> logger) : ICacheProvider
{
    public Task SaveAsync<T>(string key, T value, TimeSpan expiry)
    {
        var options = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiry
        };
memCache.Set(key, value, options);
        logger.LogInformation("Data saved to in-memory cache, key: {key} expiry:{expiry}", key, expiry.TotalSeconds);
        return Task.CompletedTask;
    }
    public Task<T?> GetAsync<T>(string key)
    {
        memCache.TryGetValue(key, out T? cacheResult);
        return Task.FromResult(cacheResult);
    }
    public Task<bool> DeleteAsync(string key)
    {
        memCache.Remove(key);
        return Task.FromResult(true);
    }
}
```


### RedisCacheProvider.cs
```
public class RedisCacheProvider(IDatabase Redis, ILogger<RedisCacheProvider> logger) : ICacheProvider
{
    public async Task SaveAsync<T>(string key, T value, TimeSpan expiry)
    {
        await Redis.StringSetAsync(key, JsonSerializer.Serialize(value), expiry);
        logger.LogInformation("Data saved to redis cache, key: {key} expiry:{expiry}", key, expiry.TotalSeconds);
    }
public async Task<T?> GetAsync<T>(string key)
    {
        var data = await Redis.StringGetAsync(key);
        if (data.IsNullOrEmpty)
        {
            return default;
        }
        return JsonSerializer.Deserialize<T>(data!);
    }
    public Task<bool> DeleteAsync(string key)
    {
        return Redis.KeyDeleteAsync(key);
    }
}
```
## Wiring It All Up
```
public static class ServiceCollectionExtension
{
    public static void AddCacheServices(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddMemoryCache();
services.AddScoped<ICacheProvider, InMemoryCacheProvider>();
        services.AddScoped<ICacheProvider, RedisCacheProvider>();
        services.AddScoped<CacheManager>();
        var redisConnection = configuration.GetConnectionString("Redis");
        var redisOptions = ConfigurationOptions.Parse(redisConnection!);
        services.AddSingleton<IConnectionMultiplexer>(ConnectionMultiplexer.Connect(redisOptions));
        services.AddScoped<IDatabase>(provider => provider.GetRequiredService<IConnectionMultiplexer>().GetDatabase());
    }
}
```

## Example
```
public static void MapWeatherApi(this WebApplication app)
{
    app.MapGet("/weatherforecast", async ([FromServices] WeatherService service,
        [FromServices] CacheManager cacheManager) =>
    {
        var result = await cacheManager.GetOrAddAsync("forecast-key",
            () => service.GetForecastsAsync(),
            TimeSpan.FromSeconds(120));
return result;
    })
    .WithName("GetWeatherForecast")
    .WithOpenApi();
}
```

# Boosting Your ASP.NET Core App with Multi-Layer Caching: A Comprehensive Guide

https://medium.com/@michaelmaurice410/boosting-your-asp-net-core-app-with-multi-layer-caching-a-comprehensive-guide-63cc5267425a

https://github.com/MICHAEL-MAURICE/MultiLayerCache

https://github.com/gtechsltn/MultiLayerCache
