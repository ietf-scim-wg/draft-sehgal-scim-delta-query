---
title: "SCIM Delta Query"
abbrev: "SCIM Delta Query"
docname: draft-sehgal-scim-delta-query-latest
category: std

ipr: trust200902
area: IETF
workgroup: SCIM
keyword: [Internet-Draft, SCIM]

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]
submissionType: IETF

author:
   -
    name: Anjali Sehgal
    organization: Amazon Web Services
    email: anjalisg@amazon.com
   -
    name: Danny Zollner
    organization: Microsoft
    email: danny.zollner@microsoft.com
    role: editor

normative:
  RFC7643:
  RFC7644:
  RFC8792:

informative:


--- abstract

This document defines extensions to the SCIM 2.0 protocol that enable clients to poll service providers for changes that have occurred since a delta (or watermark) token was issued by the service provider.

--- middle

# Introduction

Adoption of SCIM 2.0 has trended strongly in favor of "push" model implementations where SCIM clients push data to SCIM service providers. Scenarios reliant on a client regularly retrieving, or "pulling", data regarding a large number of resources from a service provider have faced challenges at scale. One of the challenges facing SCIM client implementers when trying to retrieve data from service providers is that there are limited options to improve efficiency by only retrieving resources that have changed since they were last observed by the client. ETags and the SCIM meta.lastModified attribute are sometimes considered as options here, but both have limitations and have not seen widespread adoption for use in tracking changes in SCIM.

This document aims to alleviate the efficiency problems related to not being able to omit unchanged resources from query responses by introducing extensions to the SCIM 2.0 protocol functionality to query for responses that only contain SCIM resources that have changed since an earlier time. The concept of retrieving only new changes exists in numerous other APIs and databases, but does not exist in the core SCIM 2.0 specifications. By improving SCIM's functionality in this area, scenarios reliant on clients pulling large amounts of data on a regular basis can be made significantly more efficient, lowering resource consumption for service providers and clients to return and parse query responses, respectively.

## Notational Conventions

{::boilerplate bcp14-tagged}

