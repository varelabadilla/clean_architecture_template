# Infrastructure / Caching

## Why This Folder Exists

Caching is a technical concern. The decision to cache data, where to cache it (in-memory, Redis, hybrid), and how to serialize/deserialize cache entries are all infrastructure decisions that belong here — not in the Application layer.

The Application layer uses `ICacheService` (defined in `Application/Abstractions/`) without knowing whether data is cached in Redis, MemoryCache, or not at all. This folder provides the concrete implementation.

---

## What Belongs Here

- `ICacheService` implementation (Redis, in-memory, or hybrid)
- Cache key generators or constants
- Cache serialization configuration
- Distributed cache setup helpers

---

## What Does NOT Belong Here

- `ICacheService` interface (that lives in `Application/Abstractions/`)
- Business logic for deciding what to cache (that belongs in `CachingBehavior` or handlers)
- Cache invalidation domain rules

---

## Recommended File Names

```plaintext
RedisCacheService.cs
InMemoryCacheService.cs
HybridCacheService.cs
CacheKeys.cs                    // optional: centralized cache key constants
```

---

## Redis Implementation Example

```csharp
public class RedisCacheService : ICacheService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<RedisCacheService> _logger;

    private static readonly JsonSerializerOptions _jsonOptions = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };

    public RedisCacheService(
        IDistributedCache cache,
        ILogger<RedisCacheService> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<T?> GetAsync<T>(
        string key,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var data = await _cache.GetStringAsync(key, cancellationToken);
            return data is null ? default : JsonSerializer.Deserialize<T>(data, _jsonOptions);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Cache GET failed for key {Key}", key);
            return default;
        }
    }

    public async Task SetAsync<T>(
        string key,
        T value,
        TimeSpan? expiry = null,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var options = new DistributedCacheEntryOptions();
            if (expiry.HasValue)
                options.SetAbsoluteExpiration(expiry.Value);

            var data = JsonSerializer.Serialize(value, _jsonOptions);
            await _cache.SetStringAsync(key, data, options, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Cache SET failed for key {Key}", key);
        }
    }

    public async Task RemoveAsync(
        string key,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _cache.RemoveAsync(key, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Cache REMOVE failed for key {Key}", key);
        }
    }
}
```

---

## Cache Keys

Define cache keys as constants or static properties to prevent typos and duplication across handlers:

```csharp
public static class CacheKeys
{
    public static string Order(Guid id) => $"order:{id}";
    public static string CustomerOrders(Guid customerId) => $"customer:{customerId}:orders";
    public static string ProductCatalog => "product:catalog";
}
```

---

## Important Notes

- Cache failures should **never crash the application**. Always wrap cache operations in try/catch and log warnings — fall through to the source of truth if the cache is unavailable.
- Choose absolute expiration for frequently-changing data and sliding expiration for user-specific session data. Document the reasoning for each TTL.
- In Java/Spring, use Spring Cache abstraction (`@Cacheable`, `@CacheEvict`) with a Redis `CacheManager` configuration, or implement `ICacheService` equivalent using `RedisTemplate`.
