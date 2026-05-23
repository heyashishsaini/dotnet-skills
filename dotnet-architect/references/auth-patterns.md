# Auth Patterns in .NET

## JWT + Refresh Token Flow (Default Recommendation)

### Access Token
- Short-lived: 15 minutes
- Signed with RS256 (asymmetric) in production — allows verification without sharing the secret
- Contains: userId, email, roles, tenantId (if multi-tenant)
- Stored in memory on the client (not localStorage — XSS risk)

### Refresh Token
- Long-lived: 7–30 days
- Stored in an HttpOnly, Secure, SameSite=Strict cookie
- One-time use with rotation — each use issues a new refresh token
- Stored hashed in the database (treat like a password)
- Must be revocable (track in DB, revoke on logout or suspicious activity)

### Token Generation

```csharp
public string GenerateAccessToken(User user)
{
    var claims = new[]
    {
        new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
        new Claim(JwtRegisteredClaimNames.Email, user.Email),
        new Claim("role", user.Role.ToString()),
        new Claim("tenantId", user.TenantId.ToString())
    };

    var key = new RsaSecurityKey(_rsa);
    var creds = new SigningCredentials(key, SecurityAlgorithms.RsaSha256);

    var token = new JwtSecurityToken(
        issuer: _options.Issuer,
        audience: _options.Audience,
        claims: claims,
        expires: DateTime.UtcNow.AddMinutes(15),
        signingCredentials: creds);

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

### Refresh Token Rotation

```csharp
public async Task<AuthResult> RefreshAsync(string refreshToken, CancellationToken ct)
{
    var stored = await _db.RefreshTokens
        .Include(r => r.User)
        .FirstOrDefaultAsync(r => r.TokenHash == Hash(refreshToken), ct);

    if (stored is null || stored.IsRevoked || stored.ExpiresAt < DateTime.UtcNow)
        throw new UnauthorizedException("Invalid or expired refresh token.");

    // Revoke old, issue new
    stored.IsRevoked = true;
    var newRefreshToken = GenerateRefreshToken(stored.User);
    _db.RefreshTokens.Add(newRefreshToken);
    await _db.SaveChangesAsync(ct);

    return new AuthResult(
        AccessToken: GenerateAccessToken(stored.User),
        RefreshToken: newRefreshToken.Token);
}
```

## Authorization Patterns

### Role-Based (simple)
```csharp
[Authorize(Roles = "Admin,Manager")]
public IActionResult AdminOnly() { ... }
```

### Policy-Based (recommended for anything non-trivial)
```csharp
// Registration
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanPublishContent", policy =>
        policy.RequireRole("Editor", "Admin")
              .RequireClaim("tenantId"));
});

// Usage
[Authorize(Policy = "CanPublishContent")]
public IActionResult Publish() { ... }
```

### Resource-Based Authorization
When the authorization decision depends on the resource being accessed:
```csharp
var authResult = await _authService.AuthorizeAsync(User, document, "CanEditDocument");
if (!authResult.Succeeded) return Forbid();
```

Use `IAuthorizationHandler` to implement the policy logic.

## Multi-Tenancy

Two main approaches:

**Shared database, tenant discriminator column**
- Add `TenantId` to every entity
- Use a global query filter in EF Core to automatically scope queries
- Simpler to operate, weaker isolation

```csharp
builder.Entity<Order>().HasQueryFilter(o => o.TenantId == _tenantContext.TenantId);
```

**Separate database per tenant**
- Stronger isolation, easier compliance (GDPR, SOC2)
- Complex to operate — migrations across N databases
- Use connection string resolution at runtime based on tenant

Resolve tenant from: subdomain, custom header (`X-Tenant-Id`), or JWT claim.

## OAuth2 / External Identity

For B2C apps or when you want social login, prefer delegating to an Identity Provider:
- Azure AD B2C
- Auth0
- Keycloak (self-hosted)

Your API becomes a resource server, validating tokens issued by the IdP. Don't build your
own OAuth2 server — use `OpenIddict` or `Duende IdentityServer` only if you genuinely need
to issue tokens to third parties.
