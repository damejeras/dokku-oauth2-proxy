# dokku-oauth2-proxy

OAuth2 authentication proxy plugin for Dokku. Provides Google OAuth2 authentication for your Dokku apps using a single shared [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) instance.

## Features

- Single shared oauth2-proxy container for all apps
- Per-app OAuth2 protection enable/disable
- Global email allowlist for access control
- Shared cookie domain for SSO across apps
- Automatic nginx integration using `auth_request`

## Requirements

- Dokku 0.4.0+
- Docker (already required by Dokku)
- Google OAuth2 application credentials

## Installation

```bash
dokku plugin:install https://github.com/dokku/dokku-oauth2-proxy.git oauth2
```

This will:
- Pull the oauth2-proxy Docker image
- Initialize plugin storage

## Setup

### 1. Create Google OAuth2 Application

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Navigate to **APIs & Services** → **Credentials**
3. Click **Create Credentials** → **OAuth 2.0 Client ID**
4. Select **Web application** as application type
5. Configure authorized redirect URIs:
   - Add your auth domain callback: `https://auth.example.com/oauth2/callback`
   - Replace `auth.example.com` with your actual auth domain
   - **Note**: You only need ONE redirect URI for all your apps!
6. Note the **Client ID** and **Client Secret**

### 2. Configure OAuth2 Proxy

```bash
dokku oauth2-proxy:setup <client-id> <client-secret> <auth-domain>
```

Example:
```bash
dokku oauth2-proxy:setup \
  "123456789.apps.googleusercontent.com" \
  "YOUR_CLIENT_SECRET_HERE" \
  "auth.example.com"
```

Parameters:
- `client-id`: Google OAuth2 Client ID
- `client-secret`: Google OAuth2 Client Secret
- `auth-domain`: Subdomain where OAuth2 flow will be handled (e.g., `auth.example.com`)

This command will:
- Store your OAuth2 credentials
- Generate a secure cookie secret
- Start the oauth2-proxy app
- Create nginx configuration for the auth domain


## Usage

### Enable OAuth2 for an App

```bash
dokku oauth2-proxy:enable myapp
```

This adds OAuth2 authentication to your app. By default, any authenticated Google user can access the app.

### Restrict Access by Email

Manage the allowlist file directly on the Dokku host:

Allowlist file path:
- `/var/lib/dokku/data/storage/oauth2-proxy/allowed_emails`

Syntax:
- One email per line (lowercase recommended)

Example contents:
```
user@example.com
admin@example.com
```

### Disable OAuth2 for an App

```bash
dokku oauth2-proxy:disable myapp
```

## How It Works

This plugin uses a **centralized authentication architecture**:

1. **Cookie Validation Only**: Protected apps only validate authentication cookies - they do NOT handle OAuth flows
2. **Centralized Auth Domain**: All OAuth authentication happens at a single auth domain (e.g., `auth.example.com`)
3. **401 Response**: When a user visits a protected app without a valid cookie, nginx returns HTTP 401
4. **App-Controlled UX**: Your application decides how to handle the 401 (redirect to auth domain, show login page, etc.)
5. **Google OAuth Flow**: When users visit the auth domain, they complete Google OAuth and get a cookie
6. **Cookie Sharing**: The authentication cookie is set at the parent domain (e.g., `.example.com`)
7. **SSO**: The cookie is accessible across all subdomains, providing single sign-on
8. **Email Validation**: If configured, oauth2-proxy validates the user's email against the global allowlist file

### Benefits of This Architecture

- **Clean Separation**: Apps don't need OAuth endpoints - they just validate cookies
- **API-Friendly**: Returns 401 instead of HTML redirects, works great with JSON APIs
- **Single Callback URL**: Only one redirect URI needed in Google OAuth (the auth domain)
- **Flexible UX**: Each app can handle authentication however it wants

## Architecture

```
                                    ┌─────────────────┐
                                    │  Google OAuth   │
                                    └────────┬────────┘
                                             │
                                             ▼
┌──────────────┐              ┌──────────────────────┐
│ Protected    │   401        │  Auth Domain         │
│ App          ├──────────────▶  auth.example.com    │
│ app.example  │              │  (OAuth flow)        │
└──────┬───────┘              └──────────┬───────────┘
       │                                  │
       │ Cookie validation                │ Set cookie
       │ via auth_request                 │ at .example.com
       │                                  │
       └────────────┬─────────────────────┘
                    ▼
         ┌─────────────────────┐
         │  oauth2-proxy       │
         │  (127.0.0.1:4180)   │
         └─────────────────────┘
```