This document utilizes line folding within JSON examples using a single backslash ('\') character as outlined in [RFC8792].


# Definitions

Delta Token
: An opaque value issued by the SCIM service provider that provides a point of reference for the service provider to later return representations of resources that were changed after the point that the token's value references.

Change / Changed Resource
: A SCIM resource that has been created, deleted, or updated.

# Overview

The delta query functionality defined in this document is intended to function as follows:

* The SCIM client obtains a delta token from a SCIM service provider.
* The SCIM client makes a query against the SCIM service provider that contains the delta token.
* The SCIM service provider responds with all resources that match the query parameters and have changed since the provided delta token was issued.
* The SCIM service provider includes a new delta token with the final page of results it returns.
* The SCIM client then optionally uses the newly provided delta token to repeat this process at a future time.

# Delta Query Components

This document defines extensions to existing schemas defined in the SCIM 2.0 specifications [RFC7643] and [RFC7644] and introduces endpoints, query parameters and protocol elements.

## New Path Extension Endpoints

This document defines a set of extensions to the list of SCIM endpoints in {{Section 3.2 of RFC7644}}. The following endpoints are added to the list:

|Resource|Endpoint|Operations|Description|
Delta Token|\[prefix\]/.deltaToken|GET|Acquire a delta token.|
Delta Query|\[prefix\]/.delta|POST|Search from system root or within a resource endpoint for resources that have changed since the acquisition of the delta token.|

## Delta Tokens

Delta tokens may be returned from HTTP requests to the /.deltaToken path extension preceded by a SCIM resource endpoint (e.g.: /Users) or the server root. Delta tokens can also be returned as part of the service provider's response to a query redeeming another delta token.

Delta tokens provided in response to /.deltaToken requests against the server root MUST be valid for delta query requests made against all resource types on the service provider that support delta query. Delta tokens provided in response to a /.deltaToken request against a specific SCIM resource endpoint are only required to be valid for requests made against that specific resource endpoint.

Response to /.deltaToken requests MUST be identified using the following URI: "urn:ietf:params:scim:api:messages:2.0:delta:token". The following single-valued attributes are defined for responses:

value
: A string value containing a delta token.  REQUIRED.

expiry
: A dateTime value representing the time that the delta token value will be valid until. Service providers SHOULD attempt to provide an accurate expiry value. Service providers MAY have implementation-specific logic that invalidates delta tokens prior to the provided expiry time.  RECOMMENDED.

### Acquiring an initial delta token

A client wishing to obtain an initial delta token from a service provider that supports delta query can make a GET request to the /.deltaToken endpoint, as shown in the below example:

~~~
GET Users/.deltaToken
Host: example.com
Accept: application/scim+json
Authorization: Bearer foo
~~~

The service provider's response would return a token similar to the following example:

~~~
HTTP/1.1 200 OK
Content-Type: application/scim+json

{
    "schemas":"urn:ietf:params:scim:api:messages:2.0:delta:token",
    "value":"eyJkZWx0YVRva2VuIjoiQVJkZmF",
    "expiry":"2019-06-25T06:00:00Z"
}
~~~

## ListResponse Schema Extension

This document adds the following single-valued attribute to the "urn:ietf:params:scim:api:messages:2.0:ListResponse" schema defined in {{Section 3.4.2 of RFC7644}}.

nextDeltaToken
: A complex type representing the next delta token issued by the service provider. The sub-attributes of nextDeltaToken are "value" and "expiry" and carry the same descriptions and requirements as the matching attributes defined in the urn:ietf:params:scim:api:messages:2.0:delta:token schema in the Delta Tokens section of this document.

The nextDeltaToken attribute is included in the response to a query made with another delta token. The value of a delta token returned in this attribute MUST reference the point at which the final page of changed resources was returned by the service provider. This attribute MUST NOT be returned prior to the final page of changed resources and MUST be returned on the final page.

## ServiceProviderConfig

This document adds the following complex attribute to the ServiceProviderConfig resource defined in {{Section 4 of RFC7644}}.

DeltaQuery
: A complex attribute that indicates advertised delta query configuration options.  REQUIRED.

    supported
    : A Boolean type indicating if the service provider supports delta query.  REQUIRED.

    deltaTokenExpiry
    : A positive integer specifying the maximum number of seconds that a delta token is anticipated to be accepted by the service provider after being issued. Service providers that do not provide values for the "expiry" delta token attribute MUST advertise an estimated delta token lifetime in this attribute. Service providers that do not have a globally consistent lifetime for issued delta tokens SHOULD use the "expiry" value on each delta token instead.  OPTIONAL.

    supportedResources
    : A multi-valued string type indicating what resources on the service provider support returning changes via delta query. Values MUST be either 'ServerRoot' or names of resource types that exist on the service provider, e.g., 'User' or 'Group'.  OPTIONAL.

# Delta Query Protocol

## Delta Query Requests
A client can use a previously obtained delta token to request information about changes that have occurred to the service provider's identity data since the delta token was issued. Clients attempting to make a delta query request MUST use the HTTP POST verb combined with the "/.delta" path extension. service providers MAY implement support for delta query requests against only certain resource types. Service providers MUST allow delta query requests to be made against resource endpoints (e.g.: /Users) that support delta query, and MAY allow requests to be made against the root of the server. The "/.delta" path extension MAY be appended to the end of a valid SCIM resource URL or the SCIM server root. For example:

/Users/.delta

\<baseUrl\>/.delta

POST requests to /.delta MUST be identified using the URI "urn:ietf:params:scim:api:messages:2.0:delta:request". The schema of this query format includes the attributes defined in {{Section 3.4.3 of RFC7644}} for the urn:ietf:params:scim:api:messages:2.0:SearchRequest schema, any additional attributes added to the SearchRequest schema by other future SCIM 2.0 specifications, and the following additional single-valued attribute:

deltaToken
: The string value of a delta token previously issued by the service provider. The delta token value provided by the client MUST remain the same while paging through a response that extends across more than one page of results.  REQUIRED.

Service providers responding to a delta query request MUST return all resources that meet the following criteria:

* Changed since the provided delta token was issued
* Match all applied query filters
* The client is authorized to read the resource

Services providers SHALL evaluate all query parameters specified in a delta query request against the attribute values of the underlying SCIM resource and not the attributes or values of any delta query API message schemas.

## Delta Query Responses

This document defines a new "delta query response" SCIM message format that is returned by the service provider in the "Resources" collection in a ListResponse message. The delta query response message schema is a wrapper that contains full or partial representations of changed SCIM resources. Delta query responses consist of information about the change as well as a representation of the changed resource in either of two formats. The "operations" attribute contains one or more complex objects that represent changes to specific attributes on the resource, whereas the "data" attribute can be used to provide a traditional SCIM representation of the changed resource without specifying what attributes have changed. Delta query responses MUST be identified using the following URI: "urn:ietf:params:scim:api:messages:2.0:delta:response".

The following single-valued attributes are defined in this schema:

resourceType
: The resource type of the changed resource represented by the delta.  REQUIRED.

changeType
: A string representing what type of change has occurred.  Allowed values are "create", "update", and "delete".  REQUIRED.

changedResourceId
: A string representing the 'id' value of the changed resource.  REQUIRED.

data
: A complex object containing the changed resource.  REQUIRED if "operations" is null. MUST NOT be returned if "operations" is not null.


The following multi-valued attribute is defined in this schema:

operations
: A complex object representing the attribute-level changes that have occurred on the changed resource. REQUIRED if "data" is null". MUST NOT be returned if "data" is not null. This attribute has the following sub-attributes:

    op
    : A string that states what type of update has occurred. The possible values are aligned to the SCIM PATCH semantics defined in {{Section 3.5.2 of RFC7644}}. Allowed values are "add", "replace", and "remove".  REQUIRED.

    path
    : A string following the SCIM attribute notation and attribute path rules, representing the attribute that was updated. Supports attribute path filters as defined in {{Section 3.5.2 of RFC7644}}.  OPTIONAL.

    value
    : The new value(s) that the attribute or attribute value(s) specified in the path sub-attribute have been updated to contain.  REQUIRED when the value of "op" is "add" or "replace". MUST NOT be returned if the value of "op" is "remove".

Service providers SHOULD NOT return multiple delta query responses for the same changed resource in the same page of results. When returning attribute changes via the "operations" attribute, service providers MAY distribute attribute changes for a single changed resource between multiple delta query responses across multiple pages. Splitting attribute changes across multiple pages allows for resources with multi-valued attributes such as the group resource's "members" attribute to return all changes while maintaining smaller and more consistent page sizes.

### Change Types

Delta query responses are categorized into one of three change types.

#### Create

Newly created resources MUST be represented by a delta query response with a "changeType" of "Create". Delta query responses of type "Create" MUST return either the the full current state of the resource or the state of the resource at the time of creation in the "data" attribute.

Newly created resources that were created with or updated after creation to have a large amount of data MAY return a "Create" operation followed by one or more "Update" operations to provide an accurate representation of the current state of the resource split across multiple delta query response messages.

#### Update

Updated resources MUST be represented by a delta query response with a "changeType" of "Update". Service providers MAY return either the full current state of the updated resource via the "data" attribute, or only the changed attributes via the "operations" attribute. Service providers SHOULD return only changed attributes when possible.

#### Delete

Deleted resources MUST be represented by a delta query response with a "changeType" of "Delete". Deleted resources SHALL NOT have a value in either "data" or "operations".

### Operations

The structure of the "operations" attribute in the delta query response message is based on the SCIM PATCH semantics defined in {{Section 3.5.2 of RFC7644}}. The rules defined in this section and its subsections apply to the "operations" attribute's "op", "path", and "value" sub-attributes.

#### Add

Using the "add" op can represent new values being added to any attribute, and values being either added or replaced on single-valued attributes. Examples include:

##### Adding or replacing a value to a single-valued attribute

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:response"],
    "changedResourceId":"U49z",
    "resourceType":"User",
    "changeType":"Update",
    "operations":[
        {
            "op":"add",
            "path":"urn:ietf:params:scim:schemas:extension:enterprise\
:2.0:User:employeeNumber",
            "value": "123456"
        }
    ]
}
~~~

