# Internal Kubernetes Apps with Google Login Using OAuth2 Proxy

This repository contains Kubernetes manifests to secure internal applications with Google authentication using `oauth2-proxy`. Users are redirected to Google login before accessing protected apps.

For a detailed walkthrough, see the full article: [Securing Internal Kubernetes Apps with Google Login Using OAuth2 Proxy](https://www.markcallen.com/securing-internal-kubernetes-apps-with-google-login-using-oauth2-proxy/)

## 🧱 Prerequisites

Before you begin, ensure you have:

- A Kubernetes cluster with:
  - **cert-manager** installed with a ClusterIssuer named `letsencrypt-dns` ([installation guide](https://www.markcallen.com/how-to-install-and-configure-cert-manager-with-cloudflare-on-kubernetes/))
  - **external-dns** configured with your DNS provider ([installation guide](https://www.markcallen.com/automate-kubernetes-dns-with-externaldns-and-cloudflare/))
  - **nginx-ingress-controller** installed
- A verified domain (replace `markcallen.dev` with your domain throughout)
- Two subdomains ready:
  - `login.yourdomain.com` → hosts `oauth2-proxy`
  - `whoami.yourdomain.com` → your protected app (example)
- Google OAuth credentials (see Step 1 below)

## 🛠️ Step 1: Create Google OAuth Credentials

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or use an existing one
3. Navigate to **OAuth consent screen** → Choose "External" → Fill out app info
4. Add your domain (e.g., `markcallen.dev`) as an **Authorized Domain**
5. Go to **Credentials** → **Create Credentials** → **OAuth Client ID**
   - Application type: **Web App**
   - Authorized JavaScript origins: `https://login.yourdomain.com`
   - Authorized redirect URIs: `https://login.yourdomain.com/oauth2/callback`
6. Click **Create**
7. Copy your **Client ID** and **Client Secret** (you'll need these next)

## 🔐 Step 2: Deploy OAuth2 Proxy

### Create the namespace

```bash
kubectl create ns oauth2
```

### Create the login certificate

Apply the `login-cert.yaml` file to request a TLS certificate for your login subdomain:

```bash
kubectl apply -f login-cert.yaml
```

**Note:** Update the `dnsNames` field in `login-cert.yaml` with your domain before applying.

Wait for the certificate to be issued:

```bash
kubectl describe order -n oauth2
```

Look for: `Normal  Complete  Order completed successfully`

### Install oauth2-proxy with Helm

Add the Helm repository:

```bash
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
helm repo update
```

Generate a cookie secret:

```bash
openssl rand -base64 32 | tr -d '\n' | tr '+/' '-_'
```

### Configure values.yaml

Edit the `values.yaml` file and replace the following values:

- `clientID` - Your Google OAuth Client ID
- `clientSecret` - Your Google OAuth Client Secret
- `cookieSecret` - The cookie secret generated above
- `redirectUrl` - Your login domain callback URL
- `email_domains` - Your organization's email domain
- `cookie_domains` - Your root domain (with leading dot)
- `whitelist_domains` - Your root domain (with leading dot)
- Update the `ingress.hosts` and `tls.hosts` with your login subdomain

Install using Helm:

```bash
helm install oauth2-proxy oauth2-proxy/oauth2-proxy -f values.yaml --namespace oauth2
```

### Verify the deployment

Check that DNS is set up correctly:

```bash
dig +short login.yourdomain.com
```

Test HTTPS access:

```bash
curl https://login.yourdomain.com
```

You should get a response without certificate errors.

## 📦 Step 3: Deploy the Protected App

This example uses the `whoami` application, but you can protect any internal app using the same pattern.

### Deploy the application

```bash
kubectl apply -f deployment.yaml
```

This creates:

- A Deployment running the `traefik/whoami` container
- A Service exposing port 80

## 🔐 Step 4: Create Certificate for Protected App

Apply the certificate manifest:

```bash
kubectl apply -f whoami-cert.yaml
```

**Note:** Update the `dnsNames` field in `whoami-cert.yaml` with your protected app subdomain.

Wait for the certificate to be ready:

```bash
kubectl get certificate -n default
```

## 🌐 Step 5: Create Protected Ingress

Apply the ingress manifest:

```bash
kubectl apply -f ingress.yaml
```

**Important:** Before applying, update the following in `ingress.yaml`:

- Replace `whoami.markcallen.dev` with your protected app subdomain
- Replace `login.markcallen.dev` with your login subdomain in the auth-signin annotation
- Ensure `auth-url` points to your oauth2-proxy service (update if namespace differs)

### Key annotations in ingress.yaml

```yaml
nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy.oauth2.svc.cluster.local/oauth2/auth"
nginx.ingress.kubernetes.io/auth-signin: "https://login.yourdomain.com/oauth2/start?rd=https://$host$escaped_request_uri"
nginx.ingress.kubernetes.io/auth-response-headers: "X-Forwarded-User"
```

These annotations:

- Route authentication requests to oauth2-proxy
- Redirect unauthenticated users to Google login
- Forward the authenticated user's email to your app

## ✅ Testing

1. Open your browser and navigate to `https://whoami.yourdomain.com`
2. You should be redirected to `https://login.yourdomain.com` → Google login
3. Sign in with a Google account matching your configured `email_domains`
4. On success, you'll be redirected back to your app with a secure cookie
5. Subsequent visits skip login (until the cookie expires)

## 🔧 Protecting Additional Apps

To protect more internal apps, repeat Steps 3-5 for each app:

1. Deploy your application
2. Create a certificate for the new subdomain
3. Create an ingress with the same authentication annotations

The same oauth2-proxy instance can protect multiple apps under `*.yourdomain.com`.

## 📝 File Reference

| File               | Purpose                                             |
| ------------------ | --------------------------------------------------- |
| `values.yaml`      | Helm values for oauth2-proxy configuration          |
| `login-cert.yaml`  | Certificate for `login.yourdomain.com`              |
| `deployment.yaml`  | Example application (whoami) deployment and service |
| `whoami-cert.yaml` | Certificate for protected app subdomain             |
| `ingress.yaml`     | Ingress with OAuth2 proxy authentication            |

## 🧠 How It Works

1. User visits `whoami.yourdomain.com`
2. Nginx ingress checks with oauth2-proxy at `/oauth2/auth`
3. If no valid cookie exists, user is redirected to Google
4. After Google authentication, oauth2-proxy issues a secure cookie
5. User is redirected back to the original app with authenticated access
6. The `X-Forwarded-User` header contains the user's email

## 🔒 Security Notes

- OAuth2 cookies are scoped to `*.yourdomain.com`
- Only users with email addresses matching `email_domains` can authenticate
- TLS certificates are automatically managed by cert-manager
- DNS records are automatically managed by external-dns
- The oauth2-proxy service is only accessible within the cluster (internal HTTP)

## 🐛 Troubleshooting

### Login redirect loop

- Verify your Google OAuth redirect URI matches exactly: `https://login.yourdomain.com/oauth2/callback`
- Check that `cookie_domains` and `whitelist_domains` include your root domain with leading dot

### Certificate not issuing

- Check cert-manager logs: `kubectl logs -n cert-manager deployment/cert-manager`
- Verify your ClusterIssuer is configured correctly

### 502 Bad Gateway

- Ensure your application pod is running: `kubectl get pods`
- Check the service is accessible: `kubectl get svc`

### DNS not resolving

- Verify external-dns logs: `kubectl logs -n external-dns deployment/external-dns`
- Confirm the annotation is present: `kubectl describe ingress whoami-ingress`

## 📚 Additional Resources

- [OAuth2 Proxy Documentation](https://oauth2-proxy.github.io/oauth2-proxy/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [ExternalDNS Documentation](https://github.com/kubernetes-sigs/external-dns)

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
