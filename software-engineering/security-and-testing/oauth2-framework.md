# What is OAuth 2.0

OAuth 2.0 provides a framework for **granting limited access** to resources without **sharing credentials**. It doesn’t confirm the user’s identity by itself. That’s why people often combine OAuth 2.0 with OpenID Connect (OIDC), which adds an identity layer on top of OAuth 2.0 for authentication.

- It’s not an authentication protocol by itself (though often used with OpenID Connect for authentication).

- It defines how a client application can request and obtain an access token from an authorization server, which can then be used to access protected resources on behalf of the user.

> OAuth2 is specifically designed for **authorization**. It defines how a client can access a user’s resources, typically by obtaining tokens. OAuth2 usually uses JWTs, but it doesn’t have to.

``` mermaid
        sequenceDiagram
    participant RO as Resource Owner (User)
    participant Client as Client App
    participant AS as Authorization Server
    participant RS as Resource Server
    Client->>RO: Authorization Request (redirect to login)
    RO->>AS: Authenticate + Grant Permission
    AS-->>Client: Authorization Grant (code)
    Client->>AS: Exchange Grant for Access Token
    AS-->>Client: Access Token (JWT)
    Client->>RS: API Request with Access Token
    RS-->>Client: Protected Resource
```

| Concept | Purpose | Example |
|----|----|----|
| **Authentication** | Verifying *who* the user is | “Log in with email/password.” |
| **Authorization** | Determining *what the user/app is allowed to do* | “Allow app to access your Google Drive files.” |
| **JWT** | A token *format* used to store claims | `{"sub":`` ``"123",`` ``"role":`` ``"admin"}` |
| **OAuth2** | A framework for authorization | Google login, GitHub OAuth apps |

## Main Roles

- `Resource`` ``Owner`: The user who owns the data.

- `Client`: The application requesting access.

- `Authorization`` ``Server`: Issues tokens after verifying the user.

- `Resource`` ``Server`: Hosts the protected resources (e.g., APIs).

## Core Flow

- `Authorization`` ``Request`: The client asks the user for permission.

- `Authorization`` ``Grant`: The user approves and the client gets a grant (e.g., code).

- `Access`` ``Token`` ``Exchange`: The client exchanges the grant for an access token.

- `API`` ``Access`: The client uses the token to call the resource server.

``` python
import requests
from requests_oauthlib import OAuth2Session

# OAuth 2.0 client details
client_id = "YOUR_CLIENT_ID"
client_secret = "YOUR_CLIENT_SECRET"
authorization_base_url = "https://example.com/oauth/authorize"
token_url = "https://example.com/oauth/token"
redirect_uri = "https://localhost/callback"

# Step 1: Redirect user to provider for authorization
oauth = OAuth2Session(client_id, redirect_uri=redirect_uri)
authorization_url, state = oauth.authorization_url(authorization_base_url)
print("Please go to this URL and authorize:", authorization_url)

# Step 2: Get the authorization response URL from user
redirect_response = input("Paste the full redirect URL here: ")

# Step 3: Fetch the access token
token = oauth.fetch_token(token_url, authorization_response=redirect_response,
                          client_secret=client_secret)
print("Access Token:", token)

# Step 4: Access a protected resource
protected_url = "https://example.com/api/user"
response = oauth.get(protected_url)
print("Protected Resource:", response.json())
```

```
Please go to this URL and authorize: https://example.com/oauth/authorize?response_type=code&client_id=YOUR_CLIENT_ID&redirect_uri=https%3A%2F%2Flocalhost%2Fcallback&state=nukc5oHUnqmZbbWoWtaLOO39noQTbA



---------------------------------------------------------------------------

MismatchingStateError                     Traceback (most recent call last)

Cell In[3], line 20
     17 redirect_response = input("Paste the full redirect URL here: ")
     19 # Step 3: Fetch the access token
---> 20 token = oauth.fetch_token(token_url, authorization_response=redirect_response,
     21                           client_secret=client_secret)
     22 print("Access Token:", token)
     24 # Step 4: Access a protected resource


File ~/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages/requests_oauthlib/oauth2_session.py:271, in OAuth2Session.fetch_token(self, token_url, code, authorization_response, body, auth, username, password, method, force_querystring, timeout, headers, verify, proxies, include_client_id, client_secret, cert, **kwargs)
    268     raise InsecureTransportError()
    270 if not code and authorization_response:
--> 271     self._client.parse_request_uri_response(
    272         authorization_response, state=self._state
    273     )
    274     code = self._client.code
    275 elif not code and isinstance(self._client, WebApplicationClient):


File ~/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages/oauthlib/oauth2/rfc6749/clients/web_application.py:220, in WebApplicationClient.parse_request_uri_response(self, uri, state)
    176 def parse_request_uri_response(self, uri, state=None):
    177     """Parse the URI query for code and state.
    178
    179     If the resource owner grants the access request, the authorization
   (...)    218         oauthlib.oauth2.rfc6749.errors.MismatchingStateError
    219     """
--> 220     response = parse_authorization_code_response(uri, state=state)
    221     self.populate_code_attributes(response)
    222     return response


File ~/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages/oauthlib/oauth2/rfc6749/parameters.py:277, in parse_authorization_code_response(uri, state)
    274 params = dict(urlparse.parse_qsl(query))
    276 if state and params.get('state') != state:
--> 277     raise MismatchingStateError()
    279 if 'error' in params:
    280     raise_from_error(params.get('error'), params)


MismatchingStateError: (mismatching_state) CSRF Warning! State not equal in request and response.
```

