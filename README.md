# CityPulse API

> One request in, a complete city snapshot out.

CityPulse is a MuleSoft 4 integration API. Give it a city name and it returns a single JSON response that combines the city's location, its live weather, details about its country, and a generated weather tip. Behind one clean endpoint it orchestrates three external APIs, and it also lets clients save favorite cities.

- **Runtime:** Mule 4.x (Anypoint Studio)
- **Project name:** `citypulse-api` (currently `advanced1` in Studio, rename via Refactor, Rename)
- **Base URL (local):** `http://localhost:8081`

---

## 1. Architecture

```
Client (Postman)
      |
      v
HTTP Listener  (source, port 8081)
      |
      v
Mule flows  -- validation, orchestration, DataWeave, error handling
      |
      +--> Open-Meteo Geocoding API   (city name -> latitude/longitude)
      +--> Open-Meteo Weather API     (current weather)      } in parallel
      +--> REST Countries API v5      (country details)      } (Scatter-Gather)
      +--> Object Store               (favorites, persistent)
```

| Role | Component |
|---|---|
| Source (trigger) | HTTP Listener |
| Processors | Choice, Raise Error, Transform Message, Set Variable, Scatter-Gather, Flow Reference |
| Outbound calls | HTTP Request connector (3 external APIs) |
| Storage | Object Store connector |
| Error handling | One global error handler, shared by every flow |

## 2. Endpoints

| Method | Path | Purpose | Success | Status |
|---|---|---|---|---|
| GET | `/city?name=` | City, weather, country info, and tip | 200 | Done |
| POST | `/favorites` | Save a favorite city | 201 | Step 6 |
| GET | `/favorites` | List all saved favorites | 200 | Step 6 |
| GET | `/health` | Application status check | 200 | Step 7 |

### GET /city?name=Cairo

Calls the Geocoding API first, then fetches weather and country data **in parallel**, then merges everything.

```json
{
  "city": "Cairo",
  "country": "Egypt",
  "capital": "Cairo",
  "region": "Africa",
  "population": 109262178,
  "currency": "EGP",
  "flag": "🇪🇬",
  "weather": { "temperatureC": 22.1, "windKmh": 9.4, "isDay": false },
  "tip": "Mild. A light jacket helps.",
  "generatedAt": "2026-10-11T00:30:00+03:00"
}
```

Tip logic: 35 C or above is "very hot", 25 or above is "warm", 15 or above is "mild", otherwise "cold".

### POST /favorites

Request body (`Content-Type: application/json`):

```json
{ "name": "Cairo", "country": "Egypt" }
```

Response `201`:

```json
{ "message": "Favorite saved", "city": "Cairo" }
```

The key is the lowercase city name, so saving the same city twice overwrites it instead of creating a duplicate. Data is persistent and survives restarts.

### GET /favorites

```json
{
  "count": 2,
  "favorites": [
    { "name": "Cairo", "country": "Egypt", "savedAt": "2026-10-11T00:31:00+03:00" },
    { "name": "London", "country": "UK", "savedAt": "2026-10-11T00:32:00+03:00" }
  ]
}
```

### GET /health

```json
{ "status": "UP" }
```

## 3. Error responses

All errors are JSON and come from one global error handler.

| Situation | Status | Body |
|---|---|---|
| `/city` with no `name` | 400 | `{"error": "Bad Request", "message": "Query parameter 'name' is required"}` |
| `POST /favorites` with no `name` | 400 | `{"error": "Bad Request", "message": "Field 'name' is required in the body"}` |
| City does not exist | 404 | `{"error": "Not Found", "message": "City not found"}` |
| An external API is down or fails | 502 | `{"error": "Bad Gateway", "message": "An external service is unavailable"}` |
| Anything unexpected | 500 | `{"error": "Internal Server Error", "message": "Something went wrong"}` |

502 and 500 use fixed messages on purpose, so internal details (hostnames, stack traces) are never exposed.

### How errors work

1. A flow raises a named error, for example `APP:NOT_FOUND`.
2. `global-error-handler` matches it by type (top to bottom, first match wins, `ANY` last).
3. The handler builds the JSON body and stores the status in `vars.httpStatus`.
4. Each listener's **Error response** sends `vars.httpStatus default 500` with the handler's payload.

