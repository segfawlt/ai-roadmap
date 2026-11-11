# **OAuth2 Python Assignment**

## **Learning Goals**

Students will:

1.  Understand OAuth2 Authorization Code Flow

2.  Build a local HTTP server using only `http.server`

3.  Manually construct authorization URLs

4.  Exchange authorization codes for access tokens

5.  Call protected APIs using tokens

## **Assignment Specification**

You must implement OAuth2 login with **GitHub** using pure Python.

**Requirements:**

1.  Start a local HTTP server on port **8000**

2.  Open the browser automatically to the OAuth2 authorization URL

3.  Receive the `code` parameter on redirect

4.  Exchange the code for an access token

5.  Call the GitHub API `/user` endpoint

6.  Print the user profile data to the console

## **Starter Code (Complete Minimal Example)**

Save this as **oauth2_github.py**.

``` python
import http.server
import socketserver
import webbrowser
import requests
import threading
import urllib.parse
import os

CLIENT_ID = os.getenv("CLIENT_ID") or "YOUR_CLIENT_ID"
CLIENT_SECRET = os.getenv("CLIENT_SECRET") or "YOUR_CLIENT_SECRET"

REDIRECT_URI = "http://localhost:8000/callback"
AUTH_URL = "https://github.com/login/oauth/authorize"
TOKEN_URL = "https://github.com/login/oauth/access_token"
USER_API_URL = "https://api.github.com/user"

oauth_code = None


class OAuthHandler(http.server.SimpleHTTPRequestHandler):
    """Handles the /callback request and extracts the code."""

    def do_GET(self):
        global oauth_code

        parsed = urllib.parse.urlparse(self.path)
        path = parsed.path

        if path == "/callback":
            query = urllib.parse.parse_qs(parsed.query)
            oauth_code = query.get("code", [None])[0]

            self.send_response(200)
            self.send_header("Content-type", "text/html")
            self.end_headers()
            self.wfile.write(b"<h1>Authentication complete! You can close this tab.</h1>")
        else:
            self.send_error(404)


def start_server():
    with socketserver.TCPServer(("localhost", 8000), OAuthHandler) as httpd:
        print("Server running on http://localhost:8000 ...")
        httpd.serve_forever()


def exchange_code_for_token(code):
    response = requests.post(
        TOKEN_URL,
        headers={"Accept": "application/json"},
        data={
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET,
            "code": code,
            "redirect_uri": REDIRECT_URI
        }
    )
    response.raise_for_status()
    return response.json()["access_token"]


def get_user_profile(token):
    response = requests.get(
        USER_API_URL,
        headers={"Authorization": f"Bearer {token}"}
    )
    response.raise_for_status()
    return response.json()


def main():
    # Start local server in background thread
    server_thread = threading.Thread(target=start_server, daemon=True)
    server_thread.start()

    # Build the authorization URL
    params = urllib.parse.urlencode({
        "client_id": CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "scope": "read:user",
        "response_type": "code"
    })
    auth_url = f"{AUTH_URL}?{params}"

    print("Opening browser for authentication...")
    webbrowser.open(auth_url)

    # Wait for OAuth code
    while oauth_code is None:
        pass  # simple wait loop

    print(f"Authorization code received: {oauth_code}")

    # Exchange code for token
    token = exchange_code_for_token(oauth_code)
    print(f"Access token received: {token}")

    # Fetch user profile
    profile = get_user_profile(token)
    print("\n=== GitHub Profile ===")
    print(profile)


if __name__ == "__main__":
    main()
```

------------------------------------------------------------------------

## **Assignment Tasks for Students**

### **1. Setup**

- Create a GitHub OAuth app

- Set redirect URL: `http://localhost:8000/callback`

- Export environment variables:

``` bash
export CLIENT_ID=your_id
export CLIENT_SECRET=your_secret
```

------------------------------------------------------------------------

### **2. Implement OAuth2 Flow**

Students must:

✓ Start a local server ✓ Listen for `/callback` ✓ Parse query parameters ✓ Open browser to GitHub OAuth ✓ Capture authorization code ✓ Send POST to `https://github.com/login/oauth/access_token` ✓ Store & print the access token ✓ Call `/user` with token ✓ Display profile data

------------------------------------------------------------------------

### **3. Written Questions**

1.  Why is the redirect URI necessary?

2.  Why is the authorization code exchanged for an access token?

3.  Why is a local HTTP server needed?

4.  What security risks exist when storing client secrets locally?

5.  Explain the purpose of scopes.
