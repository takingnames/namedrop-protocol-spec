# The NameDrop Protocol (draft version 0.1.0)

NameDrop is developed by [TakingNames.io][0] for delegating control over DNS
domains and subdomains. It is an open protocol, and implementation by others
is encouraged.


# Overview

NameDrop is based on [OAuth2][1], with a few additions to facilitate domain
name delegation.

One key difference from how OAuth2 is generally implemented, is that client
registration with the authorization server is not required before initiating
grant flows. To maintain a level of security, it is required that the
`client_id` be a prefix string of the `redirect_uri`, ie the `redirect_uri`
(which is where the token ends up) must be on the same domain as the
`client_id`. This method is essentially what is described [here][2]. NameDrop
authorization servers should display the `client_id` to users and inform them
that is who is requesting access.

All API endpoints described in this article are assumed to be appended to a
base URL. For example, TakingNames.io uses

`https://takingnames.io/namedrop`

It is not necessary for the API name to start with `/namedrop`, but it can
be useful for namespacing if the server has other non-NameDrop endpoints.

# OAuth2 scopes

NameDrop currently only supports a single OAuth2 scope: `subdomain`. This
scope requests complete control over a domain or subdomain. There will likely
be additional scopes added in the future, such as `wildcard`, `dyndns`, scopes
that restrict to specific record types, etc.


# OAuth2 endpoints

The basic OAuth2 endpoints are defined as follows:

**`GET /authorize`**

Authorization endpoint (user consent to get code). Can be a web browser
redirect, or a direct link, such as one printed from a CLI application.


**`GET /code`**

Redirect endpoint (where code is returned on client). Always a web browser
redirect.


**`POST /token`**

Token endpoint (swap code for token). Always server-to-server.


# Other endpoints

**`GET /token-data`**

Retrieves data for the token used in the request. This is critical for the
client application to determine what permissions have been granted.


**`PUT /records`**

Creates a new record. The provided token must have the proper permissions.


**`GET /my-ip`**

Returns the public IP of the client, as seen from the server. This is useful
for helping self-hosted clients test whether they can be reached by the outside
world.



[0]: https://takingnames.io

[1]: https://oauth.net/2/

[2]: https://aaronparecki.com/2018/07/07/7/oauth-for-the-open-web
