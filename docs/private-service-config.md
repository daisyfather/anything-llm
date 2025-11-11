# Private service configuration for AnythingLLM

This guide explains how to point AnythingLLM at privately hosted infrastructure
without routing requests through an HTTP proxy. The settings below map directly
to environment variables consumed by the server and frontend applications.

## Backend: connect directly to self-hosted services

### Use a private Qdrant deployment

1. Set the vector database provider to Qdrant.
2. Provide the base URL of your private Qdrant instance. Use the direct service
   host (for example an internal load balancer or the container hostname) and
   include the port. Do **not** prefix the URL with a proxy hostname.
3. Optionally provide an API key if your Qdrant instance requires it.

```bash
VECTOR_DB="qdrant"
QDRANT_ENDPOINT="http://qdrant.internal:6333"
# QDRANT_API_KEY="your-token-if-required"
```

AnythingLLM passes these values directly to the Qdrant SDK so requests are sent
straight to the host specified in `QDRANT_ENDPOINT`.【F:server/.env.example†L247-L250】

### Point the LLM provider to LocalAI (or another local host)

To run inference against a self-hosted LLM without a proxy, pick one of the
local providers and set its base path to the private service URL. The example
below uses LocalAI.

```bash
LLM_PROVIDER="localai"
LOCAL_AI_BASE_PATH="http://localai.internal:8080/v1"
LOCAL_AI_MODEL_PREF="luna-ai-llama2"
# LOCAL_AI_MODEL_TOKEN_LIMIT=4096
# LOCAL_AI_API_KEY="optional-if-your-instance-uses-auth"
```

Optionally configure embeddings to use the same LocalAI host so that all
requests stay within your network perimeter.

```bash
EMBEDDING_ENGINE="localai"
EMBEDDING_BASE_PATH="http://localai.internal:8080/v1"
EMBEDDING_MODEL_PREF="text-embedding-ada-002"
```

These variables are consumed directly by the backend and bypass any proxy
layer. Avoid the `generic-openai` provider if you want to enforce direct calls
because that workflow is designed for proxy-compatible OpenAI APIs.【F:server/.env.example†L33-L37】【F:server/.env.example†L90-L95】【F:server/.env.example†L168-L171】

### Ensure the hostnames resolve inside the AnythingLLM deployment

If you deploy with Docker, add the private service hostnames to your Docker
network (for example using `extra_hosts` or an overlay network). On bare metal
installs, update `/etc/hosts` or your DNS so the AnythingLLM server can resolve
`qdrant.internal` and `localai.internal` directly.

## Frontend: disable the native login page for DataPower SSO

The frontend honors the Simple SSO environment variables that are read from the
backend API. To hide the built-in login modal and immediately redirect users to
an external DataPower flow, set the following on the server:

```bash
SIMPLE_SSO_ENABLED=1
SIMPLE_SSO_NO_LOGIN=1
SIMPLE_SSO_NO_LOGIN_REDIRECT="https://datapower.example.com/login"
```

When these flags are present the frontend skips the `/login` route and performs
an automatic redirect to the URL you provide. Use your DataPower SSO entrypoint
in `SIMPLE_SSO_NO_LOGIN_REDIRECT`.【F:server/.env.example†L360-L362】【F:server/models/systemSettings.js†L673-L684】

After updating the environment variables restart the AnythingLLM server so it
can reload the new configuration. The frontend consumes the flags via the
System Settings API, so no additional build steps are required.
