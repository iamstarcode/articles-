---
title: "Using Supabase Authorization with FastAPI"
datePublished: Wed Jan 31 2024 13:34:56 GMT+0000 (Coordinated Universal Time)
cuid: cls1txnkv000609jl78bwfpk9
slug: using-supabase-authorization-with-fastapi
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1706707979262/6a8ca98c-0c73-425d-bd6c-809dc0f663b1.png
tags: fastapi, supabase

---

# Introduction

Authentication is one of the most important parts when writing backend projects, you can either spin up one yourself or make use of providers that provide authentication services like [Supabase](https://supabase.com/). [FastAPI](https://fastapi.tiangolo.com/), known for its high performance and automatic OpenAPI and JSON Schema generation, meets Supabase, a robust open-source alternative to Firebase, in a collaborative effort to streamline the development of modern applications.  
Throughout this article, we'll walk through the steps of setting up Supabase for authentication, configuring FastAPI to work seamlessly with Supabase, and implementing robust authorization mechanisms to secure your application.

# **Supabase Setup**

It's ideal to use Supabase local development to set up a Supabase project, however, we can head over to [Supabase](https://supabase.com/) to create an account and create a project and have access to the online dashboard. To set up a local development Supabase project, we can use the [Supabase CLI](https://github.com/supabase/cli). Once we have the CLI installed, we can then use the `supabase init` command to create a project in any folder of ours. The CLI relies on [Docker](https://www.docker.com/) and must be running when trying the command, for the first time running, the CLI will proceed to download required Docker images to run local containers of some Supabase services including PostgreSQL and a local dashboard.

After the init command, we can then run the command `supabase start`, this command will start all Supabase services and print local credentials including keys and URLs.

```bash
Started supabase local development setup.

         API URL: http://localhost:54321
     GraphQL URL: http://127.0.0.1:54321/graphql/v1
          DB URL: postgresql://postgres:postgres@localhost:54322/postgres
      Studio URL: http://localhost:54323
    Inbucket URL: http://localhost:54324
      JWT secret: super-secret-jwt-token-with-at-least-32-characters-long
        anon key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
        service_role key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

# FastAPI setup

As of the time of writing this article, **FastAPI** requires at least a **Python 3.8** version to run. To create a FastAPI project, we will set up a virtual environment, this way, packages installed in our project won't conflict with global python installation. To create a virtual environment, we will run this command at the root of our project.

```bash
python -m venv .venv
## or
python3 -m venv .venv
```

We can then activate the virtual environment.

**On Windows:**

```bash
.venv\Scripts\activate
```

**On macOS and Linux:**

```bash
source .venv/bin/activate
```

We will need a host of packages to install alongside **FastAPI,** including **supabase**, **pydantic-settings** and **PyJWT**. At the root of our project, we will create a requirements.txt file containing our required packages.

```ini
fastapi
uvicorn[standard]
supabase
pydantic-settings
PyJWT
```

To install the packages, we run the command.

```bash
pip install -r requirements.txt
```

It is good practice to have your API keys stored in environment variables rather than have them hard-coded in our code. To load environment variables, **FastAPI** uses **Pydantic**, which provides an optional Pydantic feature for loading a settings or config class from environment variables or secrets files through the **pydantic-settings** package.

In the root of our project, we will create a `config.py` and a dotenv file where we will have our `SUPABASE_URL`, `SUPABASE_ANON_KEY` and `SUPABASE_JWT_SECRET` secret.

```ini
SUPABASE_JWT_SECRET=super-secret-jwt-token-with-at-least-32-characters-long
SUPABASE_URL=http://127.0.0.1:54321
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZS1kZW1vIiwicm9sZSI6ImFub24iLCJleHAiOjE5ODM4MTI5OTZ9.CRXP1A7WOeoJeXxjNni43kdQwgnWNReilDMblYTn_I0
```

*/.env*

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    supabase_url: str
    supabase_anon_key: str
    supabase_jwt_secret: str
    model_config = SettingsConfigDict(env_file=".env")
```

*/config.py*

Given a particular secret key in a dotenv file, the **Settings** class automatically loads its value into its properties using the snake case representation of the key.

**Validating JWT**

When you sign in using Supabase authentication, Supabase issues JWT tokens(access and refresh tokens) to maintain authorization without having to sign in users over and over again, as long as we can verify the JWT token with the JWT secret that was used to sign the token. At the root of the project, we will create an `auth.py`.

```python
from typing import Annotated
from fastapi import HTTPException, Header
import jwt
from config import Settings
settings = Settings()

def validate_jwt(authorization: Annotated[str, Header()]) -> str:
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=400, detail="Invalid Authorization")
    access_token = authorization.split(" ")[1]
    if not access_token:
        raise HTTPException(status_code=400, detail="Invalid Authorization")
    try:
        jwt.decode(
            access_token,
            settings.supabase_jwt_secret,
            algorithms=["HS256"],
            options={"verify_aud": False, "verify_signature": True},
        )

    except jwt.InvalidSignatureError as e:
        print(f"Error: {e}")
        raise HTTPException(status_code=400, detail="Invalid Authorization")
    else:
        return access_token
```

*/auth.py*

In any frontend app where Authentication has been successful, it is common to have an access token sent over the authorization bearer header when making requests to backend applications. In the `validate_jwt` function we use the `jwt.decode` function to decode the JWT access token obtained from the authorization bearer header using the `settings.supabase_jwt_secret` of the Settings class and return the `access_token`.

**Creating Supabase client**

Before creating a Supabase client, we will need to use the access token obtained for the `validate_jwt` function. **FastAPI** has a very powerful but intuitive [**<abbr>Dependency Injection</abbr>**](https://en.wikipedia.org/wiki/Dependency_injection) system. Before creating our Supabase client we "**depend**" on having a valid access token.

```python
from typing import Annotated
from fastapi import Depends, Header

from supabase import create_client, Client
from supabase.client import ClientOptions

from auth import validate_jwt
from config import Settings

settings = Settings()

async def get_supabase_client(
    access_token: Annotated[str, Depends(validate_jwt)],
) -> Client:
    supabase: Client = create_client(
        settings.supabase_url,
        settings.supabase_anon_key,
        options=ClientOptions(
            persist_session=False,
            auto_refresh_token=False,
        ),
    )

    supabase.auth.set_session(access_token, refresh_token="")
    supabase.postgrest.auth(access_token)

    return supabase
```

*/libs/supabase.py*

Our `get_supabase_client` function depends on the `validate_jwt` dependency, and any of our route paths in the API that requires just authorization will depend on the `validate_jwt` dependency, while any route path requires the Supabase client will depend on the s[ub-dependency](https://fastapi.tiangolo.com/tutorial/dependencies/sub-dependencies/) `get_supabase_client`. The access token is used to set a Supabase session, and also the postgrest session as this is required for [RLS(Row Level Security)](https://supabase.com/docs/guides/auth/row-level-security) policies to hit when making Supabase client calls.

**Protecting Route Paths**

To protect routes that require only authorization we can use the `validate_jwt` dependency directly on the `@app` decorator and use the dependencies parameter to set the dependency, while for any route path that requires a Supabase client, we will use the sub-dependency `get_supabase_client` on the route path function and receive the a Supabase client object.

```python
from typing import List
from typing_extensions import Annotated

from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware

from supabase import Client

from libs.supabase import get_supabase_client
from auth import validate_jwt
from models.todo import Todo

origins = [
    "http://127.0.0.1:3000",
]

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/", dependencies=[Depends(validate_jwt)])
async def home():
    return {"msg": "Hello World!"}

@app.get("/todos")
async def get_todos(
    supabase: Annotated[Client, Depends(get_supabase_client)]
) -> List[Todo]:
    todos = supabase.table("todos").select("*").execute()
    return [Todo(**item) for item in todos.data]
```

*/main.py*

Supabase authorization with FastAPI provides a robust solution for building secure and scalable web applications. By leveraging Supabase's authentication and authorization features, developers can offload user management tasks and focus on building the core functionality of their applications.

The full source code can be found here.

[FastAPI app source](https://github.com/iamstarcode/supabase-authorization-fastapi)

A simple Todo app in Next front end can be found here.

[Nextjs Todo app source](https://github.com/iamstarcode/supabase-with-python-backend)