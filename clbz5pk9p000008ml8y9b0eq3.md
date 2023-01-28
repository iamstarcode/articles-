# Refresh Token rotation in NestJS JWT authentication

When creating server-side applications one of the things that come up in the early stages of development is authentication. Setting up a proper authentication can be quite daunting, you've got to decide how you secure your API properly, the strategy you intend to implement, and the type of application that consumes the API. JWTs are most commonly used to identify an authenticated user. They are issued by an authentication server and are consumed by the client (to secure its APIs).

JWTs contain JSON objects which have the information that needs to be shared. Each JWT is also signed using cryptography (hashing) to ensure that the JSON contents (also known as JWT claims) cannot be altered by the client or a malicious party.

### **Access token and Refresh token**

In token-based authentication when users authenticate using sign-in credentials or any third-party provider, access tokens are generated which allows users to stay authenticated, however, access tokens are usually short-lived, this is done for various security reasons such as when an access token is compromised, an attacker has a short time to which it can use the stolen token. Depending on implementation for storage consideration, it is best tokens are properly stored, access tokens are best stored in memory and refresh tokens are best stored as HTTP-only cookies to prevent XXS(Cross-Site-Scripting) attacks and for mobile applications, stored in platform secure stores.

### **Refresh Token**

To avoid users to keep providing sign-in credentials whenever an access token expires, a refresh token can be used to mitigate this, on sign-in or sign-up a pair of both access and refresh tokens will be generated, and the access token with a relatively short amount of time, usually less than an hour while the refresh token could be as long as weeks, or months before it expires. Using refresh token rotation, security threats can be further reduced, every time an application exchanges a refresh token to get a new access token, a new refresh token is also returned along with a new access token, and the old one is invalidated. Therefore, you no longer have a long-lived refresh token that, if compromised, could provide illegitimate access to resources. A detailed explanation can be found here on the [auth0](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation) website.

### **Setup**

In a local development environment, with docker, an instance of PostgreSQL container is used to run a local database which will be used to store user data, [Prisma ORM](https://www.prisma.io) will be used to create both users table and tokens schema, and also used to work with the database.

```yaml
services:
  db:
    image: postgres
    ports:
      - 5432:5432
    restart: always
    environment:
      POSTGRES_DB: nest
      POSTRGRES_USER: root
      POSTGRES_PASSWORD: root
  cache:
    image: redis
    restart: always
    ports:
      - '6379:6379'
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
```

/docker-compose.yml

```pgsql
model User {
  id                               Int        @id @default(autoincrement())
  email                            String?    @unique
  password                         String?
  Token[]
  firstName                        String?
  lastName                         String?
  createdAt                        DateTime   @default(now())
  updatedAt                        DateTime   @updatedAt

  @@map("users")
}

model Token {
  id           String   @id @default(uuid())
  userId       Int
  refreshToken String
  expiresAt    DateTime
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("tokens")
}
```

*/prisma/schema.prisma*

Primarily JWTs are statelessness authentication i.e. You don't need to perform a lookup in the database for validating tokens, they can stay on the front-end app and continue to be used for making requests to API, but poses a threat if there's ever a leak of these tokens if not properly implemented on the front-end. Depending on use cases, the tokens table could be removed and have the refreshToken field on the users table, this way we can prevent logging in on multiple devices. The tokens table is used to store a token per app, making it supports multiple device login and also to make it able to detect a breached or compromised refresh token by identifying an invalid refresh token usage(Reuse detection).

[NestJS](https://docs.nestjs.com/security/authentication) uses [passport](https://www.passportjs.org/) under the hood to implement its authentication layers which support the jwt strategy, it handles the signing, decrypting, and validation of JWTs within the application. The `@nestjs/passport` module provides a built-in Guard. This Guard invokes the Passport strategies and decides if the incoming request will be handled or not. Two strategies are going to be created `JwtStrategy` and `JwtRefreshTokenStrategy`. `JwtStrategy` will be used to secure protected routes, and `JwtRefreshTokenStrategy` will be used to validate and verify a refresh token before actually returning a new pair of access and refresh tokens.

```javascript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

*src/guards/jwt-guard.ts*

```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export default class JwtRefreshGuard extends AuthGuard('jwt-refresh-token') {}
```

*src/guards/jwt-refresh-guard.ts*

When a request is sent to the API, and the endpoint is [**guarded**](https://docs.nestjs.com/guards) with `JwtAuthGuard`, this guard will invoke the `JwtStrategy` and extract the access token from the authorization bearer header token, it then checks if it's a valid token and hasn't expired before handling the request using the `JWT_ACCESS_TOKEN_SECRET` secret to decrypt JWT token.

```javascript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: config.get('JWT_ACCESS_TOKEN_SECRET'),
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, email: payload.email };
  }
}
```

*src/strategies/jwt-strategy.ts*

When an access token has expired the front-end app is expected to send a request to the `/refresh` endpoint with both the "Token-Id" and "Authorization" header from the request containing both the tokenId and refresh token, the `/refresh` end-point will be guarded by the `JwtRefreshGuard` which calls the `JwtRefreshTokenStaragety`. The `JwtRefreshTokenStaragety` strategy before running the validate function checks if the refresh token hasn't expired and also if it's a valid refresh token by using `JWT_REFRESH_TOKEN_SECRET` secret which is used to encrypt and decrypt JWT tokens in the app. In `JwtRefreshTokenStaragety` constructor, we set `passReqToCallback` to true, so we can be able to pass the request object to the validate function. We can then be able to access both the refresh token and tokenId from the header which can then be passed to the `authService.getUserIfRefreshTokenMatches` for an attempt to return a new pair of both access token and refresh token.

```javascript
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Request } from 'express';
import { ITokenPayload } from '../interfaces/ITokenPayload';
import { AuthService } from '../auth.service';

