---
title: "Configuring FastAPI for use with PowerBI when using Entra ID for OAuth2."
date: "2026-07-26"
tags: ['Python', 'FastAPI', 'Entra ID', 'PowerBI', 'OAuth2', 'Tutorial']
description: "A guide to enabling a FastAPI integrated with Entra ID to be accessible via PowerBI."
author: ["Naveen Kannan"]
weight: 10
draft: false
ShowToc: true
TocOpen: true
cover:
  image: "https://fastapi.tiangolo.com/img/logo-margin/logo-teal.png"
  alt: "<alt text>"
  caption: "The FastAPI logo."
  relative: false
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
params:
  comments: true
  ShowCodeCopyButtons: true
  ShowReadingTime: true
---

# Introduction

[FastAPI](https://fastapi.tiangolo.com/) is a framework for building APIs with Python based on standard Python type hints. It's fast and easy to stand up, and as a Python developer, it was relatively easy to pick up and get started with it. 

As part of deploying a FastAPI instance in production, I integrated FastAPI with Entra ID for OAuth2 via [this really cool library, FastAPI-Azure-Auth.](https://vibber-ai.github.io/fastapi-azure-auth/)

After integrating this library into my FastAPI instance, I was able to secure access to the API server. However, I quickly ran into an interesting problem. While trying to connect a PowerBI instance to the API, I found that the API settings needed some more configuration to enable interactive logging in via Microsoft SSO within PowerBI. 

## Some context

FastAPI needs to be configured to return a `WWW-Authenticate` header when a user attempts to access the API without authorization (a `HTTP 401 Unauthorized`) error. 

When an unauthenticated access request is made to a secure API, it can be configured to return a header called `WWW-Authenticate` within it's response when it returns a `HTTP 401 Unauthorized` error. 

The `WWW-Authenticate` header returns information that can be used for the user to authenticate themselves and return to the API with a valid, unexpired authentication token. 

As there is no element of interaction with a HTTP API, the onus is on the application/user that is querying the API to extract the information returned in the `WWW-Authenticate` header and to proceed to the identity provider (in this case, Microsoft Entra ID.) 

In our case, when PowerBI (via Power Query) requests access to a Web API via an Organizational account, it sends a request to the endpoint to the URL with an empty bearer token. It then expects a `WWW-Authenticate` header along with the Microsoft Entra ID authorization URI to use. This authorization URI should return the tenant that is to be used for the OAuth2 process. Therefore, all we need to do is configure an error handler within the FastAPI instance to return a `WWW-Authenticate` header when it returns a `HTTP 401 Unauthorized` error.  

The [Power Query documentation](https://learn.microsoft.com/en-us/power-query/connector-authentication) documents the supported workflow when trying to access a Web API via an Organizational account. 

From the documentation, this is the reponse expected by Power Query when it is given a `HTTP 401 Unauthorized` error. 

```
HTTP/1.1 401 Unauthorized
Cache-Control: private
Content-Type: text/html
Server:
WWW-Authenticate: Bearer authorization_uri=https://login.microsoftonline.com/aaaabbbb-0000-cccc-1111-dddd2222eeee/oauth2/authorize
Date: Wed, 15 Aug 2018 15:02:04 GMT
Content-Length: 49
```

So to sum it up, we need to configure FastAPI to return a `WWW-Authenticate` header when it returns a `HTTP 401 Unauthorized` error, and this header needs to return the authorization URI associated with the tenant that is able to access the API service via OAuth2.


## The Solution

Note that this guide assumes that you have already configured your FastAPI instance to work with Entra ID for OAuth2 flows. 

As a general example (which may differ from your set-up), I usually have my code that handles the Entra ID authentication flow within a folder called `modules` in the project root, in a file called `azure.py`, which I can then import into my `main.py` file. 

Following from the basic example within the FastAPI-Azure-Auth documentation, here's what `azure.py` looks like:

```python
from fastapi_azure_auth import SingleTenantAzureAuthorizationCodeBearer
from pydantic import AnyHttpUrl, computed_field
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    BACKEND_CORS_ORIGINS: list[str | AnyHttpUrl] = [
        "https://your.server.com",
    ]
    OPENAPI_CLIENT_ID: str = ""
    APP_CLIENT_ID: str = ""
    TENANT_ID: str = ""
    SCOPE_DESCRIPTION: str = "user_impersonation"

    @computed_field
    @property
    def SCOPE_NAME(self) -> str:
        return f"api://{self.APP_CLIENT_ID}/{self.SCOPE_DESCRIPTION}"

    @computed_field
    @property
    def SCOPES(self) -> dict:
        return {
            self.SCOPE_NAME: self.SCOPE_DESCRIPTION,
        }

    @computed_field
    @property
    def OPENAPI_AUTHORIZATION_URL(self) -> str:
        return (
            f"https://login.microsoftonline.com/{self.TENANT_ID}/oauth2/v2.0/authorize"
        )

    @computed_field
    @property
    def OPENAPI_TOKEN_URL(self) -> str:
        return f"https://login.microsoftonline.com/{self.TENANT_ID}/oauth2/v2.0/token"

settings = Settings()

azure_scheme = SingleTenantAzureAuthorizationCodeBearer(
    app_client_id=settings.APP_CLIENT_ID,
    tenant_id=settings.TENANT_ID,
    scopes=settings.SCOPES,
)

```

> [!NOTE]
> This example doesn't go into how you supply the module with the required `OPENAPI_CLIENT_ID`, `APP_CLIENT_ID` and `TENANT_ID` variables.
> It is good practice to use a secret vault to supply these variables when calling `Settings()` to override the default empty strings.
> For example:
>>
>>```
>>settings = Settings(
>>    APP_CLIENT_ID=app_client_id,
>>    OPENAPI_CLIENT_ID=openapi_client_id,
>>    TENANT_ID=tenant_id,
>>)
>>```
>In this example, the three variables supplied to `Settings()` have been called from a secret vault.

## The Error Handler

Define a custom error handler within your FastAPI code. For example, create a folder called `modules` in your project root, and create a file called `errors.py` within `modules`. Put the following in that file.

```python
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
from modules import azure

async def custom_http_exception_handler(request: Request, exc: HTTPException):
    if exc.status_code == 401:
        auth_url = f"https://login.microsoftonline.com/{azure.settings.TENANT_ID}/oauth2/authorize"
        return JSONResponse(
            status_code=401,
            content={"detail": "Not authenticated"},
            headers={
            "WWW-Authenticate": f'Bearer authorization_uri="{auth_url}"'
            },
        )
    else:
        return JSONResponse(
            status_code=exc.status_code, 
            content={"detail": exc.detail})
```
In this code snippet, we are doing the following:

- When a HTTP Exception occurs, the functions checks to see if a 401 error is returned.
- We define the URL for OAuth2 authorization flow by using the tenant ID taken from the Azure config.
- We construct and return the `WWW-Authenticate` header to be compliant with the format expected by Power Query.
- For HTTP Errors that are not 401, the default response is passed back.

We can then import this error handler in `main.py` as follows:

```python
from typing import Annotated, AsyncGenerator
from contextlib import asynccontextmanager
from starlette.middleware.cors import CORSMiddleware
from fastapi import FastAPI, HTTPException, Security
from fastapi_azure_auth.user import User
from modules import azure
from modules.errors import custom_http_exception_handler

settings = azure.settings
azure_scheme = azure.azure_scheme

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """
    Load OpenID config on startup.
    """
    await azure_scheme.openid_config.load_config()
    yield

app = FastAPI(
    lifespan=lifespan,
    swagger_ui_oauth2_redirect_url="/api/oauth2-redirect",
    swagger_ui_init_oauth={
        "usePkceWithAuthorizationCodeGrant": True,
        "clientId": settings.OPENAPI_CLIENT_ID,
        "scopes": settings.SCOPE_NAME,
    },
)

if settings.BACKEND_CORS_ORIGINS:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=[str(origin) for origin in settings.BACKEND_CORS_ORIGINS],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
app.add_exception_handler(HTTPException, custom_http_exception_handler)

@app.get("/", dependencies=[Security(azure_scheme)])
async def root():
    return {"message": "Hello World"}

if __name__ == '__main__':
    uvicorn.run('main:app', reload=True)
```

With this error handler imported, PowerBI should be able to use the Power Query Connector to enable SSO via your Microsoft login when it is attempting to connect to your API!

# References

- [FastAPI-Azure-Auth documentation](https://vibber-ai.github.io/fastapi-azure-auth/)
- [Microsoft Learn documentation on the Power Query connector and authentication flows](https://learn.microsoft.com/en-us/power-query/connector-authentication)
- [Documenation on the `WWW-Authenticate` header standards](https://www.rfc-editor.org/info/rfc7235/#section-4.1)