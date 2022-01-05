# The NameDrop Protocol (draft version 0.2.0)

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


**`GET /callback`**

Redirect endpoint (where code is returned on client). Always a web browser
redirect.


**`POST /token`**

Token endpoint (swap code for token). Always server-to-server.


# Other endpoints

**`GET /token-data`**

Retrieves data for the token used in the request. This is critical for the
client application to determine what permissions have been granted.

Data is returned as JSON in the following format:

```json
{
  "owner": "<owner identifier>",
  "scopes": [
    {
      "domain": "<domain of scope>",
      "host": "<host of scope>",
    },
    {
      "domain": "<domain of scope>",
      "host": "<host of scope>",
    }
    ...
  ]
}
```

Host values can contain wildcard characters. In this case, the scope grants
permissions for any subdomain which has the host as a suffix, minus the
wildcard character '\*'.


**`PUT /records`**

Creates a new record. The provided token must have the proper permissions.

The request is JSON in the following format:

```json
{
  "domain": "<domain>",
  "host": "<host>",
  "type": "<type>",
  "value": "<value>",
  "ttl": <ttl>,
  "priority": <priority>,
}
```

Where `type` is the record type such as `A`, `CNAME`, `MX`, etc. `ttl` and
`priority` are both integers.


**`GET /my-ip`**

Returns the public IP of the client, as seen from the server. This is useful
for helping self-hosted clients test whether they can be reached by the outside
world.

The IP is returned as a simple string.


**`GET /ip-domain`**

This causes the server to create a special `A` and/or `AAAA` record pointing at
the client's IP address, as seen by the server. The domain must start with the
IP address, but with '.' or ':' characters replaced with '-'. The rest of the
domain can be anything. The created domain is returned as a simple string.

So, for example, TakingNames.io creates the record and returns something like
this:

`157-245-231-242.bootstrap.takingnames.live`

The purpose of these domains is to allow the client to retrieve a TLS
certificate from a service like [LetsEncrypt][3], which makes the OAuth2 flows
more secure. This is particularly useful for self-hosters who are trying to
bootstrap a service that doesn't yet have a domain or certificate.

The server should ensure the domain remains valid for at least 5 minutes
after a successful request, but no guarantees are required beyond that.



[0]: https://takingnames.io

[1]: https://oauth.net/2/

[2]: https://aaronparecki.com/2018/07/07/7/oauth-for-the-open-web

[3]: https://letsencrypt.org/