> Every listener must define its own Error response. Without it, errors fall back to a plain 500.

## 4. External APIs

| API | Host | Path used | Auth |
|---|---|---|---|
| Geocoding | `geocoding-api.open-meteo.com` | `/v1/search` | None |
| Weather | `api.open-meteo.com` | `/v1/forecast` | None |
| Countries (v5) | `api.restcountries.com` | `/countries/v5/codes.alpha_2/{code}` | `Authorization: Bearer <API key>` |

**API key note:** REST Countries retired its old v1 to v4 APIs, and v5 requires a key. During development the public demo key `rc_live_demo` works but always returns a sample record (Canada). For real data, create a free key at `restcountries.com/sign-up`. Never commit the real key. It belongs in `config.properties` (Step 7) and is excluded from version control and screenshots.

## 5. Naming conventions

| Element | Convention | Example |
|---|---|---|
| Project | kebab-case | `citypulse-api` |
| Config files | kebab-case, one per feature | `global-config.xml`, `city-insight.xml`, `favorites.xml` |
| Flows | method + feature + `-flow` | `get-city-insight-flow`, `post-favorites-flow` |
| Sub-flows | kebab-case + `-subflow` | `log-request-subflow` |
| Global configs | purpose + type | `geocoding-http-request-config` |
| Variables | camelCase | `vars.geoResult`, `vars.httpStatus` |
| Properties | lowercase dot notation | `restcountries.api.key` |
| Display names | Say what the step does | `Request Geocoding API` |
| Custom errors | `APP:` namespace, UPPER_SNAKE_CASE | `APP:BAD_REQUEST` |

## 6. Project structure

```
src/main/mule/
  global-config.xml     Default error handler setting + global-error-handler
  city-insight.xml      Listener/request configs, GET /city flow, log sub-flow
  favorites.xml         POST /favorites and GET /favorites flows, Object Store
  health.xml            GET /health flow
src/main/resources/
  config.properties     Hosts, port, API key
src/test/munit/
  citypulse-test-suite.xml
```

## 7. Global configurations

| Name | Type | Settings |
|---|---|---|
| `HTTP_Listener_config` | HTTP Listener | HTTP, `0.0.0.0`, port 8081 |
| `geocoding-http-request-config` | HTTP Request | HTTPS, `geocoding-api.open-meteo.com`, 443 |
| `weather-http-request-config` | HTTP Request | HTTPS, `api.open-meteo.com`, 443 |
| `country-http-request-config` | HTTP Request | HTTPS, `api.restcountries.com`, 443 |
| `favorites-object-store` | Object Store | Persistent |
| `object-store-config` | Object Store connector | Defaults |
| `global-error-handler` | Error handler | 5 handlers, set as the app default |

## 8. Build plan and status

| Step | Topic | Status |
|---|---|---|
| 1 | Geocoding call with HTTP Listener and HTTP Request | Done |
| 2 | Validation (Choice, Raise Error) and custom errors | Done |
| 3 | Scatter-Gather: weather and country in parallel | Done |
| 4 | Merge into the final response with the weather tip | Done |
| 5 | Logging sub-flow and global error handler (incl. 502) | Done |
| 6 | Favorites with the Object Store (POST and GET) | In progress |
| 7 | `GET /health`, properties file, secure the API key | Planned |
| 8 | MUnit tests, Postman collection, flow diagram | Planned |

### Step details

1. **Geocoding:** `GET /city` calls the Geocoding API with `attributes.queryParams.name`.
2. **Validation:** two Choice routers raise `APP:BAD_REQUEST` (missing name) and `APP:NOT_FOUND` (empty `results`).
3. **Scatter-Gather:** the geocoding result is saved in `vars.geoResult`, then weather and country requests run at the same time. Total time is the slower call, not the sum.
4. **Merge:** one Transform Message combines both routes (`payload."0"` and `payload."1"`) and builds the tip with `if / else if`.
5. **Reuse and errors:** `log-request-subflow` is called by every flow through a Flow Reference. `global-error-handler` handles `APP:BAD_REQUEST`, `APP:NOT_FOUND`, `HTTP:CONNECTIVITY`, `MULE:COMPOSITE_ROUTING` (Scatter-Gather failures), and `ANY`.
6. **Favorites:** a persistent Object Store with Store and Retrieve all. Keys are lowercase city names.
7. **Health and config:** `/health` flow, `config.properties` with `${...}` placeholders, API key moved out of the flow.
8. **Quality:** MUnit tests with mocked HTTP requests, an exported Postman collection, and a flow diagram.