##### Adding or replacing multiple attribute values

~~~
{
    "op":"add",
    "value": {
        "displayName": "John Doe",
        "active": true,
        "urn:ietf:params:scim:schemas:extension:enterprise\
:2.0:User:employeeNumber": "12345"
    }
}
~~~

##### Adding a value to a multi-valued attribute

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:response"],
    "changedResourceId":"U49z",
    "resourceType":"User",
    "changeType":"Update",
    "operations":[
        {
            "op":"add",
            "path":"phoneNumbers",
            "value":[
                {
                    "value":"555-555-5555",
                    "type":"work"
                }
            ]

        }
    ]
}
~~~

##### Adding members being added to a group

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:response"],
    "changedResourceId":"G88",
    "resourceType":"Group",
    "changeType":"Update",
    "operations":[
        {
            "op":"add",
            "path":"members",
            "value":[
                {
                    "value":"memberId7"
                },
                {
                    "value":"memberId8"
                },
                {
                    "value":"memberId9"
                }
            ]
        }
    ]
}
~~~


#### Remove

The "remove" op is used when one or more values of an attribute have been removed. Examples include:

##### Removing a single-valued attribute

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:response"],
    "changedResourceId":"U40iA1Q9",
    "resourceType":"User",
    "changeType":"Update",
    "operations":[
        {
            "op":"remove",
            "path":"manager"
        }
    ]
}
~~~

