# From Source to Production with OAuth: The Full Kindling Flow

Your laptop is your staging environment. That's not a slogan — it's the actual architecture. In this post, we'll take a polyglot microservice app from local source code to a production Kubernetes cluster with TLS, Auth0 login, and Stripe webhooks — all working end-to-end. No cloud staging environment. No Docker Compose. No YAML by hand.

The app is [oauth-example](https://github.com/kindling-sh/oauth-example), a four-service demo you can clone and follow along with.

---

## What we're building

A microservices app with real external integrations:

```
Browser → UI (React/nginx)
            ↓
         Gateway (Go) ← Auth0 OIDC login/callback
            ↓              ← Stripe webhook receiver
       ┌────┴────┐
    Orders     Inventory
   (Python)    (Node.js)
   Postgres    MongoDB
      Redis ←──── shared event queue
```

- **UI** — React dashboard served by nginx
- **Gateway** — Go reverse proxy with Auth0 OIDC and Stripe webhook verification
- **Orders** — Python FastAPI with Postgres, publishes events to Redis
- **Inventory** — Node.js Fastify with MongoDB, consumes order events from Redis

The interesting part: Auth0 needs a public HTTPS callback URL, and Stripe needs a public HTTPS webhook endpoint. These are the exact things that make local development painful — and the exact things kindling solves.

---

## Step 1: Init the cluster

```bash
brew install kindlingdev/tap/kindling
kindling init
```

That's it. You now have a Kind cluster with an operator, an in-cluster container registry, and a Traefik ingress controller. All local, all disposable.

## Step 2: Register a CI runner

```bash
kindling runners -u <github-user> -r oauth-test -t <github-pat>
```

This deploys a self-hosted GitHub Actions runner inside your Kind cluster. When you push code, CI runs locally — builds happen with Kaniko, images land in the in-cluster registry at `localhost:5001`. No DockerHub. No ECR. No waiting for remote CI.

## Step 3: Generate the CI workflow

```bash
kindling generate -k <api-key> -r .
```

Kindling's AI analyzes your repo — finds the four services, detects languages and frameworks, reads the Dockerfiles, discovers the deploy manifests — and generates a GitHub Actions workflow that builds and deploys everything. The generated workflow handles:

- Building each service with Kaniko
- Pushing images to the in-cluster registry
- Applying the DSE (DevStagingEnvironment) manifests
- Service-to-service wiring via environment variables
- Dependency provisioning (Postgres, Redis, MongoDB)

You don't write this workflow. You don't maintain it. Push code and it runs.

## Step 4: Create Auth0 and Stripe accounts and set credentials

The gateway's deploy manifest references secrets via `secretKeyRef` — Kubernetes won't start the pod if they don't exist. You need to create your external service accounts and set the credentials *before* the first deploy.

### Create an Auth0 application

1. Sign up or log in at [auth0.com](https://auth0.com)
2. In the Auth0 Dashboard, go to **Applications → Create Application**
3. Name it something like `oauth-example-dev`, select **Regular Web Application**, and click **Create**
4. In the **Settings** tab, copy these three values — you'll need them in a moment:
   - **Domain** (e.g. `your-tenant.us.auth0.com`)
   - **Client ID**
   - **Client Secret**

Don't configure the callback URLs yet — you'll need a public tunnel URL first, which comes after the deploy.

### Create a Stripe webhook endpoint

1. Sign up or log in at [dashboard.stripe.com](https://dashboard.stripe.com)
2. Make sure you're in **Test mode** (toggle in the top-right)
3. Open the **Webhooks** tab in [Workbench](https://dashboard.stripe.com/webhooks)

You'll create the actual webhook endpoint later once you have a tunnel URL. For now, just copy your **Stripe Secret Key** from **Developers → API keys** — it starts with `sk_test_`.

### Set the pre-deploy secrets

Set the credentials so the pods can start:

```bash
kindling secrets set AUTH0_DOMAIN your-tenant.auth0.com
kindling secrets set AUTH0_CLIENT_ID your-client-id
kindling secrets set AUTH0_CLIENT_SECRET your-client-secret
kindling secrets set SESSION_SECRET $(openssl rand -hex 32)
kindling secrets set STRIPE_SECRET_KEY sk_test_your_stripe_secret_key
kindling secrets set STRIPE_WEBHOOK_SECRET placeholder
kindling secrets set PUBLIC_URL http://localhost
```

`STRIPE_WEBHOOK_SECRET` and `PUBLIC_URL` are placeholders — the gateway will start without them working, but it won't crash. You'll set the real values after starting a tunnel.

Note: `STRIPE_SECRET_KEY` (starts with `sk_test_`) is your Stripe **API key** from **Developers → API keys**. This is different from `STRIPE_WEBHOOK_SECRET` (starts with `whsec_`), which is the webhook **signing secret** you'll get after creating a webhook endpoint in Step 6.

The DSE manifests reference these via `secretKeyRef` — the operator injects them as environment variables into the gateway pod. No secrets in YAML. No secrets in env files. No secrets in git.

## Step 5: Push and deploy

```bash
git add -A && git commit -m "initial deploy"
git push origin main
```

The runner picks up the push, builds all four images, and the operator deploys them. Check status:

```bash
kindling status
```

```
▸ Dev Staging Environments
    📦 jeff-vincent-gateway      9090   jeff-vincent-gateway.localhost
    📦 jeff-vincent-inventory    3000   jeff-vincent-inventory.localhost
    📦 jeff-vincent-orders       5000   jeff-vincent-orders.localhost
    📦 jeff-vincent-ui           80     jeff-vincent-ui.localhost

▸ All Deployments
    jeff-vincent-gateway             1/1
    jeff-vincent-inventory           1/1
    jeff-vincent-inventory-mongodb   1/1
    jeff-vincent-orders              1/1
    jeff-vincent-orders-postgres     1/1
    jeff-vincent-orders-redis        1/1
    jeff-vincent-ui                  1/1
```

Four services, three databases, all running. Open `http://jeff-vincent-ui.localhost` and the dashboard is live. Auth0 and Stripe aren't wired yet — that's next.

## Step 6: Start a tunnel and configure OAuth + webhooks

Now that the services are running, you need a public HTTPS URL for Auth0 callbacks and Stripe webhooks. Kindling uses Cloudflare's free quick tunnels — no account, no domain, no configuration:

```bash
brew install cloudflare/cloudflare/cloudflared   # one-time install
kindling expose
```

You'll get a public URL like `https://verb-noun-adj-noun.trycloudflare.com`. Copy it.

The tunnel runs in the background. The URL changes each time you restart it, so you'll update the callback URLs in Auth0 and Stripe when you need to test external integrations. Most day-to-day development doesn't need the tunnel at all.

### Configure Auth0 callback URLs

Go back to your Auth0 application settings and set:

> **Auth0 Dashboard → Applications → oauth-example-dev → Settings → Application URIs**
>
> **Allowed Callback URLs**
> ```
> https://<your-tunnel-url>/auth/callback
> ```
>
> **Allowed Logout URLs**
> ```
> https://<your-tunnel-url>
> ```
>
> **Allowed Web Origins**
> ```
> https://<your-tunnel-url>
> ```
>
> Replace `<your-tunnel-url>` with the URL from `kindling expose` (e.g. `verb-noun-adj-noun.trycloudflare.com`).

Click **Save Changes**.

The callback URL is what Auth0 redirects to after a user authenticates. It must match *exactly* what the gateway sends in the OIDC authorization request — the path `/auth/callback` is handled by the gateway's OIDC callback handler, which exchanges the authorization code for tokens and sets a session cookie.

The gateway's OIDC integration uses Auth0's [Universal Login](https://auth0.com/docs/authenticate/login/auth0-universal-login) — no custom login page needed. For more detail, see [Auth0's Getting Started guide](https://auth0.com/docs/get-started).

### Create the Stripe webhook endpoint

Now create the webhook endpoint in Stripe with your tunnel URL:

1. In [Workbench → Webhooks](https://dashboard.stripe.com/webhooks), click **Create an event destination**
2. Walk through the creation flow:

> **Events from**
> ```
> Your account
> ```
>
> **API version**
> Select your default API version (or latest).
>
> **Events**
> ```
> checkout.session.completed
> payment_intent.succeeded
> ```
>
> Click **Continue**, then select **Webhook endpoint** as the destination type.
>
> Click **Continue**, then set:
>
> **Endpoint URL**
> ```
> https://<your-tunnel-url>/webhooks/stripe
> ```

3. Click **Create destination**
4. Select your new endpoint, then click **Click to reveal** to copy the signing secret (`whsec_...`)

The webhook URL is where Stripe sends event payloads via POST. The gateway receives the request at `/webhooks/stripe`, verifies the `Stripe-Signature` header against the signing secret using HMAC-SHA256, and forwards validated payloads to the orders service. Unverified requests are rejected with a 400. See [Stripe's webhook docs](https://docs.stripe.com/webhooks) for background on signature verification and event types.

### Set the real PUBLIC_URL and webhook secret

Now update the placeholder secrets with the real values:

```bash
kindling secrets set PUBLIC_URL https://<your-tunnel-url>
kindling secrets set STRIPE_WEBHOOK_SECRET whsec_your_signing_secret
```

The gateway pod will restart automatically to pick up the new values. Verify with `kindling status` that the gateway is back to 1/1.

## Step 7: The inner dev loop

Now you're developing. You change code. You need it reflected immediately. Check status:

```bash
kindling status
```

```
▸ Dev Staging Environments
    📦 jeff-vincent-gateway      9090   jeff-vincent-gateway.localhost
    📦 jeff-vincent-inventory    3000   jeff-vincent-inventory.localhost
    📦 jeff-vincent-orders       5000   jeff-vincent-orders.localhost
    📦 jeff-vincent-ui           80     jeff-vincent-ui.localhost

▸ All Deployments
    jeff-vincent-gateway             1/1
    jeff-vincent-inventory           1/1
    jeff-vincent-inventory-mongodb   1/1
    jeff-vincent-orders              1/1
    jeff-vincent-orders-postgres     1/1
    jeff-vincent-orders-redis        1/1
    jeff-vincent-ui                  1/1
```

Four services, three databases, all running. Open `http://jeff-vincent-ui.localhost` and the dashboard is live.

## Step 6: The inner dev loop

Now you're developing. You change code. You need it reflected immediately.

```bash
kindling sync -d jeff-vincent-gateway
```

File changes sync directly into the running pod — no image rebuild, no redeploy. For Go services, the binary recompiles inside the container. For Python and Node.js, the file change triggers a reload automatically.

Need to step through code? Run `debug` from the project root:

```bash
cd /path/to/oauth-test
kindling debug -d jeff-vincent-orders --port 5678
```

> **Note:** `kindling debug` must be run from the project root directory — it needs access to the source tree to set up the debug session.

Attach your IDE's debugger to `localhost:5678` and set breakpoints in the orders service while it's running inside the cluster, talking to real Postgres and Redis.

## Step 8: Test the OAuth and Stripe flows

With the services running and secrets configured, verify everything is wired:

```bash
curl http://jeff-vincent-gateway.localhost/auth/status
# {"auth0_configured":true,"callback_url":"https://<your-tunnel-url>/auth/callback"}

curl http://jeff-vincent-gateway.localhost/stripe/status
# {"stripe_webhook_configured":true,"webhook_url":"https://<your-tunnel-url>/webhooks/stripe"}
```

Open your tunnel URL in a browser (e.g. `https://verb-noun-adj-noun.trycloudflare.com`), click Login — Auth0's universal login page loads, you authenticate, and the callback redirects to your tunnel URL. The request routes through Cloudflare → localhost:80 → Traefik → the gateway ingress, which exchanges the code for tokens and sets a session cookie. The full OIDC flow, running locally.

For Stripe, trigger a test event from the Stripe Dashboard (or use the [Stripe CLI](https://docs.stripe.com/stripe-cli)). The event hits the gateway at your tunnel URL via the same path — Cloudflare → localhost:80 → Traefik → gateway — the signature is verified against the signing secret, and the payload is forwarded to the orders service, which updates the order status in Postgres.

## Step 9: Deploy to production

The dev environment is validated. Auth0 callbacks work. Stripe webhooks land. Time to go live.

```bash
kindling dashboard --prod-context <your-prod-cluster-context>
```

The production dashboard connects to your real cluster. From there:

1. **Snapshot** your dev environment — the operator captures the exact image tags, env vars, and dependency configuration
2. **Deploy** to production — images are re-tagged and pushed to your production registry, manifests are applied
3. **TLS** — cert-manager provisions Let's Encrypt certificates automatically

> **Note:** After deploying, you'll need to update your domain's DNS A record to point to the new load balancer IP (find it with `kubectl get svc traefik -n traefik`). DNS propagation can take a few minutes — during that time you may see a browser warning like "Your connection is not private" (`ERR_CERT_AUTHORITY_INVALID`). This is normal. Go walk your dog, knit a sweater, contemplate the mass of the universe — whatever you do, don't sit there refreshing the browser. Once DNS propagates, cert-manager will complete the ACME challenge and the Let's Encrypt certificate will be issued automatically.

In production, swap the secrets for production values:

- `AUTH0_DOMAIN` → your production Auth0 tenant
- `AUTH0_CLIENT_ID` / `AUTH0_CLIENT_SECRET` → production app credentials
- `STRIPE_SECRET_KEY` → production Stripe API key (starts with `sk_live_`)
- `STRIPE_WEBHOOK_SECRET` → production webhook signing secret (starts with `whsec_`)
- `PUBLIC_URL` → your production domain

The same code, the same architecture, the same deployment model. The only things that change are the secrets and the domain.

---

## What just happened

Let's trace the full path:

1. **Source code** on your laptop
2. **`kindling init`** — local Kind cluster with operator, registry, ingress
3. **`kindling generate`** — AI-generated CI workflow
4. **`kindling secrets set`** — set Auth0/Stripe credentials so pods can start
5. **`git push`** — local runner builds with Kaniko, deploys via operator
6. **`kindling expose` + `kindling secrets set`** — start tunnel, configure Auth0/Stripe callback URLs, set PUBLIC_URL
7. **`kindling sync`** — live code changes without rebuilding
8. **Test OAuth + Stripe** — verify callback flows end-to-end
9. **Production deploy** — same images, same config, real cluster with TLS

No Docker Compose file. No Helm charts. No Terraform. No cloud staging environment bill. No "works on my machine" gaps between dev and prod.

The entire staging environment runs on your laptop. When it works there, it works in production — because it's the same Kubernetes, the same container images, the same networking model.

---

## Try it

```bash
brew install kindlingdev/tap/kindling
git clone https://github.com/kindling-sh/oauth-example
cd oauth-example
kindling init
kindling runners -u <user> -r oauth-example -t <pat>
kindling generate -k <api-key> -r .
# Set credentials so pods can start
kindling secrets set AUTH0_DOMAIN <your-domain>
kindling secrets set AUTH0_CLIENT_ID <your-id>
kindling secrets set AUTH0_CLIENT_SECRET <your-secret>
kindling secrets set SESSION_SECRET $(openssl rand -hex 32)
kindling secrets set STRIPE_SECRET_KEY <sk_test_...>
kindling secrets set STRIPE_WEBHOOK_SECRET placeholder
kindling secrets set PUBLIC_URL http://localhost
# Deploy
git push origin main
# Start tunnel, configure Auth0/Stripe callback URLs, then update secrets
kindling expose
# copy the tunnel URL, set it in Auth0 + Stripe dashboards
kindling secrets set PUBLIC_URL <your-tunnel-url>
kindling secrets set STRIPE_WEBHOOK_SECRET <whsec_...>
```

Your laptop is your staging environment. Start building.
