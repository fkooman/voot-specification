# Introduction
VOOT 3.0 is a simple protocol for cross-domain read-only access to information
about persons and their group membership within an organization. It can be seen 
as making LDAP-like information available as a web service.

The API is loosly based on the OpenSocial specification, but this is just 
for historical reasons and not all requirements of OpenSocial are met. Only the 
JSON data format is supported for example.

# Use Cases
All the use cases that are valid for LDAP are also valid for VOOT. For instance,
requesting information about users, their group memberships and the members of
a group. VOOT can however not be used to authenticate users as there is no 
password information available to VOOT.

Web based applications can sometimes interface with LDAP to work with group
memberships to base their authorization on this for instance. It is 
notoriously hard or even impossible to get this working cross-domain in a 
secure fashion. This is where VOOT steps in.

# Authorization
This specification will consider two authorization models:

* Basic Authentication [RFC 2617] if the VOOT provider trusts the client not to
  abuse full access to the user database;
* OAuth 2.0 [RFC 6749] if there is minimal trust between the VOOT provider and 
  the client where it is left to the user to authorize the client explicitly
  that wants to access information about just the user that grants the 
  permission.

These modes can for example be combined. A proxy using the VOOT protocol can 
use OAuth 2.0 for authorization towards clients while the proxy requests 
other VOOT endpoints protected by Basic Authentication. This can for example be
helpful in "hub and spoke" identity federations.

< PICTURE OF PROXY >

# API
The API supports three calls.

The API calls make use of `@me` which is a placeholder for the user that 
authorized the client. This of course only works when the application uses OAuth 
2.0 as the access token used by the client is bound to the user that authorized 
it.

For the Basic Authentication case an actual user identifier and group 
identifier need to be specified. It is out of scope how the client obtains 
these identifiers.

If the user `admin` authorized a client to act on its behalf, the following
calls have identical results:

    /groups/@me
    /groups/admin

    /people/@me/members
    /people/admin/members

    /people/@me
    /people/admin

The calls using `@me` MUST be supported, the calls using the user identifier
MUST be supported when using Basic Authentication and MAY be supported when
using OAuth 2.0.

## Retrieve Group Membership
This call retrieves a list of all groups the user is a member of.

    /groups/@me

This call MUST be supported. The result can include the following keys with
information about the groups, where only `id` MUST be present:

* `id`
* `title`
* `description`
* `voot_membership_role`

The `id` field contains a local (to the provider) unique identifier of the 
group. It SHOULD be opague to the client. The `title` field contains the 
short human readable name of the group. The `description` field can contain a 
longer description of the group with possibly its purpose. The field 
`voot_membership_role` indicates the role of the user in that
particular group. It can be any of these values: `admin`, `manager` or `member`. 

## Retrieve Members of a Group
This call retrieves a list of all members of a group the user is a member of.

    /people/@me/{groupId}

Where `{groupId}` is replaced with a group identifier obtained through the
call used to retrieve Group Membership.

This call MAY be supported. The result can include the following keys with
information about the user, where only `id` MUST be present:

* `id`
* `displayName`

The `id` field contains a local (to this provider) unique identifier of the 
user. It SHOULD be opague to the client. The `displayName` field contains the
name by which the user prefers to be addressed and can possibly be set by the
user themselves at the provider. The `displayName` field is OPTIONAL.

## Retrieve User Information
This call retrieves additional information about a user.

    /people/@me

This call MUST be supported. The result can include the following keys with
information about the user, where only `id` MUST be present:

* `id`
* `displayName`
* `commonName`
* `emails`

The `id` field contains a local (to this provider) unique identifier of the 
user. It SHOULD be opague to the client. The `displayName` field contains the
name by which the user prefers to be addressed and can possibly be set by the
user themselves at the provider. The `displayName` field is OPTIONAL. The
`commonName` field contains the official full name of the user. This field 
cannot be modified by the user themselves. The `emails` field contains a list
of email addresses belonging to the user. 