##### Removing a value from a typed multi-valued attribute
~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:response"],
    "changedResourceId":"U40iA1Q9",
    "resourceType":"User",
    "changeType":"Update",
    "operations":[
        {
            "op":"remove",
            "path":"emails[type eq \"work\" and \
value eq \"user5@example.com\"]"
        }
    ]
}
~~~
##### Removing members from a group

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:response"],
    "changedResourceId":"G88",
    "resourceType":"Group",
    "changeType":"Update",
    "operations":[
        {
            "op":"remove",
            "path":"members[value eq \"member1\"]",
        },
        {
            "op":"remove",
            "path":"members[value eq \"member2\"]",
        }
    ]
}
~~~


#### Replace

The "replace" op can be used when some or all of the values of an attribute have been replaced. When responding with operations about multi-valued attributes, service providers SHOULD provide unambiguous valuePath filters when possible, and SHALL provide all current values of the attribute when constructing an unambiguous valuePath filter is not possible. Examples include:

##### Replacing a single-valued attribute

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:response"],
    "changedResourceId":"U40iA1Q9",
    "resourceType":"User",
    "changeType":"Update",
    "operations":[
        {
            "op":"replace",
            "path":"userName",
            "value":"jensenb@example.com"
        }
    ]
}
~~~

##### Replacing values in a typed multi-valued attribute

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:response"],
    "changedResourceId":"U40iA1Q9",
    "resourceType":"User",
    "changeType":"Update",
    "operations":[
        {
            "op":"replace",
            "path":"phoneNumbers[type eq \"work\" \
and primary eq true].value",
            "value":"555-555-5556"
        }
    ]
}
~~~

## Request and Response Examples

### Requesting changed resources

The following example shows a client using a previously obtained delta token to make a delta query request for all changes for the User resource type.

~~~
POST /Users/.delta
Host: example.com
Accept: application/scim+json
Authorization: Bearer foo

{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:request"],
    "deltaToken":"dGVzdC1kZWx0YVRva2Vu"
}
~~~

The service provider response would then contain any User resources that had changed since the delta token "dGVzdC1kZWx0YVRva2Vu" was issued. The following example shows a response with a newly created user, a user with attribute updates, and a deleted user.

~~~
HTTP/1.1 200/OK
Content-Type: application/scim+json

{
    "totalResults": 3,
    "itemsPerPage": 3,
    "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
    "Resources": [
        {
            "schemas":"urn:ietf:scim:api:messages:2.0:delta:response",
            "changedResourceId": "123",
            "resourceType": "User",
            "changeType": "Create",
            "data": {
                "userName": "bjensen",
                "name": {
                    "formatted": "Ms. Barbara J Jensen III",
                    "familyName": "Jensen",
                    "givenName": "Barbara"
                },
                "id": "123",
                "active": true,
                "phoneNumbers": [
                    {
                        "value": "555-555-5555",
                        "type": "work"
                    }
                ]
            }
        },
        {
            "schemas":"urn:ietf:scim:api:messages:2.0:delta:response",
            "changedResourceId": "456",
            "resourceType": "User",
            "changeType": "Update",
            "operations": [
                {
                    "op": "replace",
                    "path": "name.givenName",
                    "value": "Jim"
                },
                {
                    "op": "add",
                    "path": "phoneNumbers",
                    "value": [
                        {
                            "value": "555-555-4567",
                            "type": "mobile"
                        }
                    ]
                }
            ]

        },
        {
            "schemas":"urn:ietf:scim:api:messages:2.0:delta:response",
            "changedResourceId": "789",
            "resourceType": "User",
            "changeType": "Delete"
        }
    ]
}
~~~

