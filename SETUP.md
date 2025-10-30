# OAuth2-Proxy Setup Guide
## Google Authentication for @parity.io emails

This setup protects your internal Docker application with Google OAuth, only allowing users with `@parity.io` email addresses.

---

## Prerequisites

- Docker and Docker Compose installed
- Google Cloud Platform account
- Your internal application Docker image

---

## Step 1: Configure Google OAuth

1. **Go to [Google Cloud Console](https://console.cloud.google.com/)**

2. **Create a new project** (or select existing)

3. **Create OAuth 2.0 Credentials**:
   - Go to "APIs & Services" → "Credentials"
   - Click "Create Credentials" → "OAuth client ID"
   - Application type: "Web application"
   - Name: `oauth2-proxy`
   - **Authorized redirect URIs**: Add these URLs:
     - `http://localhost:4180/oauth2/callback` (for local testing)
     - `https://your-domain.com/oauth2/callback` (for production)
   - Click "Create"
   - **Save the Client ID and Client Secret** - you'll need these

---

## Step 2: Generate Cookie Secret

**Use Python to generate a properly formatted cookie secret** (base64url encoded 32 random bytes):

```bash
python -c 'import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())'
```

**Note**: The output will be 44 characters long - this is correct! OAuth2-Proxy will decode it to get the 32 bytes needed for AES encryption.

---

## Step 3: Configure Environment Variables

1. **Copy the example environment file**:
   ```bash
   cp .env.example .env
   ```

2. **Edit `.env`** and fill in your credentials:
   ```bash
   OAUTH2_PROXY_CLIENT_ID=your-client-id.apps.googleusercontent.com
   OAUTH2_PROXY_CLIENT_SECRET=your-client-secret
   OAUTH2_PROXY_COOKIE_SECRET=your-generated-cookie-secret
   OAUTH2_PROXY_REDIRECT_URL=http://localhost:4180/oauth2/callback
   ```

---

## Step 4: Update Docker Compose

Edit `docker-compose.yml` and update the `internal-app` service with your actual application:

```yaml
internal-app:
  image: your-actual-app-image:tag  # Replace this
  # Add any environment variables your app needs
  # Add volumes, ports (internal only), etc.
```

---

## Step 5: Start the Services

```bash
docker-compose up -d
```

Check logs:
```bash
docker-compose logs -f oauth2-proxy
```

---

## Step 6: Test the Setup

1. **Open your browser** and go to: `http://localhost:4180`

2. **You should be redirected** to Google sign-in

3. **Sign in with a @parity.io email**

4. **After authentication**, you'll be redirected to your internal app

5. **Test with non-@parity.io email** - should be rejected

---

## Configuration Details

### Email Restriction
The proxy only allows `@parity.io` emails (configured in `oauth2-proxy.cfg`):
```
email_domains = ["parity.io"]
```

### Upstream App
The proxy forwards authenticated requests to your internal app:
```
upstreams = ["http://internal-app:8080"]
```

Update the port (`:8080`) if your app uses a different port.

---

## Production Deployment

For production, you need to:

1. **Use HTTPS** - Update config:
   ```
   cookie_secure = true
   redirect_url = "https://your-domain.com/oauth2/callback"
   ```

2. **Update Google OAuth redirect URL** to your production domain

3. **Use a reverse proxy** (nginx/traefik) with SSL certificates

4. **Set proper cookie domain**:
   ```
   cookie_domain = ".your-domain.com"
   ```

---

## Troubleshooting

### "Error redeeming code" or "Invalid redirect URI"
- Check that the redirect URL in `.env` matches exactly what's in Google Cloud Console
- Ensure the URL includes the protocol (`http://` or `https://`)

### "403 Permission Denied"
- User's email is not `@parity.io`
- Check logs: `docker-compose logs oauth2-proxy`

### Can't access the app
- Ensure `internal-app` service is running: `docker-compose ps`
- Check the upstream URL in `oauth2-proxy.cfg` matches your app's port
- Verify network connectivity: `docker-compose exec oauth2-proxy wget -O- http://internal-app:8080`

### Cookie issues
- Ensure `OAUTH2_PROXY_COOKIE_SECRET` is set and valid (base64 encoded, 32 bytes)
- For localhost testing, `cookie_secure = false` is fine
- For production with HTTPS, set `cookie_secure = true`

---

## Additional Configuration

### Add more allowed domains
Edit `oauth2-proxy.cfg`:
```
email_domains = [
  "parity.io",
  "another-domain.com"
]
```

### Change session duration
Edit `oauth2-proxy.cfg`:
```
cookie_expire = "168h"  # 7 days
cookie_refresh = "1h"   # Refresh token every hour
```

### Add specific email addresses
Instead of domain-based, allow specific emails:
```
authenticated_emails_file = "/path/to/emails.txt"
```

Create `emails.txt`:
```
user1@parity.io
user2@parity.io
```

---

## Architecture

```
User → OAuth2-Proxy (port 4180) → Google OAuth
         ↓ (authenticated @parity.io user)
       Internal App (not exposed directly)
```

The internal app is **not accessible** without going through the OAuth2-Proxy, ensuring all access is authenticated.

---

## Useful Commands

```bash
# Start services
docker-compose up -d

# View logs
docker-compose logs -f oauth2-proxy

# Restart proxy after config changes
docker-compose restart oauth2-proxy

# Stop all services
docker-compose down

# Generate new cookie secret
openssl rand -base64 32
```

---

## Security Notes

- **Never commit `.env`** - add it to `.gitignore`
- **Rotate secrets regularly** - especially `OAUTH2_PROXY_CLIENT_SECRET` and `OAUTH2_PROXY_COOKIE_SECRET`
- **Use HTTPS in production** - required for secure cookies
- **Restrict OAuth client** - in Google Console, restrict to specific domains if possible
- **Monitor access logs** - review who's accessing your application

---

## References

- [OAuth2-Proxy Documentation](https://oauth2-proxy.github.io/oauth2-proxy/)
- [Google OAuth Setup](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/google)
- [Configuration Options](https://oauth2-proxy.github.io/oauth2-proxy/configuration/overview)
