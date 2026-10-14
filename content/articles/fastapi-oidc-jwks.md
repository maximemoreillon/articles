---
date: "2025-09-13T11:00:00+09:00"
title: "Verifying OIDC access tokens in FastAPI with JWKS"
tags: ["Python", "FastAPI", "OIDC", "Authentication"]
---

When a frontend authenticates its users with an OIDC identity provider, it typically calls its backend API with the user's access token in the `Authorization: Bearer` header. Before trusting the claims in that token, the API needs to verify its signature using the public keys of the identity provider, which are published at the provider's JWKS (JSON Web Key Set) endpoint. Since this has to be done in every API that sits behind the identity provider, I packaged the verification logic as a FastAPI dependency, [`fastapi-oidc-jwks`](https://github.com/maximemoreillon/fastapi-oidc-jwks), published on PyPI.

## Installation

```bash
pip install fastapi-oidc-jwks
```

## How it works

The dependency is a class whose instances are callable, so a single instance, configured with the JWKS URI of the identity provider, can be passed to FastAPI's `Depends`. It relies on [PyJWT](https://pyjwt.readthedocs.io/), whose `PyJWKClient` fetches the key set from the JWKS endpoint and picks the key matching the `kid` header of the token. The token is then decoded and its signature verified with that key:

```python
from typing import Optional
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from jwt import PyJWKClient


class AuthDependency:
    def __init__(
        self,
        jwks_uri: str,
        issuer: Optional[str] = None,
        audience: Optional[str] = None,
    ):

        if not jwks_uri:
            raise Exception("jwks_uri not set")

        self.jwks_uri = jwks_uri
        self.jwks_client = PyJWKClient(self.jwks_uri)
        self.issuer = issuer
        self.audience = audience

    def __call__(
        self,
        credentials: HTTPAuthorizationCredentials = Security(HTTPBearer()),
    ):
        token = credentials.credentials

        try:
            signing_key = self.jwks_client.get_signing_key_from_jwt(token)
            return jwt.decode(
                token,
                signing_key.key,
                algorithms=["RS256"],
                audience=self.audience,
                issuer=self.issuer,
                options={
                    "verify_iss": self.issuer is not None,
                    "verify_aud": self.audience is not None,
                },
            )

        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail="Token expired")
        except jwt.InvalidAudienceError:
            raise HTTPException(status_code=401, detail="Invalid audience")
        except jwt.InvalidIssuerError:
            raise HTTPException(status_code=401, detail="Invalid issuer")
        except jwt.PyJWTError as e:
            raise HTTPException(status_code=401, detail=f"Invalid token: {str(e)}")
```

`HTTPBearer` takes care of extracting the token from the `Authorization` header and rejects requests that don't have one. If the token is valid, the dependency returns its claims; otherwise, it responds with a `401` and a message describing what went wrong.

## Usage

The dependency can be applied to a whole router, so that every route of that router requires a valid token. Routes that need to know who the user is can also declare it as a parameter to receive the token's claims:

```python
from fastapi import FastAPI, APIRouter, Depends
from fastapi_oidc_jwks import AuthDependency

app = FastAPI()

auth = AuthDependency("Your OIDC provider's JWKS URI here")

router = APIRouter(dependencies=[Depends(auth)])

@router.get("/user")
async def handle(user: dict = Depends(auth)):
    return {"user": user}

app.include_router(router)
```

The JWKS URI of an identity provider can be found under `jwks_uri` in its discovery document, served at `/.well-known/openid-configuration` under the issuer's URL.

## Checking the issuer and the audience

A valid signature only proves that the token was issued by the identity provider, not that it was meant for this API. The `issuer` and `audience` parameters, when set, make the dependency also check the `iss` and `aud` claims of the token. They are optional because the content of these claims depends on how the identity provider is configured, so they are only checked when explicitly provided:

```python
auth = AuthDependency(
    "Your OIDC provider's JWKS URI here",
    issuer="Your OIDC provider's issuer URL here",
    audience="Your API's audience here",
)
```

## Middleware

The package also provides `AuthMiddleware` for cases where every request to the application should be authenticated. Instead of being returned, the token's claims are attached to the request, under `request.state.user`:

```python
from fastapi import FastAPI, Request
from fastapi_oidc_jwks import AuthMiddleware

app = FastAPI()

app.add_middleware(AuthMiddleware, jwks_url="Your OIDC provider's JWKS URI here")

@app.get("/user")
async def handle(request: Request):
    return {"user": request.state.user}
```