## 9. Mule concepts demonstrated

- Flows, sub-flows, and Flow Reference (reuse)
- HTTP Listener and HTTP Request connectors with global configurations
- Query params, request bodies, and `attributes`
- DataWeave: variables, nested data, `if / else`, `pluck`, `default`, `now()`
- Choice router and conditional logic
- Raise Error, custom error types, On Error Propagate
- Global error handler and default error handler setting
- Scatter-Gather (parallel processing)
- Object Store (persistence)
- Properties files and externalized secrets
- MUnit testing

## 10. Running and testing

1. Save all files (Ctrl+Shift+S).
2. Right-click the project, then **Run As, Mule Application**.
3. Wait for `DEPLOYED` in the Console.
4. Run the tests below in Postman.

### Test cases

| # | Request | Expected |
|---|---|---|
| 1 | `GET /city?name=Cairo` | 200, combined JSON |
| 2 | `GET /city?name=London` | 200 |
| 3 | `GET /city?name=Tokyo` | 200 |
| 4 | `GET /city?name=cairo` | 200 (case-insensitive) |
| 5 | `GET /city?name=New York` | 200 |
| 6 | `GET /city` | 400 |
| 7 | `GET /city?name=` | 400 |
| 8 | `GET /city?name=Xyzabcq` | 404 |
| 9 | `GET /cityy` | 404, `No listener for endpoint` |
| 10 | `POST /city?name=Cairo` | 405 |
| 11 | Weather host set to `api.open-meteo.invalid` | 502 |
| 12 | Geocoding host set to `geocoding-api.open-meteo.invalid` | 502 |
| 13 | Country header key set to a wrong value | 502 |
| 14 | `GET /favorites` (empty store) | 200, `count` 0 |
| 15 | `POST /favorites` `{"name": "Cairo", "country": "Egypt"}` | 201 |
| 16 | `POST /favorites` `{"name": "London", "country": "UK"}` | 201 |
| 17 | `GET /favorites` | 200, `count` 2 |
| 18 | `POST /favorites` `{"country": "Egypt"}` | 400 |
| 19 | `POST /favorites` Cairo again | 201, `count` stays 2 |
| 20 | Restart the app, then `GET /favorites` | Favorites still present |
| 21 | `GET /health` | 200, `{"status": "UP"}` |

For tests 11 to 13, change the setting, restart, call `/city?name=Cairo`, then change it back.

The Console should also show one `Request <METHOD> <path> from ...` line per call (the sub-flow), and an error stack trace after any 4xx or 5xx (normal, because On Error Propagate logs and re-throws).

## 11. Troubleshooting log

| Symptom | Cause | Fix |
|---|---|---|
| Build fails with `Premature end of file` | Empty `.xml` file in `src/main/mule` | Delete it, or recreate with New, Mule Configuration File |
| `PARSE_EXPRESSION attribute type without an event` | Double `#[ #[ ... ] ]` | With fx on, write plain DataWeave with no `#[ ]` |
| `No host value provided` / `getHost() is null` | Request config has an empty Host | Edit the config and fill in Host |
| Geocoding returns 400 on port 443 | Protocol set to HTTP | Set Protocol to HTTPS |
| `No listener for endpoint: /city` | App not restarted, or no listener config | Restart the app and check the listener |
| Country call returns `success: false`, deprecated | REST Countries v3.1 was retired | Use v5 host, path, and a Bearer key |
| Always 500 on errors | Listener Error response missing | Set status `vars.httpStatus default 500` and body `payload` |
| Variable `vars.geoResult` is null | Stray tab or space in the variable name | Retype the Name as `geoResult` |
| `HTTP:BAD_REQUEST` thrown for a missing name | Choice router missing or placed after the request | Put `Check Name Provided` before the Geocoding call |

## 12. Author

Name: ____________________  
Date: 2026-10-11
