# Agent guide: Zoho CRM Python SDK 8.0

This document gives AI assistants and developers concise context for working in this repository.

## What this repo is

- **Zoho CRM Python SDK 8.0**: Python wrapper for **Zoho CRM REST API v8**.
- **Package name**: `zohocrmsdk8_0` (install: `pip install zohocrmsdk8_0`).
- **Auth**: OAuth 2.0; tokens can be stored in a file (default), MySQL (optional), or a custom store.
- **License**: Apache 2.0.

## Repository structure

```
zohocrm-python-sdk-8.0/
├── README.md                 # User-facing overview and install
├── setup.py                  # Package definition (no pyproject.toml)
├── AGENTS.md                 # This file
├── zohocrmsdk/               # Main SDK package (used by root samples)
│   └── src/
│       ├── json_details.json # API metadata used by SDK
│       └── com/zoho/
│           ├── api/authenticator/   # OAuth, Token, TokenStore, OAuthToken
│           │   └── store/           # FileStore, DBStore (MySQL)
│           ├── api/logger/          # Logger, SDKLogger
│           └── crm/api/             # All CRM API modules
│               ├── initializer.py   # SDK initialization
│               ├── dc/              # DataCenter (US, EU, IN, etc.)
│               ├── util/             # CommonAPIHandler, connectors, converters
│               ├── exception/        # SDKException
│               └── <feature>/        # record, fields, modules, users, ...
├── samples/                  # Example scripts by feature
│   ├── record/, fields/, modules/, users/, bulk_read/, ...
│   └── custom_store/        # Custom TokenStore example
└── versions/                 # Versioned snapshots (1.0.0 … 5.0.0)
    └── <ver>/zohocrmsdk/, samples/
```

- **Edits for the “current” SDK**: change code under **`zohocrmsdk/`** at repo root (not under `versions/`).
- **Samples** import from the long path, e.g. `from zohocrmsdk.src.com.zoho.crm.api ...`.

## Core concepts

### 1. Initialization (required before any API call)

- **Entry point**: `Initializer.initialize(environment, token, store=None, sdk_config=None, resource_path=None, logger=None, proxy=None)`.
- **environment**: A `DataCenter.Environment` (e.g. `USDataCenter.PRODUCTION()`). Data centers: US, EU, IN, CN, JP, AU, CA, SA.
- **token**: An implementation of `Token`; in practice always `OAuthToken` (client_id, client_secret, grant_token or refresh_token, redirect_url, etc.).
- **store**: Where tokens are persisted. Default `None` → **FileStore** at `os.getcwd() + "/sdk_tokens.txt"`. For MySQL use **DBStore** (requires `pip install zohocrmsdk8_0[mysql]`). For custom backends implement **TokenStore** (find_token, save_token, delete_token, get_tokens, delete_tokens, find_token_by_id).
- **resource_path**: Directory for SDK-generated JSON (e.g. module field details). Default `os.getcwd()`. Must be a valid directory path.
- **sdk_config**: Optional `SDKConfig` (auto_refresh_fields, pick_list_validation, read_timeout, connect_timeout, update_api_domain).
- **logger**: Optional `Logger(level, file_path)`. If file_path is set, SDK uses `FileHandler` (writes to disk). For cloud, prefer `logger=None` or a logger that logs to stdout only.
- **proxy**: Optional `RequestProxy` for HTTP proxy.

After init, the SDK holds a **global** initializer (and optionally per-thread state for `switch_user`). There is no dependency-injected “client” instance.

### 2. Authentication and token storage

- **Token** (abstract): `authenticate(url_connection)`, `remove()`, `generate_token()`.
- **OAuthToken**: Handles grant/refresh and access token; calls `store.find_token()` / `store.save_token()` so tokens persist across runs. Access token is refreshed automatically when missing or near expiry.
- **FileStore**: Persists tokens in a **CSV file** at the given path (default `sdk_tokens.txt` in cwd). Local filesystem only.
- **DBStore**: Persists tokens in **MySQL** (table default name `oauthtoken`, DB default `zohooauth`). Requires `mysql.connector`. Constants for defaults: `Constants.MYSQL_HOST`, `MYSQL_DATABASE_NAME`, etc.
- Tokens are **environment- and domain-specific** (e.g. Production vs Sandbox, US vs EU). Using the wrong token for the environment/domain will cause errors.

