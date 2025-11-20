# 🔒 Security Guide / Guide de Sécurité

## 📋 Overview / Vue d'ensemble

FlowCDN implements multiple layers of security including HTTPS/SSL, CORS, Helmet security headers, and API key authentication.

FlowCDN implémente plusieurs couches de sécurité incluant HTTPS/SSL, CORS, en-têtes de sécurité Helmet, et authentification par clés API.

---

## 🔐 HTTPS/SSL

### Current Status / État Actuel

**⚠️ HTTPS is NOT currently enabled in production**
**⚠️ HTTPS n'est PAS actuellement activé en production**

The FlowCDN MVP currently runs in **HTTP-only mode**. HTTPS support is implemented in the codebase but not yet activated.

Le MVP FlowCDN fonctionne actuellement en **mode HTTP uniquement**. Le support HTTPS est implémenté dans le code mais pas encore activé.

**Why HTTP for now? / Pourquoi HTTP pour l'instant ?**
- Development and testing phase / Phase de développement et tests
- Waiting for production domain configuration / En attente de la configuration du domaine de production
- Let's Encrypt will be used in production / Let's Encrypt sera utilisé en production

---

### Development / Développement

**Recommended: Use HTTP for local development**
**Recommandé : Utiliser HTTP pour le développement local**

```bash
# Default configuration (HTTP only)
# Configuration par défaut (HTTP uniquement)
npm run dev
# Server runs on http://localhost:3000
```

**Optional: Test HTTPS locally (requires admin rights)**
**Optionnel : Tester HTTPS localement (nécessite droits admin)**

If you need to test HTTPS-specific features locally:

Si vous devez tester des fonctionnalités spécifiques à HTTPS localement :

```powershell
# 1. Run PowerShell as Administrator
# 1. Exécuter PowerShell en tant qu'Administrateur

# 2. Install mkcert
choco install mkcert -y

# 3. Install local CA
mkcert -install

# 4. Generate certificates
mkcert -key-file certs/key.pem -cert-file certs/cert.pem localhost 127.0.0.1 ::1

# 5. Enable HTTPS in .env
# 5. Activer HTTPS dans .env
# ENABLE_HTTPS=true
# SSL_KEY_PATH=./certs/key.pem
# SSL_CERT_PATH=./certs/cert.pem
```

---

### Production (Planned) / Production (Prévu)

**🎯 Let's Encrypt will be used for production SSL certificates**
**🎯 Let's Encrypt sera utilisé pour les certificats SSL de production**

Let's Encrypt provides free, automated SSL certificates with auto-renewal.

Let's Encrypt fournit des certificats SSL gratuits et automatisés avec renouvellement automatique.

#### Setup Steps / Étapes de Configuration

**1. Install Certbot**
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install certbot python3-certbot-nginx

# CentOS/RHEL
sudo yum install certbot python3-certbot-nginx
```

**2. Generate Certificate**
```bash
# Replace yourdomain.com with your actual domain
# Remplacer yourdomain.com par votre domaine réel
sudo certbot certonly --nginx -d api.flowcdn.com

# Or with standalone mode (if no web server running)
# Ou en mode standalone (si aucun serveur web en cours)
sudo certbot certonly --standalone -d api.flowcdn.com
```

**3. Configure FlowCDN**
```env
# .env
NODE_ENV=production
ENABLE_HTTPS=true
SSL_KEY_PATH=/etc/letsencrypt/live/api.flowcdn.com/privkey.pem
SSL_CERT_PATH=/etc/letsencrypt/live/api.flowcdn.com/fullchain.pem
```

**4. Setup Auto-Renewal**
```bash
# Test renewal
sudo certbot renew --dry-run

# Add to crontab for automatic renewal
sudo crontab -e
# Add this line:
0 0 * * * certbot renew --quiet --post-hook "systemctl restart flowcdn"
```

**5. Restart FlowCDN**
```bash
# With systemd
sudo systemctl restart flowcdn

# With PM2
pm2 restart flowcdn

