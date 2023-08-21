# Okta

You can sign up to an [Okta developer account](https://developer.okta.com/) to get started quickly with an OIDC-compliant identity provider,
and run some authentication and authorisation tests from your TKG platform.

The simplest use case is to have users authenticate against Okta, retrieve the groups they belong to and assign Kubernetes permissions based on groups.
RBAC is still configured on the Kubernetes end, whilst the authentication part is completely delegated to the IdP.

## Create a new application

Log into your Okta developer account and go to `Applications` -> `Applications` -> `Create a new app integration`,
select `OIDC - OpenID Connect` as sign-in method and `Web Application` as application type.

Pick a name for the app (i.e. `tkgm`), enable the refresh token amd save; you will have to [set the sign-in redirect URI](./identity-management.md#configure-redirect-uri) when pinniped is up and running.
Save the Client ID and the Client Secret, you will need them for configuring pinniped.

## Create users and groups

Now you may want to configure a few groups in `Directory` -> `Groups` and assign people to them, as well as the application you just created,
so that group members will be able to authenticate to your cluster.
Initially you have just one user, whose username is the email address you registered to Okta with,
but you can also create more if you wish, if you want to simulate a multi-user multi-group scenario.

## Configure authorization server

You need to ensure that group memberships are passed along in the JWT token when a user authenticates,
and you must instruct the authorization server to do so.

Go to `Security` -> `API` and hit the pencil next to the `default` authorization server to modify it.
Go to `Claims` and add a new claim: the name must be `groups`, included in `ID Token` type `Always`,
and you may want to pass all the groups, with no filtering whatsoever.
So the value type must be `Groups` and the filter `Matches regex` `.*`, and it must be included in `Any scope`.

You can test the settings in the `Token Preview` tab.

Further details are available in the [official Okta docs](https://developer.okta.com/docs/guides/customize-tokens-groups-claim/main/).

## Define variables

You can now define some variables that will be used for configuring Pinniped,
replacing the `OKTA_*` placeholders with the values specific to your app:

```sh
IDP_NAME="okta"
OIDC_ISSUER_URL="https://OKTA_DOMAIN"
OIDC_CLIENT_ID="OKTA_CLIENT_ID"
OIDC_CLIENT_SECRET="OKTA_CLIENT_SECRET"
OIDC_GROUPS_CLAIM="groups"
OIDC_USERNAME_CLAIM="email"
OIDC_SCOPES="email,profile,openid,groups"
```
