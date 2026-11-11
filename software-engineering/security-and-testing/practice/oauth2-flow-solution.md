# **FULL SOLUTION — OAuth2 in Pure Python (GitHub Authorization Code Flow)**

**No Flask, no FastAPI, only built-in modules + `requests`.**

------------------------------------------------------------------------

## 1. **Solution Code**

Save as **solution_oauth2_pure_python.py**

``` python
"""
OAuth2 Authorization Code Flow (Pure Python Version)
----------------------------------------------------
This script demonstrates OAuth2 login with GitHub using only:
- http.server
- socketserver
- urllib.parse
- webbrowser
- requests

No web frameworks (Flask/FastAPI) are used.
"""

import http.server
import socketserver
import urllib.parse
import webbrowser
import threading
import requests
import os
import sys


# -------------------------------
# Configuration
# -------------------------------
CLIENT_ID = os.getenv("CLIENT_ID") or "REPLACE_ME"
CLIENT_SECRET = os.getenv("CLIENT_SECRET") or "REPLACE_ME"

if CLIENT_ID == "REPLACE_ME":
    print("Error: Please set CLIENT_ID and CLIENT_SECRET environment variables.")
    sys.exit(1)

AUTH_URL = "https://github.com/login/oauth/authorize"
TOKEN_URL = "https://github.com/login/oauth/access_token"
USER_URL = "https://api.github.com/user"
REDIRECT_URI = "http://localhost:8000/callback"

oauth_code = None  # will store the authorization code from callback


# -------------------------------
# Step 1 — Local HTTP Callback Server
# -------------------------------
class OAuthCallbackHandler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        global oauth_code

        parsed = urllib.parse.urlparse(self.path)

        if parsed.path == "/callback":
            params = urllib.parse.parse_qs(parsed.query)
            oauth_code = params.get("code", [None])[0]

            # Show confirmation page
            self.send_response(200)
            self.send_header("Content-Type", "text/html")
            self.end_headers()
            self.wfile.write(b"<h1>Login Successful!</h1>You can close this tab.")

        else:
            self.send_error(404)


def start_callback_server():
    with socketserver.TCPServer(("localhost", 8000), OAuthCallbackHandler) as httpd:
        print("Callback server running on http://localhost:8000 ...")
        httpd.serve_forever()


# -------------------------------
# Step 2 — Exchange Code for Token
# -------------------------------
def exchange_code_for_token(code: str) -> str:
    """Send POST request to GitHub to trade code for access token"""
    response = requests.post(
        TOKEN_URL,
        headers={"Accept": "application/json"},
        data={
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
            "code": code,
            "redirect_uri": REDIRECT_URI,
        }
    )
    response.raise_for_status()
    json_data = response.json()
    return json_data["access_token"]


# -------------------------------
# Step 3 — Fetch Protected Resource
# -------------------------------
def get_github_profile(token: str) -> dict:
    """Call the GitHub API with the provided access token"""
    response = requests.get(
        USER_URL,
        headers={"Authorization": f"Bearer {token}"}
    )
    response.raise_for_status()
    return response.json()


# -------------------------------
# Main Program
# -------------------------------
def main():
    global oauth_code

    # Start callback server in background thread
    server_thread = threading.Thread(target=start_callback_server, daemon=True)
    server_thread.start()

    # Construct authorization URL
    params = urllib.parse.urlencode({
        "client_id": CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "scope": "read:user",
        "response_type": "code",
    })
    url = f"{AUTH_URL}?{params}"

    print("\nOpening your browser to authenticate with GitHub...")
    webbrowser.open(url)

    # Wait for user authorization
    print("Waiting for authorization...")
    while oauth_code is None:
        pass

    print(f"\nAuthorization code received: {oauth_code}")

    # Step 2: Exchange for access token
    token = exchange_code_for_token(oauth_code)
    print(f"Access token received: {token}")

    # Step 3: Call API
    profile = get_github_profile(token)

    print("\n=== GitHub User Profile ===")
    for k, v in profile.items():
        print(f"{k}: {v}")

    print("\nDone.")


if __name__ == "__main__":
    main()
```

------------------------------------------------------------------------

## 2. **Explanation of the Solution**

### **Step 1 — Start Local HTTP Server**

We use `http.server` + `socketserver` to listen for GitHub’s redirect:

```
http://localhost:8000/callback?code=xxxx
```

The handler extracts the `code` and shows a simple HTML message.

------------------------------------------------------------------------

### **Step 2 — Redirect User to GitHub**

Python opens the browser automatically:

``` python
webbrowser.open(auth_url)
```

------------------------------------------------------------------------

### **Step 3 — Receive Authorization Code**

Once the user approves the login, GitHub redirects to:

```
/callback?code=12345
```

The Python script captures this.

------------------------------------------------------------------------

### **Step 4 — Exchange Code for an Access Token**

``` python
requests.post(TOKEN_URL, ...)
```

GitHub returns:

``` json
{
  "access_token": "xxxx",
  "token_type": "bearer"
}
```

------------------------------------------------------------------------

### **Step 5 — Use Token to Get Profile Information**

``` python
requests.get(USER_URL, headers={"Authorization": f"Bearer {token}"})
```

This returns user’s login, name, avatar URL, etc.

------------------------------------------------------------------------

## 3. **Solutions to Written Questions**

### **1. Why is the redirect URI necessary?**

It tells the OAuth provider (GitHub) *where to send the authorization code* after the user approves access.

------------------------------------------------------------------------

### **2. Why do we exchange the authorization code for an access token?**

Because the code is just proof of authorization — the real API key is the **access token**. This two-step flow increases security.

------------------------------------------------------------------------

### **3. Why do we need a local HTTP server?**

OAuth providers must redirect the user to a URL. Without a local server, your script cannot receive:

```
/callback?code=xxxx
```

------------------------------------------------------------------------

### **4. What security risks exist with storing client secrets locally?**

- Secrets can be read by any user on the system

- Checked into Git repos by mistake

- Malware can steal them

- They cannot be revoked per-machine easily

------------------------------------------------------------------------

### **5. Purpose of OAuth scopes?**

Scopes limit what the access token can do. Example: `read:user` allows reading profile info, but not repositories.