### 3. How an API call is made

- Each feature has an **Operations** class (e.g. `RecordOperations(module_api_name)`).
- Pattern:
  1. Build API path (e.g. `/crm/v8/Leads/`).
  2. Create **CommonAPIHandler**; set path, HTTP method, category (READ/CREATE/UPDATE/DELETE/ACTION), **ParameterMap**, **HeaderMap**, optional body.
  3. Call **Utility.get_fields(...)** when the operation needs module field metadata.
  4. Call **handler_instance.api_call(ResponseHandler.__module__, 'application/json')**.
- **CommonAPIHandler.api_call()**: Resolves URL from Initializer’s environment, authenticates the **APIHTTPConnector** via `Initializer.get_initializer().token.authenticate(connector)`, serializes body with **JSONConverter** (or other converter), calls **APIHTTPConnector.fire_request()** (uses **requests**), then deserializes response into the feature’s response types (e.g. **ResponseWrapper**, **APIException**).
- Responses are returned as **APIResponse** (headers, status_code, object, optional response_json).

### 4. Request/response types

- **ParameterMap** + **Param**: query parameters. **HeaderMap** + **Header**: headers.
- Request bodies: feature-specific **BodyWrapper** / **ActionWrapper** etc.
- Response body types: feature-specific (e.g. **ResponseWrapper** with `get_data()`, **APIException** with status/code/details/message). Check `response.get_object()` and branch on type (e.g. `isinstance(..., ResponseWrapper)` or `APIException`).
- **APIResponse**: `get_status_code()`, `get_object()`, `get_headers()`, etc.

### 5. Data centers and environments

- **DataCenter** is abstract. Concrete classes: **USDataCenter**, **EUDataCenter**, **INDataCenter**, **CNDataCenter**, **JPDataCenter**, **AUDataCenter**, **CADataCenter**, **SADataCenter**.
- Each exposes **PRODUCTION()**, **SANDBOX()**, **DEVELOPER()** returning an **Environment** (url, accounts_url, file_upload_url).
- **DataCenter.get(config)** can return the right data center from a config string (e.g. URL or region key).

### 6. Exceptions and config

- **SDKException(code, message, details=None, cause=None)** for initialization errors, auth errors, and API handling. Catch this in application code.
- **SDKConfig**: auto_refresh_fields, pick_list_validation, read_timeout, connect_timeout, update_api_domain.

## Important file locations

| Purpose | Path |
|--------|------|
| SDK init | `zohocrmsdk/src/com/zoho/crm/api/initializer.py` |
| OAuth token | `zohocrmsdk/src/com/zoho/api/authenticator/oauth_token.py` |
| Token stores | `zohocrmsdk/src/com/zoho/api/authenticator/store/token_store.py`, `file_store.py`, `db_store.py` |
| API request/response flow | `zohocrmsdk/src/com/zoho/crm/api/util/common_api_handler.py` |
| HTTP layer | `zohocrmsdk/src/com/zoho/crm/api/util/api_http_connector.py` |
| Constants | `zohocrmsdk/src/com/zoho/crm/api/util/constants.py` |
| Data centers | `zohocrmsdk/src/com/zoho/crm/api/dc/*.py` |
| Record API (example) | `zohocrmsdk/src/com/zoho/crm/api/record/record_operations.py` |
| API metadata | `zohocrmsdk/src/json_details.json` |
| Public API surface | `zohocrmsdk/src/com/zoho/crm/api/__init__.py` (re-exports modules) |

## Conventions in this codebase

