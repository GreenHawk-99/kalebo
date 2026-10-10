# kalebo

A clean, modular Ktor 3 application built with Kotlin, JetBrains Exposed, HikariCP, and MySQL. Kalebo provides the main API and database gateway for the project group.

## Related projects

- [Pokitaru](file:///C:/Users/green/WebstormProjects/pokitaru) — React/TypeScript frontend.
- [Veldin](file:///C:/Users/green/PycharmProjects/veldin) — downloader backend.
- See [Pokitaru's `guideline.md`](file:///C:/Users/green/WebstormProjects/pokitaru/guideline.md) for the documented service mapping and request flow.

## Development

### Project structure

```
kalebo/
├── docker-compose.yml             # MySQL database container configuration
├── build.gradle.kts               # Gradle build & dependency definitions
├── settings.gradle.kts            # Gradle settings & project configuration
├── gradle/
│   └── libs.versions.toml         # Version catalog
├── gradlew / gradlew.bat          # Gradle wrapper binaries
└── src/
    ├── main/
    │   ├── kotlin/
    │   │   └── me/greenhawk/kalebo/
    │   │       ├── main.kt            # Application entry point
    │   │       ├── config/
    │   │       │   └── DatabaseFactory.kt # HikariCP pool & Exposed initialization
    │   │       ├── dao/
    │   │       │   ├── Tables.kt          # Exposed Table definitions
    │   │       │   ├── UserDao.kt         # User DAO interface & implementation
    │   │       │   ├── ServerDao.kt       # Server DAO interface & implementation
    │   │       │   └── NotificationDao.kt # Notification DAO interface & implementation
    │   │       ├── dto/
    │   │       │   ├── ApiResponse.kt     # Generic API response wrapper
    │   │       │   ├── UserDto.kt         # User DTOs
    │   │       │   ├── ServerDto.kt       # Server DTOs
    │   │       │   ├── CreditDto.kt       # Credit DTOs
    │   │       │   └── NotificationDto.kt # Notification DTOs
    │   │       ├── plugins/
    │   │       │   ├── Databases.kt       # Database lifecycle plugin
    │   │       │   ├── Http.kt            # CORS & Default headers
    │   │       │   ├── Routing.kt         # Root routing aggregator
    │   │       │   ├── Serialization.kt   # JSON Content Negotiation
    │   │       │   └── StatusPages.kt     # Global error handling
    │   │       └── routes/
    │   │           ├── UserRoutes.kt          # User REST endpoints (/api/users)
    │   │           ├── ServerRoutes.kt        # Server REST endpoints (/api/servers)
    │   │           └── NotificationRoutes.kt  # Notification REST endpoints (/api/users/{steamId}/notifications)
    │   └── resources/
    │       ├── application.yaml   # Ktor server & database configurations
    │       └── logback.xml        # Logging configuration
    └── test/
        └── kotlin/
            └── me/greenhawk/kalebo/
                ├── UserDaoTest.kt     # DAO unit tests (with H2 in-memory MySQL mode)
                ├── UserApiTest.kt     # End-to-end HTTP integration tests
                └── ServerTest.kt      # Root & health endpoint tests
```

### Local MySQL database (Docker Compose)

To spin up the local MySQL database instance:

```bash
docker compose up -d
```

### Database Credentials
- **Host**: `localhost:3306`
- **Database**: `kalebo_db`
- **Username**: `kalebo_user`
- **Password**: `kalebo_password`
- **Root Password**: `rootpassword`

To stop the database:
```bash
docker compose down
```

### Build and run

| Task | Command (Linux / macOS) | Command (Windows) |
|---|---|---|
| **Run Tests** | `./gradlew test` | `.\gradlew.bat test` |
| **Build Project** | `./gradlew build` | `.\gradlew.bat build` |
| **Run Server** | `./gradlew run` | `.\gradlew.bat run` |

### Steam authentication

Steam sign-in uses OpenID 2.0 (not OAuth 2.0). Configure the API with an externally reachable HTTPS callback URL and a JWT signing secret before starting it:

| Environment variable | Required | Purpose |
|---|---|---|
| `STEAM_CALLBACK_URL` | Yes for Steam login | Exact callback URL, e.g. `https://api.example.com/auth/steam/callback` |
| `JWT_SECRET` | Yes | HMAC signing secret of at least 32 characters; keep it out of source control |
| `JWT_ISSUER` | No | JWT issuer; defaults to `kalebo-api` |
| `JWT_AUDIENCE` | No | JWT audience; defaults to `kalebo-client` |
| `JWT_EXPIRATION_SECONDS` | No | Token lifetime in seconds; defaults to `3600` |

The API fails to start without `JWT_SECRET`. `STEAM_CALLBACK_URL` must use HTTPS, have no query or fragment, and end in `/auth/steam/callback`; configure that same URL in the Steam OpenID flow.

1. The frontend sends the user's browser to `GET /auth/steam`.
2. Steam redirects to `GET /auth/steam/callback`; the API verifies the OpenID response with Steam, finds or provisions the account, and assigns new accounts the `user` role.
3. The callback responds with JSON containing the Kalebo bearer token and user data:

```json
{
  "token": "<kalebo-jwt>",
  "user": {
    "steamId": "76561198000000010",
    "username": "76561198000000010",
    "role": "user"
  }
}
```

Send that token as `Authorization: Bearer <kalebo-jwt>` on API mutations. Read-only API requests, `/`, and `/health` remain public; all `POST`, `PUT`, and `DELETE` API routes require a valid, unexpired JWT. User and server identity is derived from the authenticated SteamID64, and server/notification mutations are limited to their owner. Caller-provided Steam IDs and roles do not grant identity or privileges.

Example public request:

```bash
curl http://localhost:8080/api/users
```

Example authenticated request:

```bash
curl -X PUT http://localhost:8080/api/users/76561198000000010 \
  -H "Authorization: Bearer <kalebo-jwt>" \
  -H "Content-Type: application/json" \
  -d '{"username": "player"}'
```