**Components:**
- **oauth2-proxy**: Runs as a Docker container on `127.0.0.1:4180`
- **Auth Domain**: Dedicated subdomain that handles OAuth flow and sets cookies
- **Protected Apps**: Validate cookies via nginx `auth_request`, return 401 if invalid
- **Cookies**: Shared across subdomains for SSO

## Important Notes

### Redirect URI Configuration

With the centralized architecture, you only need to configure **ONE** redirect URI in your Google OAuth application:

```
https://auth.example.com/oauth2/callback
```

Where `auth.example.com` is your auth domain configured during setup. This is much simpler than the traditional approach which requires adding each app's callback URL.

### Cookie Domain

The cookie domain must:
- Start with a dot (e.g., `.example.com`)
- Be a parent domain of all your apps
- Match the domain structure where your apps are deployed

All apps must be on subdomains of this parent domain for SSO to work.

### Email Validation

The plugin supports:
- Exact email matches: `user@example.com`

The allowlist is stored on the Dokku host at:
- `/var/lib/dokku/data/storage/oauth2-proxy/allowed_emails`

Does not support:
- Domain wildcards
- Complex regex patterns
- Google Workspace groups

### Application Integration

#### Handling Authentication

When OAuth2 protection is enabled, your app will receive:
- **HTTP 401**: If the user is not authenticated (no valid cookie)
- **HTTP 403**: If the user is authenticated but not in the allowed emails list
- **Request with headers**: If the user is authenticated and authorized

Your application should handle 401 responses by redirecting users to the auth domain:

```javascript
// Example: JavaScript fetch with error handling
fetch('/api/data')
  .then(response => {
    if (response.status === 401) {
      // Redirect to auth domain, then back to current page
      window.location.href = `https://auth.example.com/oauth2/start?rd=${encodeURIComponent(window.location.href)}`;
      return;
    }
    return response.json();
  })
```

## Commands Reference

| Command | Description |
|---------|-------------|
| `oauth2-proxy:setup <client-id> <client-secret> <auth-domain>` | Configure global OAuth2 settings |
| `oauth2-proxy:enable <app>` | Enable OAuth2 for an app |
| `oauth2-proxy:disable <app>` | Disable OAuth2 for an app |

## Troubleshooting

### OAuth2 not working after setup

Check if the container is running:
```bash
docker ps | grep dokku.oauth2-proxy
```

Check container logs:
```bash
docker logs dokku.oauth2-proxy
```

### 401 Unauthorized errors

- Verify your Google OAuth credentials are correct
- Check that the redirect URI is configured in Google Cloud Console
- Ensure the cookie domain is correct

### 403 Forbidden errors

- Check allowlist contents at `/var/lib/dokku/data/storage/oauth2-proxy/allowed_emails`
- Verify the authenticated user's email matches the allowlist

### Nginx errors

Check nginx error logs:
```bash
tail -f /var/log/nginx/error.log
```

Verify nginx configuration:
```bash
nginx -t
```

### Container won't start

View container logs:
```bash
docker logs dokku.oauth2-proxy
```

Common issues:
- Invalid OAuth credentials
- Port 4180 already in use
- Malformed cookie secret

## Security Considerations

- **Cookie Secret**: Automatically generated and stored securely. Do not share or expose.
- **HTTPS Required**: OAuth2 authentication requires HTTPS in production. Use the Dokku Let's Encrypt plugin to enable TLS on the auth app and protected apps.
- **Client Secret**: Stored in plugin properties. Protect access to `/var/lib/dokku/plugins/oauth2/`.
- **Session Duration**: Sessions are managed by oauth2-proxy. Configure session timeout via container environment variables if needed.

## Limitations

- **Single OAuth Provider**: All apps must use the same OAuth provider (Google)
- **Subdomain Requirement**: All apps must be on subdomains of the same parent domain
- **Manual Redirect URI Management**: You must manually add the auth domain redirect URI to Google Cloud Console
- **Simple Email Validation**: No support for complex patterns or group-based access control
- **Global Allowlist**: All protected apps share the same allowlist file

## Contributing

Contributions are welcome! Please submit issues and pull requests to the repository.

## License

MIT License

## Credits

Based on the [dokku-http-auth](https://github.com/dokku/dokku-http-auth) plugin pattern.

Uses [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) for OAuth2 authentication.