- **Imports**: Many files use a try/except pattern: try `from zohocrmsdk.src.com.zoho...` then except `from ..relative...`. This supports both “installed package” and “editable” layouts. Prefer not to broaden `except` beyond `ImportError` when adding new code.
- **Operations**: Each API area has a module with `*_operations.py` (e.g. `record_operations.py`) defining an Operations class. Methods take optional **ParameterMap** and **HeaderMap**; POST/PUT/PATCH take a body type (e.g. BodyWrapper).
- **Naming**: Java-style in places: getters/setters for private attributes (`__name`), class names like **BodyWrapper**, **ResponseWrapper**, **APIException**.
- **Indentation**: Some files use **tabs**; PEP 8 prefers spaces. Be consistent when editing a file.
- **No type hints**: The SDK does not use `typing`. Docstrings describe parameter types.

## Multi-user / threading

- **Initializer.switch_user(environment, token, sdk_config, proxy)** sets a **thread-local** initializer so different threads can use different credentials. The global `Initializer.initializer` is still used when the thread-local one is not set.
- For multi-tenant or multi-user apps, ensure each request/thread either uses the global init (single user) or calls `switch_user()` with the right token and that tokens are loaded from a shared store (e.g. DBStore or custom TokenStore) keyed by user/tenant.

## Samples

- **samples/** contains example scripts by feature (record, fields, modules, users, bulk_read, custom_store, etc.).
- Samples typically: call `Initializer.initialize(...)` with placeholder credentials, then call an Operations method and print or handle the response.
- **samples/custom_store/** shows how to implement a custom **TokenStore**.
- To run a sample: set valid OAuth credentials (or refresh_token), then run the sample script; ensure the SDK is importable (`pip install -e .` from repo root or install `zohocrmsdk8_0`).

## Cloud and production considerations

- **Default token store is file-based**: In serverless or ephemeral environments, use **DBStore** or a **custom TokenStore** (e.g. Redis, DynamoDB) so tokens persist and are shared across instances.
- **resource_path** defaults to `os.getcwd()`; in read-only or ephemeral environments, pass a writable path or accept that field-detail caching may not work.
- **Logging**: Avoid passing a logger with a file path in cloud; use stdout or your platform’s logging (e.g. CloudWatch). Pass `logger=None` to use the default (no file).
- **Secrets**: Load client_id, client_secret, refresh_token, and DB credentials from environment or a secret manager; do not hardcode in code.
- **Retries**: The SDK does not retry failed HTTP calls. Implement retry/backoff in application code if needed.
- **Sync only**: All I/O is synchronous (no async API). Suitable for traditional servers/workers; in serverless, consider process model and timeouts.

## Extending the SDK

- **Custom token store**: Implement **TokenStore** (find_token, save_token, delete_token, get_tokens, delete_tokens, find_token_by_id) and pass an instance to **Initializer.initialize(..., store=your_store)**.
- **New API operations**: Follow the pattern in an existing `*_operations.py`: create **CommonAPIHandler**, set path/method/params/headers/body, call **Utility.get_fields** if needed, then **handler_instance.api_call(ResponseHandler.__module__, 'application/json')** with the correct response handler module for that feature.
- **Constants**: Add new constants in `zohocrmsdk/src/com/zoho/crm/api/util/constants.py` if needed for new options or error messages.

## Quick reference: minimal initialization and one call

```python
from zohocrmsdk.src.com.zoho.api.authenticator import OAuthToken
from zohocrmsdk.src.com.zoho.crm.api import Initializer
from zohocrmsdk.src.com.zoho.crm.api.dc import USDataCenter
from zohocrmsdk.src.com.zoho.crm.api.record import RecordOperations
from zohocrmsdk.src.com.zoho.crm.api import ParameterMap, HeaderMap

environment = USDataCenter.PRODUCTION()
token = OAuthToken(client_id="...", client_secret="...", refresh_token="...")
Initializer.initialize(environment, token)

record_operations = RecordOperations("Leads")
response = record_operations.get_records(ParameterMap(), HeaderMap())
# response.get_object() -> ResponseWrapper or APIException
```

For more detail (OAuth registration, DB persistence, threading, class hierarchy), see **versions/1.0.0/README.md** (or the README in the version you use).
