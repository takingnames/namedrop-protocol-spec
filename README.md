# The NameDrop Protocol (draft version 0.3.0)

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

NameDrop scopes are prefixed with `namedrop-`, in order to facilitate
composition with other OAuth2 protocols on the same authorization server.

The following scopes are currently specified:

* `namedrop-hosts` - grants control over A, AAAA, CNAME, and ALIAS (aka ANAME) records
* `namedrop-mail` - grants control over MX, DKIM TXT, and SPF TXT records
* `namedrop-acme` - grants control over ACME TXT records
* `namedrop-atproto-handle` - grants control over atproto `did=` TXT records
  (see [here](https://atproto.com/specs/handle))

Permissions are granted to a FQDN (domain or subdomain). Currently this works
in a hierarchical fashion, ie if you have a token with permissions for
`example.com`, you can create records for any subdomain of `example.com`
(`*.example.com`). Likewise, if you have permissions for `sub.example.com`, you
can create records for `*.sub.example.com`, but not `example.com`.

# OAuth2 endpoints

The basic OAuth2 endpoints are defined as follows:

**`GET /authorize`**

Authorization endpoint (user consent to get code). Can be a web browser
redirect, or a direct link, such as one printed from a CLI application.

**`POST /token`**

Token endpoint (swap code for token). Always server-to-server.


# Token

Access tokens are returned as JSON in the following format:

```javascript
{
  "access_token": String(),
  "refresh_token": String(),
  "token_type": "bearer",
  "expires_in": Number(),
  "permissions": [
    {
      "scope": String(),
      "domain": String(),
      "host": String(),
    }
  ]
}
```

Example:

```json
{
  "access_token": "lkjaslkajsoidfnaiosnf",
  "refresh_token": "iousdoinfoiseofinsef",
  "token_type": "bearer",
  "expires_in": 3600,
  "permissions": [
    {
      "scope": "namedrop-hosts",
      "domain": "example.com",
      "host": ""
    },
    {
      "scope": "namedrop-mail",
      "domain": "example.com",
      "host": "mail"
    },
    {
      "scope": "namedrop-acme",
      "domain": "example.com",
      "host": ""
    }
  ]
}
```

# Getting and setting records

Records are managed via a simple RPC API. All requests use the POST method
with a JSON body. The `Content-Type` can be anything. This allows browser
clients to send
"[simple](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple_requests)"
requests that don't trigger CORS preflights, which are
an abomination. This is safe because all requests are authorized via the
included token property.

For `create-records`, `set-records`, and `delete-records`, the top-level
`domain` and `host` properties are used as defaults for any records where they
are missing. This information can also be inferred from the token, assuming it
doesn't have permissions for multiple domains/hosts. This can make client code
less verbose.

`type` is the record type such as `A`, `CNAME`, `MX`, etc. `ttl` and
`priority` are both integers.

When setting `value` for a record, the template variable `{{host}}` can be
used. It will be replaced with the actual host value when the server evaluates
the record. This is particularly useful for things like DKIM and ACME challenge
records.


## `POST /get-records`

Retrieves current records. This could be done via DNS, but it's convenient to
provide it as part of the NameDrop API to make things easier for clients.

The request is JSON in the following format:

```javascript
{
  "domain": String(),
  "host": String(),
  "token": String(),
}
```

Example:

```json
{
  "domain": "example.com",
  "host": "sub",
  "token": "lkjaslkajsoidfnaiosnf"
}
```

## `POST /create-records`

Create new records, returning an error if any duplicate records exist.

```javascript
{
  "domain": String(),
  "host": String(),
  "records": [
  {
    "domain": String(),
    "host": String(),
    "type": String(),
    "value": String(),
    "ttl": Number(),
    "priority": Number(),
  },
  // more records
}
```

Example:

```json
{
  "domain": "example.com",
  "host": "sub",
  "token": "lkjaslkajsoidfnaiosnf",
  "records": [
    {
      "type": "A",
      "value": "192.168.0.1"
    },
    {
      "host": "sub1._domainkey.{{host}}",
      "type": "CNAME",
      "value": "f1.example.com.dkim-server.com"
    }
  ]
}
```

## `POST /set-records`

Set records, overriding any existing duplicate records.

```javascript
{
  "domain": String(),
  "host": String(),
  "records": [
  {
    "domain": String(),
    "host": String(),
    "type": String(),
    "value": String(),
    "ttl": Number(),
    "priority": Number(),
  },
  // more records
}
```

Example:

```json
{
  "domain": "example.com",
  "host": "sub",
  "token": "lkjaslkajsoidfnaiosnf",
  "records": [
    {
      "type": "A",
      "value": "192.168.0.1"
    },
    {
      "host": "sub1._domainkey.{{host}}",
      "type": "CNAME",
      "value": "f1.example.com.dkim-server.com"
    }
  ]
}
```

## `POST /delete-records`

Delete records, silently ignoring any records that don't exist.

```javascript
{
  "domain": String(),
  "host": String(),
  "records": [
  {
    "domain": String(),
    "host": String(),
    "type": String(),
    "value": String(),
    "ttl": Number(),
    "priority": Number(),
  },
  // more records
}
```

Example:

```json
{
  "domain": "example.com",
  "host": "sub",
  "token": "lkjaslkajsoidfnaiosnf",
  "records": [
    {
      "type": "A",
      "value": "192.168.0.1"
    },
    {
      "host": "sub1._domainkey.{{host}}",
      "type": "CNAME",
      "value": "f1.example.com.dkim-server.com"
    }
  ]
}
```



# Other endpoints

## `GET /my-ip`

Returns the public IP of the client, as observed from the server. This is
useful for helping self-hosted clients test whether they can be reached by the
outside world.

The IP is returned as a simple string.


## `GET /ip-domain`

This causes the server to create a special `A` and/or `AAAA` record pointing at
the client's IP address, as observed by the server. The domain must start with
the IP address, but with '.' or ':' characters replaced with '-'. The rest of
the domain can be anything. The created domain is returned as a simple string.

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
