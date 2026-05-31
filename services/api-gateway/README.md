# ApiGateway

## Responsibility

ApiGateway is the browser-facing reverse proxy. It routes external HTTP and WebSocket traffic to downstream services using YARP.

Source path:

```text
../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway
```

## Owns

- Gateway route table.
- Downstream cluster configuration.
- External entry point for REST APIs and SignalR hub routing.
- Cross-origin and request forwarding behavior at the gateway level.

## Must Not Own

- Business logic.
- Database state.
- Domain validation.
- Payment, order, catalog, user, media, or notification decisions.

## Runtime

| Item | Value |
|---|---|
| Framework | ASP.NET Core + YARP |
| Local HTTP | `http://localhost:5000` |
| Local HTTPS | `https://localhost:7001` |
| Database | None |

## Planning Notes

- New public HTTP APIs normally require a gateway route only when they introduce a new path prefix or downstream service.
- Existing prefixes should remain owned by their current service.
- Gateway changes must be coordinated with frontend `VITE_GATEWAY_BASE_URL` and API version assumptions.
- IdentityServer public authority endpoints are served directly by IdentityService on `http://localhost:5001`; ApiGateway does not forward `/.well-known/**`, `/connect/**`, or `/Account/**`.
- Browser login, registration, session refresh, and logout are routed through `/api/v1/accounts/**` to IdentityService.
- Cookie-authenticated browser API requests use gateway mediation: ApiGateway validates the access-token cookie, forwards `Authorization: Bearer <token>` to downstream services, strips HiveSpace auth/CSRF cookies before forwarding, and rejects state-changing requests without the configured CSRF header.
- Gateway-mediated cookie browser sessions are documented in [ADR-0003](../../architecture/decisions/ADR-0003-gateway-mediated-cookie-browser-sessions.md).
- UserService profile/store routes target UserService on `http://localhost:5007`.
- Keep gateway behavior thin; service authorization and domain validation belong downstream.

## Detail

- API route ownership: `api.md`
