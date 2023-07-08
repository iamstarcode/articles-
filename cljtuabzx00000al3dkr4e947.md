---
title: "Using Supabase authentication in NestJs"
datePublished: Sat Jul 08 2023 10:05:00 GMT+0000 (Coordinated Universal Time)
cuid: cljtuabzx00000al3dkr4e947
slug: using-supabase-authentication-in-nestjs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1688810301061/038f5b96-b665-47a5-aadb-d1fca30184b7.png
tags: authentication, baas, nestjs, supabase

---

Not too long ago I decided to write my backend authentication code for an application I was building. Authentication in NestJs is less of a hassle as it uses the popular easy-to-use [passport.js](https://www.passportjs.org/) library, which is pretty solid when working with authentication in NodeJs applications. Setting up a backend authentication could seem like an easy task to do, however, you'll quickly realize you might have to keep revisiting the same code over and over again due to bugs or if you intend to add more features, this could cause a longer time to ship products. With Supabase you can set up authentication pretty easily and fast, and use it as authorization for your backend.

# **Backend-as-a-Service** (**BaaS**)

According to Wikipedia "**Backend as a service** (**BaaS**), also known as **mobile backend as a service** (**MBaaS**), is a service for providing web app and mobile app developers with a way to easily build a backend for their frontend applications. Features available include user management, push notifications, and integration with social networking services. Supabase is one of the many popular services that provide authentication and other features including user management.

You can use many of its services independently when working with **BaaS**, developers tend to worry about how they could end up being vendor locked, unlike its "popular" alternative Firebase, Supabase is a combination of open-source packages and it's built around the PostgreSQL database. They provide a hosting service but you can host your application anywhere yourself and however you see fit.

# Authentication and Authorization

Authentication in Supabase can be initiated on the client, an app can be written entirely in the front-end app. However, some actions might require the ability to run server-side code, an example could be when trying to create a signed upload in Cloudinary, signed uploads can only be done server-side. To authorize its endpoint, Supabase uses JWTs(JSON web tokens) and RLS(Row Level Security) for authorization. After successful authentication, Supabase access tokens can be used to authorize a client to access a resource on a server using its JWT secret to verify and validate the access tokens.

### **NestJs Setup**

To set up a new NestJs project, first globally install Nest CLI and create a new project using the following command.

```bash
npm i -g @nestjs/cli
nest new project-name
```

and choose your preferred package manager when shown a prompt. Nest integrates passport.js for handling different authentication strategies, and the JWT strategy has been one of them. We'll need to install additional packages for our project.

```bash
npm i @nestjs/config @nestjs/passport passport-jwt @supabase/supabase-js
npm i --save-dev @types/passport-jwt
```

# Supabase Setup

In the course of this article, we will be using the Supabase local development setup. However, you can use the online dashboard to get up and running quickly. Developing locally comes with a few benefits such as faster development as there would be no need to connect with any network etc.

Install [Supabase CLI](https://supabase.com/docs/guides/cli) based on your preferred platform and initialize the Supabase project in the root of the NestJs project using the `supabase init` command, and run `supabase start`. Make sure you have Docker installed and running as this command when running the first time downloads the required Docker images to run local containers of some Supabase services including PostgreSQL and a local dashboard. Once started and all services are running, you will see an output containing all local credentials which would look like this containing URLs and keys of your local projects.

```bash
Started supabase local development setup.

         API URL: http://localhost:54321
          DB URL: postgresql://postgres:postgres@localhost:54322/postgres
      Studio URL: http://localhost:54323
    Inbucket URL: http://localhost:54324
      JWT secret: super-secret-jwt-token-with-at-least-32-characters-long
        anon key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
        service_role key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

# Writing Code

Now that we have the required packages installed and Supabse local development set up, we need to first create a `SupabaseModule` module and a `SupabaseService` service. To create a module and a service, we can use the Nest CLI. The `SupabaseModule` module will be added to our `App` module automatically when created using the CLI.

```bash
nest g module supabase
nest g service supabase
```

We need to create a passport strategy and an authorization guard to protect routes within our application and some environment variables to properly secure sensitive information.

Create a `.env` file in the root of the project and paste the API URL, JWT secrete, and anon key as `SUPABASE_API_URL`, `SUPABASE_JWT_SECRTE`, and `SUPABASE_ANON_KEY` environment values respectively as seen when the `supabase status` command was run.

```bash
SUPABASE_API_URL=http://localhost:54321
SUPABASE_JWT_SECRTE=super-secret-jwt-token-with-at-least-32-characters-long
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The `ConfigService` is injected into our strategy so that we can get our JWT secret, used to verify and validate the JWT token extracted from the request headers before the incoming request can be handled.

```typescript
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class SupabaseStrategy extends PassportStrategy(Strategy) {
  constructor(configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get('SUPABASE_JWT_SECRET'),
    });
  }

  async validate(request: Request) {
    return request;
  }
}
```

*./src/supabase/strategy/supabase.strategy.ts*

We can then use our strategy to create a guard, which can be used to protect routes.

```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class SupabaseGuard extends AuthGuard('jwt') {}
```

*./src/supabase/guard/supabase.guard.ts*

The `SupabaseGuard` Depending on the use case, we can be method-scoped, controller-scope or global-scoped.

In the AppModule we then add all required modules used throughout our entire application, by adding them to the `imports` array of the `@Module` decorator object. To make the guard global-scoped, we add an object of properties `provide` and `useClass` with values `APP_GUARD` and `SupabaseGuard` respectively to the provider's array of the of the `@Module` decorator object.

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { SupabaseModule } from './supabase/supabase.module';
import { ConfigModule } from '@nestjs/config';
import { PassportModule } from '@nestjs/passport';
import { APP_GUARD } from '@nestjs/core';
import { SupabaseGuard } from './supabase/guards/supabase.guard';

@Module({
  controllers: [AppController],
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
    }),
    PassportModule,
    SupabaseModule,
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: SupabaseGuard,
    },
    AppService,
  ],
})
export class AppModule {}
```

*./src/app.module.ts*

### Extra!

Ordinarily, all Supabase actions can be performed from the front end, you can however still be able to perform these actions server-side. To do this we would have to create a `supabase-js` client object using the access token gotten from the authorization request bearer header. We can implement this in our `SupabaseService`.

```typescript
import { Inject, Injectable, Scope } from '@nestjs/common';
import { Request } from 'express';
import { REQUEST } from '@nestjs/core';
import { ConfigService } from '@nestjs/config';

