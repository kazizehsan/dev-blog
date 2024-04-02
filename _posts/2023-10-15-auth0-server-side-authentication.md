---
title: Server side authentication with Auth0
date: 2023-10-15 20:05:00 +0600
categories: [Guides]
tags: [login, auth0, authentication, resource-owner-password, regular-web-app]
---

> **Auth0 can be daunting ... at first.**

![Desktop View](/assets/img/bg/micah-williams-lmFJOx7hPc4-unsplash.jpg){: w="700" h="400" }
_Photo by <a href="https://unsplash.com/@mr_williams_photography?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Micah Williams</a> on <a href="https://unsplash.com/photos/black-and-gray-code-padlock-anchored-on-chain-link-fence-selective-focus-photo-lmFJOx7hPc4?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>_

## Your API Server handles registration and authentication
So, you are coming from a traditional backend-handles-all-things-authentication scenario. Now you need to build something similar with Auth0. Well, the best thing to do is to go through Auth0 documentations with patience and familiarize yourself with their terminology (`API`, `Application`, various flows, etc.). But if you are still confused, this article may help confirm a few things.

### An M2M Application

Start with creating an `API` in the existing `Tenant`.

A `Database Connection` is also created automatically.

Alongwith the `API`, a `Machine to Machine` `Application` is automatically created which contains our `domain`, `client_id` and `client_secret`. In "Auth0 Dashboard" -> "Applications" -> "Applications" -> "YOUR_M2M_APP" -> "APIs", you will find this M2M `Application` has been authorised to request access tokens for the `API` that we just created.  But this application has not yet been authorized to request access tokens for the Auth0 Management API.

Under these settings, you can set up the Auth0 SDK in the following manner (shown in Python with FastAPI). This will allow you to sign up users.

```python
# auth0.py
from auth0.authentication import Database, GetToken

from src.core.config import settings


def auth0_authapi_database():
    database = Database(settings.AUTH0_DOMAIN, settings.AUTH0_CLIENT_ID)
    return database


def auth0_authapi_token():
    token = GetToken(
        settings.AUTH0_DOMAIN,
        settings.AUTH0_CLIENT_ID,
        client_secret=settings.AUTH0_CLIENT_SECRET,
    )
    return token
```

```python
# auth.py
from fastapi import APIRouter, Depends
from fastapi.responses import Response
from auth0.authentication import Database, GetToken

from src.core.config import settings
from src.deps.auth0 import auth0_authapi_database, auth0_authapi_token


router = APIRouter()


@router.post("/v1/auth/signup", tags=["auth"])
async def signup(
    auth0api_database: Database = Depends(auth0_authapi_database),
) -> Response:
    response = auth0api_database.signup(
        email="ehsan_1@yopmail.com",
        password="Abc!_1234",
        connection="Username-Password-Authentication",
    )
    return response
```

**Resource Owner Password Flow**

But in order to use sign-in with username and password, the M2M `Application` needs to be given the `password` `Grant Type` in "Auth0 Dashboard" -> "Applications" -> "Applications" -> "YOUR_M2M_APP" -> "Settings" -> "Advanced Settings" -> "Grant Types". This `password` `Grant Type` is what comprises a Resource Owner Password Flow. Now you can sign-in.

```python
# auth.py

# ...

@router.post("/v1/auth/signin", tags=["auth"])
async def signin(
    auth0api_token: GetToken = Depends(auth0_authapi_token),
) -> Response:
    response = auth0api_token.login(
        username="ehsan_1@yopmail.com",
        password="Abc!_1234",
        realm="Username-Password-Authentication",
        audience=settings.AUTH0_API_AUDIENCE,
    )
    return response
```

**Client Credentials Flow**

By default, an M2M only has the `Client Credentials` grant type. In the Client Credentials Flow, you just provide the `Application` client id and secret to obtain access tokens from Auth0. As such, the access token has no user information, it is an access token just for the Application itself authorised to use some `API` declared in Auth0. Hence, the term Machine to Machine.

**Authorization Code Flow**

Now, it will be ok to go forward with the M2M Application created for us and use its client_id to initialise the SDK. But if you have plans in the future for your API Server to be consumed by a frontend client where Authorization Code Flow might be adopted (client provides backend with an OAuth `code`, which the backend then exchanges with Auth0 for an access token), then you also need the `Authorization Code` Grant Type.

### A Regular Web App

But instead of adding yet another Grant Type `Authorization Code` to our M2M application (which is only meant to have the `Client Credentials` grant type), we can create a new `Application` of type `Regular Web App`, which by default will have the `Authorization Code` and `Client Credentials` grant types amongst others. We need to add the `password` grant type manually here as well.

> So technically, any one of Resource Owner Password or Client Credentials or Authorization Code flows can be used with both M2M and Regular Web App type Applications.
{: .prompt-info }


### Granting Auth0 Management API permission to M2M Application

You will also need the Auth0 Management API for user CRUD operations. In `Auth0 Dashboard -> Applications -> APIs -> Auth0 Management API -> Machine To Machine Applications`, first authorize your M2M application to request access tokens for the Management API. Then expand the M2M Application to also add the required permissions (scopes).