---
id: provider
title: Keycloak Authentication Provider
sidebar_label: Keycloak
description: Adding Keycloak as an authentication provider in Backstage
---

Backstage can authenticate users using [Keycloak](https://www.keycloak.org/)
OpenID Connect. This provider is available through the community-maintained
[`@backstage-community/plugin-auth-backend-module-keycloak-provider`](https://github.com/backstage/community-plugins/tree/main/workspaces/keycloak/plugins/auth-backend-module-keycloak)
module.

## Create a client on Keycloak

To add Keycloak authentication, you must create a client in your Keycloak
realm:

1. In the Keycloak admin console, open the realm you want Backstage to
   authenticate against.
2. Navigate to **Clients** and click **Create client**.
3. Fill out the **General settings** form:
   - **Client type**: `OpenID Connect`
   - **Client ID**: `backstage` (or a name of your choice)
   - Click **Next**
4. Fill out the **Capability config** form:
   - Enable **Client authentication**
   - Keep **Standard flow** enabled
   - Click **Next**
5. Fill out the **Login settings** form:
   - **Valid redirect URIs**:
     `http://localhost:7007/api/auth/keycloak/handler/frame`
   - **Valid post logout redirect URIs**: `http://localhost:7007`
   - **Web origins**: `http://localhost:3000`
   - Click **Save**
6. Open the **Credentials** tab and copy the generated **Client secret**.

The configuration examples provided above are suitable for local development.
For a production deployment, substitute `http://localhost:7007` with the URL
that your Backstage backend is available at, and `http://localhost:3000` with
the URL of the Backstage frontend.

## Configuration

The provider configuration can then be added to your `app-config.yaml` under
the root `auth` configuration:

```yaml
auth:
  environment: development
  providers:
    keycloak:
      development:
        clientId: ${AUTH_KEYCLOAK_CLIENT_ID}
        clientSecret: ${AUTH_KEYCLOAK_CLIENT_SECRET}
        baseUrl: ${AUTH_KEYCLOAK_BASE_URL}
        realm: ${AUTH_KEYCLOAK_REALM}
        signIn:
          resolvers:
            # See https://backstage.io/docs/auth/keycloak/provider#resolvers for more resolvers
            - resolver: preferredUsernameMatchingUserEntityName
```

The Keycloak provider is a structure with these configuration keys:

- `clientId`: The client ID that you registered on Keycloak, for example
  `backstage`.
- `clientSecret`: The client secret shown on the **Credentials** tab of the
  client.
- `baseUrl`: The base URL of the Keycloak server, without a trailing
  `/realms/...` path, for example `https://keycloak.example.com`.
- `realm`: The name of the Keycloak realm that Backstage authenticates
  against.

Optional configuration such as `additionalScopes`, `postLogoutRedirectUri`,
and `prompt` is described in the
[module documentation](https://github.com/backstage/community-plugins/tree/main/workspaces/keycloak/plugins/auth-backend-module-keycloak#configuration).

### Resolvers

This provider includes several resolvers out of the box that you can use:

- `emailMatchingUserEntityProfileEmail`: Matches the email address from the
  auth provider with the User entity that has a matching
  `spec.profile.email`. If no match is found it will throw a `NotFoundError`.
- `emailLocalPartMatchingUserEntityName`: Matches the
  [local part](https://en.wikipedia.org/wiki/Email_address#Local-part) of the
  email address from the auth provider with the User entity that has a
  matching `name`. If no match is found it will throw a `NotFoundError`.
- `preferredUsernameMatchingUserEntityName`: Matches the `preferred_username`
  claim from Keycloak with the User entity that has a matching `name`. If no
  match is found it will throw a `NotFoundError`.
- `oidcSubClaimMatchingKeycloakUserId`: Matches the OIDC `sub` claim from
  Keycloak with the User entity where the value of the `keycloak.org/id`
  annotation matches. Use this resolver together with the Keycloak catalog
  module, which adds that annotation to ingested users.
- `ldapUuidMatchingAnnotation`: Matches an LDAP UUID claim from Keycloak with
  the User entity where the value of the `backstage.io/ldap-uuid` annotation
  matches. The required Keycloak mapper setup is described in the
  [module documentation](https://github.com/backstage/community-plugins/tree/main/workspaces/keycloak/plugins/auth-backend-module-keycloak#match-users-by-ldap-uuid-keycloak).

:::note Note

The resolvers will be tried in order, but will only be skipped if they throw a `NotFoundError`.

:::

If these resolvers do not fit your needs you can build a custom resolver, this
is covered in the
[Building Custom Resolvers](../identity-resolver.md#building-custom-resolvers)
section of the Sign-in Identities and Resolvers documentation.

## Backend installation

To add the provider to the backend we will first need to install the package
by running this command:

```bash title="from your Backstage root directory"
yarn --cwd packages/backend add @backstage-community/plugin-auth-backend-module-keycloak-provider
```

Then we will need to add this line:

```ts title="in packages/backend/src/index.ts"
backend.add(import('@backstage/plugin-auth-backend'));
/* highlight-add-start */
backend.add(
  import('@backstage-community/plugin-auth-backend-module-keycloak-provider'),
);
/* highlight-add-end */
```

## Synchronizing users and groups

The sign-in resolvers require a matching User entity to already exist in the
Software Catalog. The recommended way to achieve this is to install the
community-maintained
[`@backstage-community/plugin-catalog-backend-module-keycloak`](https://github.com/backstage/community-plugins/tree/main/workspaces/keycloak/plugins/catalog-backend-module-keycloak)
plugin, which synchronizes Keycloak users and groups into the catalog on a
schedule. See
[Keycloak Organizational Data](../../integrations/keycloak/org.md) for more
information.

## Adding the provider to the Backstage frontend

Backstage does not ship a built-in auth API for Keycloak, so you need to
create and register a Keycloak auth API reference in your app. The
[module documentation](https://github.com/backstage/community-plugins/tree/main/workspaces/keycloak/plugins/auth-backend-module-keycloak#adding-the-provider-to-the-backstage-frontend)
contains a complete example. Then add the `SignInPage` component as shown in
[Adding the provider to the sign-in page](../index.md#sign-in-configuration).
