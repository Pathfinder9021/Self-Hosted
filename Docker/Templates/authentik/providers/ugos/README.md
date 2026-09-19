# UGOS Auth Proxy

PAM-based authentication proxy for integrating Ugreen NAS with [authentik](https://goauthentik.io/) SSO.

> 🇷🇺 [Документация на русском](README.ru.md)

## Overview

This service enables authentik to authenticate UGOS NAS users without storing passwords and supports automatic user provisioning. Each login attempt is validated in real-time via PAM (reading `/etc/shadow`).

**Key Features:**

- 🔒 **No password caching** — every login is validated in real-time
- ⚡ **Instant sync** — password changes in UGOS apply immediately to SSO
- 👤 **Auto-provisioning** — new users are automatically created in authentik on first login
- 🛡️ **Simple & reliable** — PAM-based authentication

## Architecture

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│    Browser      │────▶│     authentik        │────▶│ ugos-auth-proxy │
│                 │     │  (Expression Policy) │     │  (PAM + passwd) │
└─────────────────┘     └──────────────────────┘     └─────────────────┘
                                                              │
                                                              ▼
                                                      ┌────────────────┐
                                                      │  /etc/shadow   │
                                                      │  /etc/passwd   │
                                                      │  /etc/group    │
                                                      │   (UGOS NAS)   │
                                                      └────────────────┘
```

**Flow:**
1. User enters credentials in authentik
2. authentik's Expression Policy calls ugos-auth-proxy `/validate`
3. If user doesn't exist in authentik, policy calls `/user-info` to get user details
4. Policy creates user in authentik with UGOS attributes
5. User is logged in


## Quick Start

### Docker

```bash
# Start the container
docker compose up -d

# Verify it's running
curl http://localhost:9980/health
```


## API Reference

### POST /validate

Validates user credentials via PAM.

**Request:**

```json
{
  "username": "ugadmin",
  "password": "secret"
}
```

**Success Response (200):**

```json
{
  "valid": true,
  "username": "ugadmin"
}
```

**Failure Response (200):**

```json
{
  "valid": false,
  "error": "Invalid credentials"
}
```

### GET /user-info

Retrieves user information from `/etc/passwd` and `/etc/group`. Used for auto-provisioning users in authentik.

**Query Parameters:**
- `username` (required) — UGOS username

**Success Response (200):**

```json
{
  "exists": true,
  "username": "ugadmin",
  "uid": 1026,
  "gid": 100,
  "full_name": "Admin User",
  "email": "admin@example.com",
  "home": "/var/services/homes/ugadmin",
  "shell": "/bin/sh",
  "groups": ["administrators", "users", "http"]
}
```

**User Not Found Response (200):**

```json
{
  "exists": false,
  "error": "User not found"
}
```

### GET /health

Health check endpoint for monitoring and load balancers.

```json
{
  "status": "healthy",
  "service": "ugos-auth-proxy",
  "version": "0.2.0"
}
```

### GET /ready

Readiness probe for Kubernetes.

```json
{
  "status": "ready"
}
```


## Authentik Integration

### 1. Create Groups

Go to __Directory -> Groups__ and create the following groups.

  | Group Name            | Description                        |
  |-----------------------|------------------------------------|
  | `UGOS Users`          | All auto-provisioned UGOS users    |
  | `UGOS Admins`         | Administrators (mapped from UGOS)  |
  | `UGOS Family`         | Family members (mapped from UGOS)  |


### 2. Create Prompts

Go to __Flows and Stages -> Prompts__ and create the following prompts.

- Create a prompt with a name `ugos-password-prompt`

  | Prompt Property       | Prompt Property Value              |
  |-----------------------|------------------------------------|
  | Name                  | ugos-password-prompt               |
  | Field Key             | password                           |
  | Label                 | Password                           |
  | Type                  | Password: Masked input             |
  | Required              | True                               |
  | Placeholder           | Password                           |
  | Order                 | 300                                |


### 3. Create Stages

Go to __Flows and Stages -> Stages__ and create the following stages.

- Create a stage with a name `ugos-user-identification-stage`

  | Stage Property        | Stage Property Value               |
  |-----------------------|------------------------------------|
  | Type                  | Identification Stage               |
  | Name                  | ugos-user-identification-stage     |
  | User Fields           | Username                           | 
  | Pretend User Exists   | True                               |

  > **Important:** Set `pretend_user_exists=True` to enable auto-provisioning for new users.

- Create a stage with a name `ugos-user-login-stage` 
  
  | Stage Property        | Stage Property Value               |
  |-----------------------|------------------------------------|
  | Type                  | User Login Stage                   |
  | Name                  | ugos-user-login-stage              |
  | Session Duration      | seconds=0                          |
  | Stay Signed In Offset | seconds=0                          |
  | Remember Device       | days=30                            |
  | Network Binding       | Bind ASN                           |
  | GeoIP Binding         | Bind Continent                     |

- Create a stage with a name `ugos-password-prompt-stage`

  | Stage Property        | Stage Property Value               |
  |-----------------------|------------------------------------|
  | Type                  | Prompt Stage                       |
  | Name                  | ugos-password-prompt-stage         |
  | Fields                | ugos-password-prompt               |
  | Validation Policies   | -                                  |

- Create a stage with a name `ugos-password-deny-stage`

  | Stage Property        | Stage Property Value               |
  |-----------------------|------------------------------------|
  | Type                  | Deny Stage                         |
  | Name                  | ugos-password-deny-stage           |
  | Deny Message          | Failed                             |


### 4. Create Expression Policy

Go to __Customization -> Policies__ and create a policy with a name `ugos-credentials-validation-policy`

  | Policy Property       | Policu Field Property              |
  |-----------------------|------------------------------------|
  | Type                  | Expression Policy                  |
  | Name                  | ugos-credentials-validation-policy |
  | Expresssion           | Copy from authentik-policy.py      |

> **Configuration:** Edit the variables at the top of the policy:
> - `UGOS_AUTH_PROXY_URL` — your ugos-auth-proxy address
> - `DEFAULT_EMAIL_DOMAIN` — default email domain for users
> - `GROUP_MAP` — UGOS to authentik group mapping

The policy automatically maps UGOS groups to authentik groups:

| UGOS Group | authentik Group | Description        |
|------------|-----------------|--------------------|
| `admin`    | `UGOS Admins`        | NAS administrators |
| `family`   | `UGOS Family`        | Family members     |

This policy handles both password validation and auto-provisioning. All auto-provisioned users are added to `UGOS Users` group.

Auto-provisioned users will have these attributes:

```json
{
  "ugos_user": true,
  "ugos_uid": 1005,
  "ugos_gid": 10,
  "ugos_groups": ["admin", "users", "ughomeusers"],
  "ugos_home": "/home/ugadmin",
  "auto_provisioned": true
}
```

### 5. Create Authentication Flow

Go to __Flows and Stages -> Flows__ and create a flow with a name `ugos-authentication-flow` 

  | Flow Property         | Flow Field Property                |
  |-----------------------|------------------------------------|
  | Name                  | ugos-authentication-flow           |
  | Title                 | Welcome to UGOS                    |
  | Slug                  | ugos-authentication                |
  | Designation           | Authentication                     |
  | Authentication        | Require no authentication          |

Navigate to flow's Stage binding tab and bind the following stages to it

  | Order                 | Stage                              |
  |-----------------------|------------------------------------|
  | 10                    | ugos-user-identification-stage     |
  | 100                   | ugos-user-login-stage              |
  | 20                    | ugos-password-prompt-stage         |
  | 25                    | ugos-password-deny-stage           |
  
  > Set __Evaluate when flow is planned__ to __False__, __Evaluate when stage is run__ to __True__, __Invalid response behavior__ to __RETRY__ and __Policy engine mode__ to __ANY__.
 
After that expand the binding for the `ugos-password-deny-stage` stage and bind the `ugos-credentials-validation-policy` policy  to `ugos-password-deny-stage` with __Enabled__ set to __True__ and __Negate Result__ to __True__.

> Use inspector to verify the policy. 

### 6. Set Default Flow

In Brand settings (System -> Brands), set `ugos-authentication` as the default authentication flow.

### 7. Local Admin Access

For emergency access with local admin (e.g., `akadmin`), use:
```
https://your-authentik-url/if/flow/default-authentication-flow/
```


## Testing

```bash
# Successful authentication
curl -X POST http://localhost:8180/validate \
  -H "Content-Type: application/json" \
  -d '{"username":"ugadmin","password":"correct_password"}'

# Failed authentication
curl -X POST http://localhost:8180/validate \
  -H "Content-Type: application/json" \
  -d '{"username":"ugadmin","password":"wrong_password"}'

# Get user info
curl "http://localhost:8180/user-info?username=ugadmin"

# Health check
curl http://localhost:8180/health
```

## Security Considerations

⚠️ **Important:**

1. The container requires access to `/etc/shadow`, `/etc/passwd`, `/etc/group` — mounted as read-only
2. **Do not expose** the port externally — use internal Docker network only
3. Consider using HTTPS between authentik and proxy (via reverse proxy)
4. The `/user-info` endpoint only exposes public user data (no passwords)

## Why PAM Instead of UGOS API?

UGOS uses a complex authentication scheme:

1. **Dynamic RSA keys** — new key pair generated for each session
2. **Password encryption** — password is encrypted with RSA public key
3. **Multi-step process** — multiple API calls for a single login
4. **Circular dependency** — public key is only available after login

PAM-based authentication bypasses these complexities and works directly with `/etc/shadow`, which stays in sync with
UGOS passwords.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