# With Docker
docker compose restart
```

#### Nginx Reverse Proxy (Recommended)

For production, it's recommended to use Nginx as a reverse proxy with Let's Encrypt:

Pour la production, il est recommandé d'utiliser Nginx comme reverse proxy avec Let's Encrypt :

```nginx
# /etc/nginx/sites-available/flowcdn
server {
    listen 80;
    server_name api.flowcdn.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.flowcdn.com;
    
    # Let's Encrypt certificates
    ssl_certificate /etc/letsencrypt/live/api.flowcdn.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.flowcdn.com/privkey.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Proxy to FlowCDN
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
# Enable site and reload Nginx
sudo ln -s /etc/nginx/sites-available/flowcdn /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🌐 CORS (Cross-Origin Resource Sharing)

### Configuration

CORS is configured in `src/config/cors.config.ts`.

#### Development Mode
- All origins allowed
- All methods allowed
- Credentials enabled

#### Production Mode
- Whitelist-based origins
- Specific methods only
- Credentials enabled

### Environment Variables

```env
# .env
CORS_ORIGINS=https://yourdomain.com,https://www.yourdomain.com,https://app.yourdomain.com
```

### Allowed Origins

By default, the following origins are allowed:
- `http://localhost:3000`
- `http://localhost:3001`
- `http://127.0.0.1:3000`
- `http://127.0.0.1:3001`
- Any origins specified in `CORS_ORIGINS` env variable

### Allowed Methods

- `GET`
- `POST`
- `PUT`
- `DELETE`
- `OPTIONS`

### Allowed Headers

- `Content-Type`
- `Authorization`
- `X-API-Key`
- `X-Requested-With`

---

## 🛡️ Helmet Security Headers

Helmet sets various HTTP headers to protect against common vulnerabilities.

### Enabled Protections

#### Content Security Policy (CSP)
Prevents XSS attacks by controlling which resources can be loaded.

#### DNS Prefetch Control
Disables browser DNS prefetching to prevent information leakage.

#### Frameguard
Prevents clickjacking by denying iframe embedding.

#### Hide Powered-By
Removes the `X-Powered-By` header to hide Express.

#### HTTP Strict Transport Security (HSTS)
Forces HTTPS connections for 1 year.

#### IE No Open
Prevents IE from executing downloads in the site's context.

#### No Sniff
Prevents MIME type sniffing.

#### Referrer Policy
Controls referrer information sent with requests.

#### XSS Filter
Enables browser's XSS filter.

---

## 🔑 API Key Authentication

See [AUTH.md](./AUTH.md) for complete authentication documentation.

### Security Features

- ✅ SHA-256 hashing
- ✅ Timing-safe comparison
- ✅ No plain-text storage
- ✅ Revocation support
- ✅ Usage tracking

---

## 🚨 Security Best Practices

### ✅ DO

- **Use HTTPS in production** - Always encrypt traffic
- **Rotate API keys regularly** - Change keys every 90 days
- **Use environment variables** - Never hardcode secrets
- **Enable rate limiting** - Prevent abuse
- **Monitor logs** - Track suspicious activity
- **Keep dependencies updated** - Run `npm audit` regularly
- **Use strong passwords** - For any admin interfaces
- **Implement CORS properly** - Whitelist only trusted domains
- **Validate all inputs** - Use Zod schemas
- **Sanitize file names** - Prevent path traversal

### ❌ DON'T

- **Don't commit secrets** - Use `.gitignore`
- **Don't use HTTP in production** - Always use HTTPS
- **Don't allow all CORS origins** - Use whitelist
- **Don't store passwords in plain text** - Always hash
- **Don't trust user input** - Always validate
- **Don't expose error details** - Use generic messages in production
- **Don't use default ports** - Change from 3000 in production
- **Don't skip security headers** - Always use Helmet

---

## 🔍 Security Checklist

### Development
- [ ] Self-signed certificates generated
- [ ] CORS configured for localhost
- [ ] API keys working
- [ ] Rate limiting active
- [ ] Input validation working

### Production
- [ ] Valid SSL certificate installed (Let's Encrypt)
- [ ] HTTPS enforced (HSTS enabled)
- [ ] CORS whitelist configured
- [ ] All secrets in environment variables
- [ ] Rate limiting configured
- [ ] Monitoring enabled
- [ ] Backups configured
- [ ] Error logging enabled
- [ ] Security headers active (Helmet)
- [ ] API keys rotated

---

## 🐛 Common Issues

### "NET::ERR_CERT_AUTHORITY_INVALID"
**Cause:** Self-signed certificate not trusted by browser.

**Solution:**
- Use mkcert for local development
- Or manually trust the certificate in your browser
- In production, use Let's Encrypt

### "CORS Error: No 'Access-Control-Allow-Origin'"
**Cause:** Origin not in whitelist.

**Solution:**
- Add origin to `CORS_ORIGINS` in `.env`
- Or add to whitelist in `src/config/cors.config.ts`

### "API key required"
**Cause:** Missing or invalid API key.

**Solution:**
- Check Authorization header format
- Verify key is active (not revoked)
- See [AUTH.md](./AUTH.md) for details

---

## 📚 Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Helmet Documentation](https://helmetjs.github.io/)
- [CORS Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Let's Encrypt](https://letsencrypt.org/)
- [mkcert](https://github.com/FiloSottile/mkcert)

---

## 🆘 Security Issues

If you discover a security vulnerability, please email:
- **Email:** security@flowcdn.com (replace with actual email)
- **Do NOT** open a public GitHub issue

---

**Stay secure! 🔒**
