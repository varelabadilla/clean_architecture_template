# Infrastructure / Identity

## Why This Folder Exists

Authentication and authorization are infrastructure concerns. The **who is this person** and **are they allowed** questions involve JWTs, OAuth providers, ASP.NET Core Identity, or external identity systems — all of which are technical implementations that have no place in the Domain or Application layers.

This folder isolates all identity-related infrastructure: user store configuration, token generation, JWT validation setup, and role/permission management.

---

## What Belongs Here

- ASP.NET Core Identity configuration (if used)
- JWT token generation and validation
- OAuth / OpenID Connect provider configuration
- Permission and role seed data or configuration
- Custom `UserManager` or `SignInManager` extensions
- Identity-related database context configuration (if using a separate context)

---

## What Does NOT Belong Here

- Authorization policies for business rules (those belong in `Api/` or Application)
- `ICurrentUserContext` implementation (that lives in `Services/` — it uses `IHttpContextAccessor`)
- User domain entities (if users are a domain concept, the entity lives in `Domain/Entities/`)

---

## Recommended File Names

```plaintext
IdentityConfiguration.cs         // ASP.NET Identity options setup
JwtTokenService.cs                // token generation
JwtSettings.cs                    // strongly-typed options
TokenValidationConfiguration.cs  // JWT validation parameters
PermissionAuthorizationHandler.cs
ApplicationUser.cs                // Identity user class (IdentityUser subclass)
ApplicationRole.cs                // Identity role class
```

---

## JWT Configuration Example

```csharp
// JwtSettings.cs
public class JwtSettings
{
    public const string SectionName = "Jwt";
    public string SecretKey { get; init; } = string.Empty;
    public string Issuer { get; init; } = string.Empty;
    public string Audience { get; init; } = string.Empty;
    public int ExpiryMinutes { get; init; } = 60;
}

// JwtTokenService.cs
public class JwtTokenService : IJwtTokenService
{
    private readonly JwtSettings _settings;

    public JwtTokenService(IOptions<JwtSettings> settings)
        => _settings = settings.Value;

    public string GenerateToken(Guid userId, string email, IEnumerable<string> roles)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, userId.ToString()),
            new(ClaimTypes.Email, email),
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_settings.SecretKey));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _settings.Issuer,
            audience: _settings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_settings.ExpiryMinutes),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Important Notes

- `ApplicationUser` is the ASP.NET Core Identity user class. It lives here because it is an infrastructure type, not a domain entity. If users are a first-class domain concept in your application, define a separate `User` domain entity and keep `ApplicationUser` as the infrastructure persistence concern.
- JWT secret keys must be stored securely (environment variables, Key Vault). Never commit them to source control.
- In Java/Spring, this folder maps to Spring Security configuration: `SecurityFilterChain`, `JwtAuthenticationFilter`, `JwtTokenProvider`, and `@PreAuthorize` configuration.