## Request Parameters
The API calls have four OPTIONAL parameters that manipulate the result obtained 
from the provider:

* `sortBy`
* `sortOrder`
* `startIndex`
* `count`

The `sortBy` parameter determines the key in the result that is used for sorting
the groups or group members. The available keys are listed below in the API 
Response section. The `sortOrder` determines the order in which the results are 
sorted. Here, two values are possible: `ascending` and `descending`. These 
parameters are OPTIONAL. It is up to the provider whether or not to sort and 
in what order if these parameters are not present. If the results are to be 
sorted, the value SHOULD be compared as strings and SHOULD be sorted 
case-insensitive.

The `startIndex` parameter determines the offset at which the start for giving
back results. The `count` parameter indicates the number of results to be
given back. The `startIndex` and `count` parameters can be used to implement
paging by returning only a subset of the results. These parameters are OPTIONAL,
if they are not provided the provider MUST consider `startIndex` equals to `0`
and `count` equal to the total number of items available in the set.

The sorting, if requested, MUST be performed on the provider before considering 
the `startIndex` and `count` parameters.

For the API call requesting user information the `sortBy` and `sortOrder` 
parameters have no effect. Using `startIndex` and `count` is possible, 
however they are of little use as there always will be only one answer.

## Response Parameters
All responses mentioned above have the same format. There are always four keys:

* `startIndex`
* `itemsPerPage`
* `totalResults`
* `entry`

Where `startIndex` contains the offset from which the results are returned, 
this is usually equals to the requested `startIndex` unless this value was not
set or invalid, possibly out of bounds. The `itemsPerPage` contains the actual
number of results in the set, as part of `entry`, returned. The `totalResults`
field contains the full number of elements available, not depending on the
`startIndex` and `count` parameters.

The `entry` key contains a list of items, either groups, people or person 
information. Below are some examples.

## API Examples
Below are some API examples for retrieve group membership, a list of group
members and information about the user.

### Retrieve Group Membership
This is an example of the response to the query:

    Host: provider.example.org
    GET /groups/@me?sortBy=title&sortOrder=ascending HTTP/1.1
    
The response looks like this:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "entry": [
            {
                "description": "Group containing employees.", 
                "id": "employees", 
                "title": "Employees", 
                "voot_membership_role": "admin"
            }, 
            {
                "description": "Group containing everyone at this institute.", 
                "id": "members", 
                "title": "Members", 
                "voot_membership_role": "member"
            }
        ], 
        "itemsPerPage": 2, 
        "startIndex": "0", 
        "totalResults": 2
    }

### Retrieve Members of a Group
This is an example of the response to the query:

    Host: provider.example.org
    GET /people/@me/members?sortBy=displayName&sortOrder=descending&startIndex=3&count=2 HTTP/1.1
    
The response looks like this:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "entry": [
            {
                "id": "mwisdom", 
                "voot_membership_role": "member"
            }, 
            {
                "id": "bmcatee", 
                "voot_membership_role": "member"
            }
        ], 
        "itemsPerPage": 2, 
        "startIndex": "3", 
        "totalResults": "7"
    }

### Retrieve User Information
This is an example of the response to the query:

    Host: provider.example.org
    GET /people/@me HTTP/1.1

The response looks liks this:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "entry": {
            "commonName": "Mr. Admin I. Strator", 
            "displayName": "admin", 
            "emails": [
                "admin@example.org", 
                "postmaster@example.org"
            ], 
            "id": "admin"
        }, 
        "itemsPerPage": 1, 
        "startIndex": 0, 
        "totalResults": 1
    }

# Error Handling
Handling failures of Authentication, either Basic or Bearer are handled in the 
ways described in RFC xxxx and RFC yyyy. This will involve sending the 
`WWW-Authenticate` header if something is wrong, for example an invalid 
OAuth 2.0 access token will result in the following response:

    HTTP/1.1 401 Unauthorized
    WWW-Authenticate: Bearer realm="Resource Server",error="invalid_token",error_description="the access token is not valid"
    Content-Type: application/json

    {"error":"invalid_token","error_description":"the access token is not valid"}

