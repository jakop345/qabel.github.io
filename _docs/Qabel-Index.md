---
title: "Identity directory"
---
# Identity directory

## Abstract

Identities can be publicly shared in an identity directory provided by the
index server.

All data is sent as JSON (encoded in UTF-8). All data types are defined
[here](../Qabel-Client-Local-Data#data-types).

## Concept & structures

The identity directory can be queried using private data (*fields*),
e.g. an email address or phone number. The index server then
returns matching identites to the requester.

Changes to the directory are submitted as *update requests*. An update
request can associate or de-associate any number of fields with/from
an identity (the combination of a key-pair, a drop URL and an
alias). The association of a particular field with an identity is an
*entry* in the directory. Entries are not unique.

An identity is always represented using this JSON structure:

    {
        "public_key": KEY,
        "drop_url": URL,
        "alias": STR
    }

### Fields

Currently defined fields are:

email
: E-Mail address

phone
: Phone number (must be capable of receiving SMS). Incoming phone
numbers are normalized to ITU-T E.164 according to the specified
locale (via HTTP Accept-Language), if they are not already. It is
recommended to normalize phone numbers on the client.

: The server might reject phone numbers due to blacklisting of their
country code. This results in a HTTP 400 Bad Request status.

## APIs

An index server may require authorization from clients for API
calls. This is a configuration setting of the server.

When the server receives a request that requires authorization it
forwards the `Authorization` header to the
[authentication API](../Qabel-Accounting#authentication) to verify the
authorization token.

Requests with no or invalid authorization supplied result in a HTTP
403 response.

### Search

* Resource: /api/v0/search/
* Methods: GET, POST
* Request data (GET): field-value pairs (`field=value[&field=value...]`)
* Request data (POST): JSON `{"query": [{"field": STR, "value": STR}, ...]}`
* Response data: `{"identities": [identity, ...]}`

When multiple field-value pairs are specified all identities matching any criteria
are returned.

At least one field-value pair must be specified. A field can be specified multiple times
(even in the GET query string, assuming the HTTP client library supports that).

The returned `identity` structures have an additional key `matches` with a list of
field-value pairs that matched it (= `[{"field": STR, "value": STR}, ...]`):


```json
{
    "identities": [
        {
            "alias": "qabel_user",
            "public_key": "12...34",
            "drop_url": "https://example.net/...",
            "matches": [
                {
                    "field": "email",
                    "value": "foo@example.net"
                }
            ]
        }
    ]
}
```

### Update

Atomically create or delete entries (therefore also update entries).

* Resource: /api/v0/update/
* Method: PUT
* Content types:

    * application/json
    * application/vnd.qabel.noisebox+json

    If the content type is plain text it contains an *update
    request*. This is only valid for requests purely made of
    *deletes*.

    If the content is a noise box, then that noise box must be created
    by the key pair the update request refers to and it must be
    encrypted for the servers [ephemeral key](#key).

    (Note: A valid noise box can only be computed by calculating the
    Diffie-Hellman of the requesting key pair and the servers
    ephemeral key. Forging a noise box would be as hard as breaking
    the CDH problem.)

    Any update request containing a *create* cannot be executed
    immediately, since they require explicit confirmation by the
    user. The HTTP status code is thus 202 (Accepted) and not
    indicative of the request status.

* Request data: `{"identity": identity, "items": [update item, ...]}`

    Update item:

      {
          "action": STR ("create" or "delete"),
          "field": STR (field name),
          "value": STR (field value)
      }

* Response data: None
* Status code:

    * 202: accepted request, will be executed when user confirms it
    * 204: request executed
    * 400: malformed request
    * 401: cryptography failure, sender key does not match update request public_key

* When receiving a 400 while submitting an encrypted request, re-fetch
  the public key and retry: the server may have been restarted.

* Update items are not required. If no update items are given then the identity
  on the index server is updated with the drop URL and alias provided. This is only
  valid if the request was encrypted.

### Identity status

Retrieve the statuses of the entries associated with an identity.

* Resource: /api/v0/status/
* Method: POST
* Content type: application/vnd.qabel.noisebox+json
* Request data: `{"api": "status", "timestamp": INT}`

    The "timestamp" is an integer timestamp counting the seconds
    since the POSIX epoch (00:00:00 Coordinated Universal Time (UTC),
    Thursday, 1 January 1970; leap seconds don't count).

    If "timestamp" is older than two days the request is rejected (HTTP 400).

    If "api" is not "status" the request is rejected (HTTP 400).

* Response data:

    ```
    {
        "identity": identity,
        "entries": [entry, ...]
    }
    ```

    If no identity with the public key of the noisebox is known,
    no entries are returned and `identity` will be `null`.

    The `entry` structure is defined as follows:

    ```
    {
        "status": "confirmed"|"unconfirmed"|"deletion-pending",
        "field": STR (field name),
        "value": STR (field value)
    }
    ```

    The "confirmed" status means that this is publicly available data,
    while "unconfirmed" means that the server is awaiting confirmation
    from the user and the entry is not yet visible. "deletion-pending"
    is analogous, but for entries which are currently public and will
    be deleted when confirmation is received.

    The latter only happens in response to unencrypted delete requests.

### Delete identity

Delete all data associated with an identity.

* Resource: /api/v0/delete-identity/
* Method: POST
* Content type: application/vnd.qabel.noisebox+json
* Request data: `{"api": "delete-identity", "timestamp": INT}`

    The "timestamp" is an integer timestamp counting the seconds
    since the POSIX epoch (00:00:00 Coordinated Universal Time (UTC),
    Thursday, 1 January 1970; leap seconds don't count).

    If "timestamp" is older than two days the request is rejected (HTTP 400).

    If "api" is not "delete-identity" the request is rejected (HTTP 400).

* Response data: None (HTTP 204)

* If no identity with the public key of the noisebox is known,
  HTTP 404 is returned and no action is taken.

### Key

Return the ephemeral server public key.

Encrypt noise boxes for the update API with this key as the recipient.

* Resource: /api/v0/key/
* Method: GET
* Request data: None
* Response data: `{"public_key": STR}`

### Verification confirmation

Often this will be accessed by the user directly (so ignore any body
returned), but in some cases it's useful to access this
programmatically.

This is part of the verification performed by the index server to ensure that
published identities connected to emails or phone numbers actually belong to
the email/phone number owner.

For example, when an update request is received which publishes a phone number,
then a SMS is sent to that phone number. Client applications can then offer
an input dialog to make the process quicker for the user (by not having to navigate
to a web page and enter a code, but rather just enter it straight away in the client).

* Resource: /\<id>/\<action>/
* Method: GET
* Request data: None
* Response data: Ignore
* `<action>` is either `confirm` or `deny`
* `<id>` is a string sent to the user
* HTTP statuses:

    - 200: ok
    - 400: invalid or expired `<id>`
    - 404: `<id>` doesn't exist or was already acted on