import { createClient, SupabaseClient } from '@supabase/supabase-js';

import { ExtractJwt } from 'passport-jwt';
import { Database } from '../../lib/database.types';

@Injectable({ scope: Scope.REQUEST })
export class SupabaseService {
  private clientInstance: SupabaseClient;

  constructor(
    @Inject(REQUEST) private readonly request: Request,
    private readonly configService: ConfigService,
  ) {}

  async getClient() {
    if (this.clientInstance) {
      return this.clientInstance;
    }

    this.clientInstance = createClient<Database>(
      this.configService.get('SUPABASE_API_URL'),
      this.configService.get('SUPABASE_ANON_KEY'),
      {
        auth: {
          persistSession: false,
        },
        global: {
          headers: {
            Authorization: `Bearer ${ExtractJwt.fromAuthHeaderAsBearerToken()(
              this.request,
            )}`,
          },
        },
      },
    );
    return this.clientInstance;
  }
}
```

*./src/supabase/supabase.service.ts*

In our use case, we want a request-based lifetime for our `supabse-js` client instances. In NestJs, injection scopes provide a mechanism for a request-based lifetime behavior. To make that possible we pass the injection scope property to the `@Injectable()` decorator object. We also need to access a reference to the original request object. We do this by injecting the `REQUEST` object, because the authentication bearer token will be part of the request authorization header. We can then extract the bearer token from the request header and use it to create `supabase-js` client instances to perform any other actions per request.

```typescript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';
import { SupabaseService } from './supabase/supabase.service';

@Controller()
export class AppController {
  constructor(
    private readonly appService: AppService,
    private readonly supabaseService: SupabaseService,
  ) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @Get('/client')
  async getClient(): Promise<any> {
    const supabaseClient = await this.supabaseService.getClient();

    const { data, error } = await supabaseClient
      .from('products')
      .select()
      .single();

    console.log(data);
    return data;
  }
}
```

### Conclusion

Shipping products faster is probably almost the most important factor when developing applications, clients aren't really worried about what tools you used, but how efficiently and fast products are delivered, **BaaS** services help reduce project deployment time and probably help reduce bugs, and developers tend to focus more on the business logic of projects. As with everything in tech, Backend as a Service (BaaS) brings forth the yin-yang of efficiency and inflexibility. On one hand, it streamlines development processes by providing ready-made backend infrastructure, saving time and resources. On the other hand, it might hinder developers' freedom to customize and integrate additional functionalities as needed.

The complete source code can be found here. 👇🏽

[https://github.com/iamstarcode/nestjs-supabase-auth](https://github.com/iamstarcode/nestjs-supabase-auth)