There are also some request errors defined, i.e.: invalid requests to the 
provider that should be dealt with in a certain manner. Only the call that 
retrieves group membership and user information MUST be supported, the other 
call, i.e.: retrieving members of a group does not need to be supported. 
When this call is disabled a response code of `400 Bad Request` is returned 
with `error` set to `unsupported_request`.

For all requests that contain query parameters for limiting and sorting results 
holds that if any of those values are invalid they are set to their defaults:

* `startIndex`: if this value is negative, not a number or >= `totalResults`-1 
  it is set to `0`;
* `count`: if this value is negative, or not a number it is set to the value of
  `totalResults`;
* `sortBy`: if the specified key is invalid, i.e.: not part of the entries, 
  sorting is disabled;
* `sortOrder`: if this value is neither `ascending` nor `descending` it is 
  set to `ascending`, if `sortBy` is specified and valid.

The error response is returned as JSON, for example:

    HTTP/1.1 404 Not Found
    Content-Type: application/json

    {
        "error": "invalid_user", 
        "error_description": "the user does not exist"
    }

The `error` field MUST be present, the `error_description` field is OPTIONAL.
 
## Retrieve Group Membership
The call looks like this:

    /groups/@me

* If Basic Authentication is used and `@me` is used an error response with 
  code `400 Bad Request` is returned. The `error` field contains 
  `unsupported_user_identifier`. If a user identifier is specified instead of 
  `@me` for providers not supporting the use of actual user identifiers the
  same error is returned;
* If the specified user does not exist at the provider an error response with
  code `404 Not Found` is returned. The `error` field contains 
  `invalid_user`;
* If any other error occurs an error response with code 
  `500 Internal Server Error` is returned. The `error` field MAY contain
  more information about the error.

## Retrieve Members of a Group
The call looks like this:

    /people/@me/members

* If Basic Authentication is used and `@me` is used an error response with 
  code `400 Bad Request` is returned. The `error` field contains 
  `unsupported_user_identifier`. If a user identifier is specified instead of 
  `@me` for providers not supporting the use of actual user identifiers the
  same error is returned;
* If the specified user does not exist at the provider an error response with
  code `404 Not Found` is returned. The `error` field contains 
  `invalid_user`;
* If the specified user is not a member of the group an error response with 
  code `403 Forbidden` is returned. The `error` field contains `not_a_member`.
  This response MUST be returned no matter whether the group exists or not;
* If the specified group does not exist at the provider an error response with
  code `404 Not Found` is returned. The `error` field contains 
  `invalid_group`;
* If any other error occurs an error response with code 
  `500 Internal Server Error` is returned. The `error` field MAY contain
  more information about the error.

### Retrieve User Information
The call looks like this:

    /people/@me

* If Basic Authentication is used and `@me` is used an error response with 
  code `400 Bad Request` is returned. The `error` field contains 
  `unsupported_user_identifier`. If a user identifier is specified instead of 
  `@me` for providers not supporting the use of actual user identifiers the
  same error is returned;
* If the specified user does not exist at the provider an error response with
  code `404 Not Found` is returned. The `error` field contains 
  `invalid_user`;

# Proxy Operation
One of the use cases is to make it possible to combine data from various 
group providers using one API endpoint. This way group membership information
can be aggregated from various sources. The proxy provides a OAuth 2.0 
protected API to clients and in the backend uses Basic Authentication to talk
to the group providers from which it needs to aggregate data.

From the client point of view there should be no difference in the API compared 
to talking directly to a group provider. There are however some special error
cases that should be considered. For instance if the remote group provider is
not available. Also the group identifiers that were scoped locally per group 
provider need to be modified to include a "scope", i.e. to indicate to what
group provider they belong.

For example the user `john`, which is a local identifier at a group provider 
can occur in multiple group providers, so it needs to be prefixed, for example
with the identifier of the group provider. The prefixed value SHOULD be 
opague as well.

# Privacy
Opague value, @me, restricting people calls etc etc