### Updates from service providers that do not support attribute-level change tracking

Service providers that do not support attribute-level change tracking MAY return the full current state of changed resources in the "data" attribute. The following example shows a delta query response by a service provider that only returns the full current state of changed resources. Clients SHALL be responsible for determining what attribute values have changed when the service provider has returned updated resource information using the "data" attribute.

~~~
POST /Users/.delta
Host: example.com
Accept: application/scim+json
Authorization: Bearer foo

{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:request"],
    "deltaToken":"3894e6d5-8e9e-4b5c-9b3b-6e8e7d4a4e9d",
    "filter":"title eq \"Tour Guide\" and \
addresses.country eq \"France\""
}
~~~

The service provider's response (some results omitted for brevity) is:

~~~
HTTP/1.1 200 OK
Content-Type: application/scim+json

{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
    "totalResults":30,
    "itemsPerPage":10,
    "Resources":[
        {
            "schemas":["urn:ietf:params:scim:api:messages:\
2.0:delta:response"],
            "changedResourceId":"qwerty",
            "resourceType":"User",
            "changeType":"Update",
            "data":{
                "userName":"bjensen",
                "name":{
                    "formatted":"Ms. Barbara J Jensen III",
                    "familyName":"Jensen",
                    "givenName":"Barbara"
                },
                "title":"Tour Guide",
                "id":"qwerty",
                "active":true,
                "emails":[
                    {
                        "value":"bjensen@example.com",
                        "type":"work",
                        "primary":true
                    }
                ],
                "addresses":[
                    {
                        "type":"work",
                        "country":"France",
                        "locality":"Paris",
                        "primary":true
                    }
                ]

            }
        },
        {...},
        {...}
    ]
}

~~~


### Creation of a large resource across multiple pages

As described in (Ref to DQ Response - Create), large resources such as groups with many members may be returned across multiple pages. The following example shows a service provider's delta query responses for a group broken into multiple delta query response messages.

~~~
POST /Groups/.delta
Host: example.com
Accept: application/scim+json
Authorization: Bearer foo

{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:delta:request"],
    "deltaToken":"3894e6d5-8e9e-4b5c-9b3b-6e8e7d4a4e9d",
}
~~~

The service provider's response may contain delta query responses pertaining to other resources. For the newly created example group with an "id" value of "G123", the following delta query responses are returned:

Some "members" values are omitted in the below examples for brevity.

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
    "totalResults":405,
    "itemsPerPage":1,
    "nextCursor":"foo",
    "Resources":[
        {
            "schemas":["urn:ietf:params:scim:api:messages:\
    2.0:delta:response"],
            "changedResourceId":"G123",
            "resourceType":"Group",
            "changeType":"Create",
            "data":{
                "id":"G1",
                "displayName":"All Users",
                "members":[
                    {
                        "value":"member1"
                    },
                        ...
                    {
                        "value":"member99"
                    }
                ]

            }
        }
    ]
}
~~~

Following the "Create" delta query response, the service provider on a later page of results returns:

~~~
{
    "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
    "totalResults":456,
    "itemsPerPage":1,
    "nextCursor":"bar",
    "Resources":[
        {
            "schemas":["urn:ietf:params:scim:api:messages:\
    2.0:delta:response"],
            "changedResourceId":"G123",
            "resourceType":"Group",
            "changeType":"Update",
            "operations":[
                {
                    "op":"add",
                    "path":"members",
                    "value":[
                        {
                            "value":"member100"
                        },
                        {
                            "value":"..."
                        },
                        {
                            "value":"member199"
                        }
                    ]
                }
            ]
        }
    ]
},
~~~

# Security Considerations
To-do

# IANA Considerations
To-do

# Acknowledgements
To-do

# Other
To-do