@Injectable()
export class JwtRefreshTokenStrategy extends PassportStrategy(
  Strategy,
  'jwt-refresh-token',
) {
  constructor(configService: ConfigService, private authService: AuthService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get('JWT_REFRESH_TOKEN_SECRET'),
      passReqToCallback: true,
    });
  }

  async validate(request: Request, payload: ITokenPayload) {
    const refreshToken = request.header('Authorization').split(' ')[1];
    const tokenId = request.header('Token-Id');
    return this.authService.getUserIfRefreshTokenMatches(
      refreshToken,
      tokenId,
      payload,
    );
  }
}
```

*src/strategies/jwt-refresh-token-strategy.ts*

The `getUserIfRefreshTokenMatches` performs various checks before actually sending a new pair of both access and refresh tokens. We first check if the tokenId associated with the refresh token is present in the tokens table if not we throw an unauthorized exception. On sign-in, sign-up or token refresh, the refresh token is encrypted using argon2 before actually being stored in a database in case there's ever a database leak. If the refresh token matches, we can then generate a new pair of both access and refresh tokens.

Due to network latency or any other issues, sometimes a refresh token might have been used which makes it invalidated, and therefore not in the database anymore, this can occur when maybe when a user runs multiple tabs of the front-end application in a browser or sometimes a lag in a network and trying to use an already invalidated refresh token, will result in an error. To mitigate this we can allow a leeway that allows the usage of a refresh token even if not in the database but is valid and issued less than a minute ago. And if it is issued over a minute and isn't in the database, this could mean an attacker is trying to **re-use** an already invalidated refresh token. When this happens we can decide to sign out the user from all the devices previously signed on and force a re-authentication process, or take any other kind of action.

```javascript
    async getUserIfRefreshTokenMatches(
    refreshToken: string,
    tokenId: string,
    payload: ITokenPayload,
  ) {
    const foundToken = await this.prismaService.token.findUnique({
      where: {
        id: tokenId,
      },
    });

    const isMatch = await argon.verify(
      foundToken.refreshToken ?? '',
      refreshToken,
    );

    const issuedAt = dayjs.unix(payload.iat);
    const diff = dayjs().diff(issuedAt, 'seconds');

    if (foundToken == null) {
      //refresh token is valid but the id is not in database
      //TODO:inform the user with the payload sub
      throw new HttpException('Unauthorized', HttpStatus.UNAUTHORIZED);
    }

    if (isMatch) {
      return await this.generateTokens(payload, tokenId);
    } else {
      //less than 1 minute leeway allows refresh for network concurrency
      if (diff < 60 * 1 * 1) {
        console.log('leeway');
        return await this.generateTokens(payload, tokenId);
      }

      //refresh token is valid but not in db
      //possible re-use!!! delete all refresh tokens(sessions) belonging to the sub
      if (payload.sub !== foundToken.userId) {
        //the sub of the token isn't the id of the token in db
        // log out all session of this payalod id, reFreshToken has been compromised
        await this.prismaService.token.deleteMany({
          where: {
            userId: payload.sub,
          },
        });
        throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
      }

      throw new HttpException('Something went wrong', HttpStatus.BAD_REQUEST);
    }
  }
```

```typescript
  private async generateTokens(payload: ITokenPayload, tokenId: string) {
    const { accessToken } = await this.getJwtAccessToken(
      payload.sub,
      payload.email,
    );

    const { refreshToken: newRefreshToken } = await this.getJwtRefreshToken(
      payload.sub,
      payload.email,
    );

    const hash = await argon.hash(newRefreshToken);

    await this.prismaService.token.update({
      where: {
        id: tokenId,
      },
      data: {
        refreshToken: hash,
      },
    });

    return {
      accessToken,
      refreshToken: newRefreshToken,
      tokenId: tokenId,
      accessTokenExpires: getAccessExpiry(),
      user: {
        id: payload.sub,
        email: payload.email,
      },
    };
  }
```

```typescript
  async getJwtAccessToken(
    sub: number,
    email?: string,
    isSecondFactorAuthenticated = false,
  ) {
    const payload: ITokenPayload = { sub, email, isSecondFactorAuthenticated };
    const accessToken = await this.jwtService.signAsync(payload, {
      secret: this.configService.get('JWT_ACCESS_TOKEN_SECRET'),
      expiresIn: JWT_ACCESS_TOKEN_EXPIRATION_TIME,
    });
    return {
      accessToken,
    };
  }
```

```typescript
  public async getJwtRefreshToken(sub: number, email: string) {
    const payload: ITokenPayload = { sub, email };
    const refreshToken = await this.jwtService.signAsync(payload, {
      secret: this.configService.get('JWT_REFRESH_TOKEN_SECRET'),
      expiresIn: JWT_REFRESH_TOKEN_EXPIRATION_TIME,
    });
    return {
      refreshToken,
    };
  }
```

*src/auth/auth.service.ts*

### Summary

Implementing authentication in web applications is a never-ending process and should be carefully looked at when implemented as it can lead to damaging data breaches. JWTs are the latest widely authentication used when compared to the older session-based authentication, with refresh token rotation which enforces short-lived tokens that can further strengthen security vulnerabilities. There are more ways by which one can further strengthen applications security, such as implementing rate limiting, using two-factor authentication or prompting users for passwords when an application has been idle for a long time etc.

The full source code can be found here 👇🏽

[Source code](https://github.com/iamstarcode/refresh-token-rotation-nest).