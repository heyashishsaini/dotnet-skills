# JWT, Identity & Auth Reference

## JWT Flow
```
Client → POST /auth/login → [Validate credentials] → Generate JWT → Return token
Client → GET /api/orders → [Bearer token in header] → Validate JWT → Execute action
```

## JWT Setup
```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });
```

## Token Generation Service
```csharp
public string GenerateToken(AppUser user, IList<string> roles)
{
    var claims = new List<Claim>
    {
        new(ClaimTypes.NameIdentifier, user.Id),
        new(ClaimTypes.Email, user.Email!),
    };
    claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]!));
    var token = new JwtSecurityToken(
        issuer: _config["Jwt:Issuer"],
        audience: _config["Jwt:Audience"],
        claims: claims,
        expires: DateTime.UtcNow.AddHours(1),
        signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

## ASP.NET Core Identity Setup
```csharp
builder.Services.AddIdentity<AppUser, IdentityRole>(options =>
{
    options.Password.RequiredLength = 8;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.User.RequireUniqueEmail = true;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();
```

## Authorization Policies
```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
    options.AddPolicy("MinAge18", policy => policy.RequireClaim("age", "18+"));
});

// Usage
[Authorize(Policy = "AdminOnly")]
[HttpDelete("{id}")]
public async Task<IActionResult> Delete(Guid id) { ... }
```

## Refresh Token Pattern
```
Access Token: short-lived (15 min – 1 hour)
Refresh Token: long-lived (7–30 days), stored in HttpOnly cookie
Flow: expired access token → send refresh token → get new access token
```

## Common Mistakes
1. Storing JWT secret in code or appsettings.json committed to git
2. Not validating token expiry (`ValidateLifetime = false`)
3. Storing sensitive data in JWT payload (it's base64, not encrypted)
4. Using symmetric key in distributed systems (use asymmetric RS256)
5. Not invalidating refresh tokens on logout