## Step-by-Step Minimal OAuth2 Server

``` python
```

```
Collecting authlib
  Downloading authlib-1.6.5-py2.py3-none-any.whl.metadata (9.8 kB)
Requirement already satisfied: starlette in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (0.48.0)
Requirement already satisfied: uvicorn in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (0.38.0)
Requirement already satisfied: cryptography in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from authlib) (46.0.3)
Requirement already satisfied: anyio<5,>=3.6.2 in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from starlette) (4.11.0)
Requirement already satisfied: typing-extensions>=4.10.0 in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from starlette) (4.15.0)
Requirement already satisfied: idna>=2.8 in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from anyio<5,>=3.6.2->starlette) (3.11)
Requirement already satisfied: sniffio>=1.1 in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from anyio<5,>=3.6.2->starlette) (1.3.1)
Requirement already satisfied: click>=7.0 in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from uvicorn) (8.3.0)
Requirement already satisfied: h11>=0.8 in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from uvicorn) (0.16.0)
Requirement already satisfied: cffi>=2.0.0 in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from cryptography->authlib) (2.0.0)
Requirement already satisfied: pycparser in /Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages (from cffi>=2.0.0->cryptography->authlib) (2.23)
Downloading authlib-1.6.5-py2.py3-none-any.whl (243 kB)
Installing collected packages: authlib
Successfully installed authlib-1.6.5
```

------------------------------------------------------------------------

## The Future of Authentication: Passkeys (2026 Update)

While JWT and OAuth2 remain essential for API authentication, the industry is rapidly moving toward **passkeys** for user-facing login flows.

### What Are Passkeys?

Passkeys are a **passwordless authentication** method based on the FIDO2/WebAuthn standard. Instead of typing a password, users authenticate with their device’s biometric sensor (fingerprint, face) or screen lock (PIN).

``` mermaid
        sequenceDiagram
    participant User as User (Browser)
    participant Server as Your API
    participant Auth as Device Authenticator

    Note over User,Auth: Registration (one-time)
    User->>Server: Request registration challenge
    Server->>User: Send challenge + options
    User->>Auth: Create credential (biometric)
    Auth->>User: Public key + credential ID
    User->>Server: Send public key (stored server-side)

    Note over User,Auth: Login (every time)
    User->>Server: Request login challenge
    Server->>User: Send challenge
    User->>Auth: Sign challenge (biometric)
    Auth->>User: Signed assertion
    User->>Server: Send signed assertion
    Server->>Server: Verify with stored public key
    Server->>User: JWT access token
```

### Why Passkeys Matter

- **NIST SP 800-63-4 (July 2025)** recognizes passkeys as AAL2-compliant (same level as hardware tokens)

- **SMS OTP no longer satisfies AAL2** — phishing and SIM-swap attacks made it insufficient

- Supported on iOS 16+, Android 9+, Windows 11, all modern browsers

- Synced across devices via iCloud Keychain / Google Password Manager

### How Passkeys Coexist with JWT/OAuth2

Passkeys replace the **login step** (how users prove their identity), not the **API authentication** mechanism:

| Layer | Technology | Role |
|----|----|----|
| **User login** | Passkeys (FIDO2/WebAuthn) | Prove identity without passwords |
| **API authentication** | JWT tokens | Stateless request authentication |
| **Delegated access** | OAuth2 + OIDC | Third-party app authorization |

After a successful passkey login, your server still issues a JWT access token for API requests — the token flow remains the same.

### Python Libraries

- [`py_webauthn`](https://github.com/duo-labs/py_webauthn) — WebAuthn server-side library for Python

- FastAPI integration: use `py_webauthn` for registration/login endpoints, then issue JWT tokens as before

You don’t need to implement passkeys from scratch for every project. Identity providers like Auth0, Okta, and Firebase Authentication already support passkeys. For most projects, delegate authentication to an identity provider and focus on API authorization with JWT.
