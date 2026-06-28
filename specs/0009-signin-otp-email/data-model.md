# Data Model: Email OTP Sign-In (0009)

## Service: IdentityService

### Entity: OtpChallenge

Represents a reusable auth-scoped OTP challenge owned by IdentityService. V1 uses it only for email OTP sign-in with `Purpose = SignIn`. The challenge token is the sole lookup key at verification time; the email address is never re-submitted during code verification.

```csharp
public class OtpChallenge : Entity<OtpChallengeId>
{
    protected OtpChallenge() { }

    public string EmailNormalized { get; private set; }
    public OtpChallengePurpose Purpose { get; private set; }
    public string ChallengeToken { get; private set; }
    public string Code { get; private set; }
    public DateTimeOffset ExpiresAt { get; private set; }
    public DateTimeOffset CanResendAt { get; private set; }
    public int AttemptCount { get; private set; }
    public bool IsUsed { get; private set; }
    public bool IsInvalidated { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }

    public static OtpChallenge Create(
        string emailNormalized,
        OtpChallengePurpose purpose,
        string challengeToken,
        string code,
        DateTimeOffset expiresAt,
        DateTimeOffset canResendAt)
    {
        return new OtpChallenge
        {
            Id = OtpChallengeId.New(),
            EmailNormalized = emailNormalized,
            Purpose = purpose,
            ChallengeToken = challengeToken,
            Code = code,
            ExpiresAt = expiresAt,
            CanResendAt = canResendAt,
            AttemptCount = 0,
            IsUsed = false,
            IsInvalidated = false,
            CreatedAt = DateTimeOffset.UtcNow,
        };
    }

    public void IncrementAttempt() => AttemptCount++;
    public void MarkUsed() => IsUsed = true;
    public void Invalidate() => IsInvalidated = true;
}
```

### Value Object: OtpChallengeId

```csharp
public record OtpChallengeId(Guid Value)
{
    public static OtpChallengeId New() => new(Ulid.NewUlid().ToGuid());
}
```

### Enum: OtpChallengePurpose

```csharp
public enum OtpChallengePurpose
{
    SignIn = 1
}
```

Future IdentityService auth flows may add values such as password reset or step-up verification without replacing the entity or table.

Representation rules for this feature:
- Domain model uses the `OtpChallengePurpose` enum.
- Database storage persists `purpose` as `int`.
- Integration events serialize `Purpose` as the enum name string.
- V1 allowed value is `SignIn` only.

### EF Core Configuration

Table: `otp_challenges`

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | `uniqueidentifier` | NOT NULL | Primary key; ULID-backed GUID |
| `email_normalized` | `nvarchar(256)` | NOT NULL | Indexed for cooldown lookup |
| `purpose` | `int` | NOT NULL | Purpose discriminator; v1 uses `SignIn` only |
| `challenge_token` | `nvarchar(64)` | NOT NULL | Unique lookup token |
| `code` | `nvarchar(6)` | NOT NULL | 6-digit numeric code |
| `expires_at` | `datetimeoffset` | NOT NULL | |
| `can_resend_at` | `datetimeoffset` | NOT NULL | |
| `attempt_count` | `int` | NOT NULL | Default 0 |
| `is_used` | `bit` | NOT NULL | Default 0 |
| `is_invalidated` | `bit` | NOT NULL | Default 0 |
| `created_at` | `datetimeoffset` | NOT NULL | |

**Indexes**:
- Primary key on `id`
- Unique index on `challenge_token`
- Non-unique composite index on `(email_normalized, purpose)` for cooldown and resend lookup

**Migration name**: `AddOtpChallenge`

### Business Invariants

1. An `OtpChallenge` is valid only when `IsUsed = false`, `IsInvalidated = false`, and `ExpiresAt > now`.
2. OTP verification always looks up by `ChallengeToken` only.
3. Attempt limiting is enforced per challenge record; reaching the configured maximum invalidates the challenge immediately.
4. Cooldown checks are purpose-aware; this feature only issues and verifies `Purpose = SignIn` challenges.
5. Resend invalidates the prior active sign-in challenge and creates a new one.
6. Only sign-in challenges may issue browser auth cookies in this feature.

### Configuration Values

| Setting | Default | Notes |
|---|---|---|
| `Otp:CodeLengthDigits` | `6` | Numeric code length |
| `Otp:ExpiryMinutes` | `10` | Challenge lifetime |
| `Otp:CooldownSeconds` | TBD at implementation | Per-email, per-purpose cooldown |
| `Otp:MaxAttempts` | `5` | Maximum incorrect attempts |
| `Otp:CleanupIntervalHours` | `6` | Cleanup cadence |
| `Otp:CleanupGraceHours` | `24` | Retention after expiry |

## Integration Event Contract

### UserOtpChallengeRequestedIntegrationEvent

Defined in `HiveSpace.Infrastructure.Messaging.Shared`. Inherits from `IntegrationEvent`.

```csharp
public class UserOtpChallengeRequestedIntegrationEvent : IntegrationEvent
{
    public string RecipientEmail { get; init; }
    public string OtpCode { get; init; }
    public DateTimeOffset ExpiresAt { get; init; }
    public string Purpose { get; init; } // enum name string; "SignIn" in v1
}
```

**Publisher**: IdentityService application layer via the transactional outbox  
**Consumer**: NotificationService for OTP email delivery  
**V1 purpose**: `SignIn`  
**Wire format**: string enum name
