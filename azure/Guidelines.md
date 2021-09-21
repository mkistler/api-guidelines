# Microsoft Azure REST API Guidelines
## History

| Date        | Notes                                                          |
| ----------- | -------------------------------------------------------------- |
| 2021-Sep-11 | Add long-running operations guidance                           |
| 2021-Aug-06 | Updated Azure REST Guidelines per Azure API Stewardship Board. |
| 2020-Jul-31 | Added service advice for initial versions                      |
| 2020-Mar-31 | 1st public release of the Azure REST API Guidelines            |

## Introduction

These guidelines offer prescriptive guidance that Azure service teams MUST follow ensuring that customers have a great experience by designing APIs meeting these goals:
- Developer friendly via consistent patterns & web standards (HTTP, REST, JSON)
- Efficient & cost-effective
- Work well with SDKs in many programming languages
- Customers can create fault-tolerant apps by supporting retries/idempotency/optimistic concurrency
- Sustainable & versionable via clear API contracts with 2 requirements:
  1. Customer workloads must never break due to a service change
  2. Customers can adopt a version without requiring code changes

Technology and software is constantly changing and evolving, and as such, this is intended to be a living document. [Open an issue](https://github.com/microsoft/api-guidelines/issues/new/choose) to suggest a change or propose a new idea.

See the [Considerations for Service Design](./ConsiderationsForServiceDesign.md) for an introduction to the topic of API design for Azure services.

*NOTE: For an existing GA'd service, don't change/break its existing API; instead, leverage these concepts for future APIs while prioritizing consistency within your existing service.*

### Prescriptive Guidance
This document offers prescriptive guidance labeled as follows:

:white_check_mark: **DO** adopt this pattern. If you feel you need an exception, contact the Azure HTTP/REST Stewardship Board **prior** to implementation.

:ballot_box_with_check: **YOU SHOULD** adopt this pattern. If not following this advice, you MUST disclose your reason during the Azure HTTP/REST Stewardship Board review.

:heavy_check_mark: **YOU MAY** consider this pattern if appropriate to your situation. No notification to the Azure HTTP/REST Stewardship Board is required.

:warning: **YOU SHOULD NOT** adopt this pattern. If not following this advice, you MUST disclose your reason during the Azure HTTP/REST Stewardship Board review.

:no_entry: **DO NOT** adopt this pattern. If you feel you need an exception, contact the Azure HTTP/REST Stewardship Board **prior** to implementation.

*If you feel you need an exception, or need clarity based on your situation, please contact the Azure HTTP/REST Stewardship Board **prior** to release of your API.*

## Building Blocks: HTTP, REST, & JSON
The Microsoft Azure Cloud platform exposes its APIs through the core building blocks of the Internet; namely HTTP, REST, and JSON. This section provides you with a general understanding of how these technologies should be applied when creating your service.

### HTTP
Azure services must adhere to the HTTP specification, [RFC7231](https://tools.ietf.org/html/rfc7231). This section further refines and constrains how service implementors should apply the constructs defined in the HTTP specification. It is therefore, important that you have a firm understanding of the following concepts:
- [Uniform Resource Locators (URLs)](URLS)
- HTTP Methods
- Request & Response Headers
- Bodies

### Uniform Resource Locators (URLs)

A Uniform Resource Locator (URL) is how developers access the resources of your service. Ultimately, URLs are how developers form a cognitive model of your service's resources.

:white_check_mark: **DO** use this URL pattern:
```text
https://<service>.<cloud>/<tenant>/<service-root>/<resource-collection>/<resource-id>/
```

Where:
 | Field | Description
 | - | - |
 | service | Name of the service (ex: blobstore, servicebus, directory, or management)
 | cloud | Cloud domain name, e.g. `azure.net` (see Azure CLI's "az cloud list")
 | tenant | Globally-unique ID of container representing tenant isolation, billing, enforced quotas, lifetime of containees (ex: subscription UUID)
 | service&#x2011;root | Service-specific path (ex: blobcontainer, myqueue)
 | resource&#x2011;collection | Name of the collection, unabbreviated, pluralized
 | resource&#x2011;id | Value of the unique id property. This MUST be the raw string/number/guid value with no quoting but properly escaped to fit in a URL segment.

:white_check_mark: **DO** use kebab-casing (preferred) or camel-casing for URL path segments. If the segment refers to a JSON field, use camel casing.

:white_check_mark: **DO** return `414-URI Too Long` if a URL exceeds 2083 characters

:white_check_mark: **DO** treat service-defined URL path segments as case-sensitive. If the passed-in case doesn't match what the service expects, the request **MUST** fail with a `404-Not found` HTTP return code.

Some customer-provided path segment values may be compared case-insensitivity if the abstraction they represent is normally compared with case-insensitivity. For example, a UUID path segment of 'c55f6b35-05f6-42da-8321-2af5099bd2a2' should be treated identical to 'C55F6B35-05F6-42DA-8321-2AF5099BD2A2'

:white_check_mark: **DO** ensure proper casing when returning a URL in an HTTP response header value or inside a JSON response body

:white_check_mark: **DO** restrict the characters in service-defined path segments to `0-9  A-Z  a-z  -  .  _  ~`, with `:` allowed only as described below to designate an action operation.

:ballot_box_with_check: **YOU SHOULD** restrict the characters allowed in user-specified path segements (i.e. path parameters values) to `0-9  A-Z  a-z  -  .  _  ~` (do not allow `:`).

:ballot_box_with_check: **YOU SHOULD** keep URLs readable; if possible, avoid UUIDs & %-encoding (ex: Cádiz is %-encoded as C%C3%A1diz)

:heavy_check_mark: **YOU MAY** use these other characters in the URL path but they will likely require %-encoding [[RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986#section-2.1)]: `/  ?  #  [  ]  @  !  $  &  '  (  )  *  +  ,  ;  =`

#### Direct Endpoint URLs

:heavy_check_mark: **YOU MAY** support a direct endpoint URL for performance/routing:
```text
https://<tenant>-<service-root>.<service>.<cloud>/...
```

Examples:
- Request URL: `https://blobstore.azure.net/contoso.com/account1/container1/blob2`
- Response header ([RFC2557](https://datatracker.ietf.org/doc/html/rfc2557#section-4)): `content-location : https://contoso-dot-com-account1.blobstore.azure.net/container1/blob2`
- GUID format: `https://00000000-0000-0000-C000-000000000046-account1.blobstore.azure.net/container1/blob2`

:white_check_mark: **DO** return URLs in response headers/bodies in a consistent form regardless of the URL used to reach the resource. Either always a UUID for `<tenant>` or always a single verified domain.

:heavy_check_mark: **YOU MAY** use URLs as values
```http
https://api.contoso.com/items?url=https://resources.contoso.com/shoes/fancy
```

### HTTP Request / Response Pattern
The HTTP Request / Response pattern dictates how your API behaves. For example: POST methods that create resources must be idempotent, GET method results may be cached, the If-Modified and ETag headers offer optimistic concurrency. The URL of a service, along with its request/response bodies, establishes the overall contract that developers have with your service. As a service provider, how you manage the overall request / response pattern should be one of the first implementation decisions you make.

Cloud applications embrace failure. Therefore, to enable customers to write fault-tolerant applications, _all_ service operations (including POST) **must** be idempotent. Implementing services in an idempotent manner, with an "exactly once" semantic, enables developers to retry requests without the risk of unintended consequences.

#### Exactly Once Behavior = Client Retries & Service Idempotency

:white_check_mark: **DO** ensure that _all_ HTTP methods are idempotent.

:ballot_box_with_check: **YOU SHOULD** use PUT or PATCH to create a resource as these HTTP methods are easy to implement, allow the customer to name their own resource, and are idempotent.

:heavy_check_mark: **YOU MAY** use POST to create a resource but you must make it idempotent and, of course, the response **MUST** return the URL of the created resource with a 201-Created. One way to make POST idempotent is to use the Repeatability-Request-ID & Repeatability-First-Sent headers (See [Repeatability of requests](#Repeatability-of-requests)).

##### HTTP Return Codes

:white_check_mark: **DO** adhere to the return codes in the following table when the method completes synchronously and is successful:

Method | Description | Response Status Code
-------|-------------|---------------------
PATCH  | Create/Modify the resource with JSON Merge Patch | `200-OK`, `201-Created`
PUT    | Create/Replace the _whole_ resource | `200-OK`, `201-Created`
POST   | Create new resource (ID set by service) | `201-Created` with URL of created resource
POST   | Action | `200-OK`, `204-No Content` (only when nothing returned in response body)
GET    | Read (i.e. list) a resource collection | `200-OK`
GET    | Read the resource | `200-OK`
DELETE | Remove the resource | `204-No Content`\; avoid `404-Not Found`

:white_check_mark: **DO** return status code `202-Accepted` and follow the guidance in [Long-Running Operations & Jobs](#long-running-operations--jobs) when a PUT, PATCH, POST, or DELETE method completes asynchronously.

:white_check_mark: **DO** treat method names as case sensitive and should always be in uppercase

:white_check_mark: **DO** return the state of the resource after a PUT, PATCH, POST, or GET operation with a `200-OK` or `201-Created`.

:white_check_mark: **DO** return a `204-No Content` without a resource/body for a DELETE operation (even if the URL identifies a resource that does not exist; do not return `404-Not Found`)

:white_check_mark: **DO** return a `403-Forbidden` when the user does not have access to the resource _unless_ this would leak information about the existence of the resource that should not be revealed for security/privacy reasons, in which case the response should be `404-Not Found`. [Rationale: a `403-Forbidden` is easier to debug for customers, but should not be used if even admitting the existence of things could potentially leak customer secrets.]

:white_check_mark: **DO** support caching and optimistic concurrency by honoring the the `If-Match`, `If-None-Match`, if-modified-since, and if-unmodified-since request headers and by returning the ETag and last-modified response headers

### HTTP Query Parameters and Header Values
Because information in the service URL, as well as the request / response, are strings, there must be a predictable, well-defined scheme to convert strings to their corresponding values.

:white_check_mark: **DO** validate all query parameter and request header values and fail the operation with `400-Bad Request` if any value fails validation. Return an error response as described in the [Handling Errors](#Handling-errors) section indicating what is wrong so customer can diagnose the issue and fix it themselves.

:white_check_mark: **DO** use the following table when translating strings:

Data type | Document that string must be
--------- | -------
Boolean   | true / false (all lowercase)
Integer   | -2<sup>53</sup>+1 to +2<sup>53</sup>-1 (for consistency with JSON limits on integers [RFC8259](https://datatracker.ietf.org/doc/html/rfc8259))
Float     | [IEEE-754 binary64](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)
String    | (Un)quoted?, max length, legal characters, case-sensitive, multiple delimiter
UUID      | 123e4567-e89b-12d3-a456-426614174000 (no {}s, hyphens, case-insensitive) [RFC4122](https://datatracker.ietf.org/doc/html/rfc4122)
Date/Time (Header) | Sun, 06 Nov 1994 08:49:37 GMT [RFC 1123, Section 5.2.14](https://datatracker.ietf.org/doc/html/rfc1123#page-55)
Date/Time (Query parameter) | YYYY-MM-DDTHH:mm:ss.sssZ (with at most 3 digits of fractional seconds) [RFC3339](https://datatracker.ietf.org/doc/html/rfc3339)
Byte array | Base-64 encoded, max length
Array      | One of a) a comma-separated list of values (preferred), or b) separate `name=value` parameter instances for each value of the array


The table below lists the headers most used by Azure services:

Header Key          | Applies to | Example
------------------- | ---------- | -------------
_authorization_     | Request    | Bearer eyJ0...Xd6j (Support Azure Active Directory)
_x-ms-useragent_    | Request    | (see [Distributed Tracing & Telemetry](#Distributed-Tracing-&-Telemetry))
traceparent         | Request    | (see [Distributed Tracing & Telemetry](#Distributed-Tracing-&-Telemetry))
tracecontext        | Request    | (see [Distributed Tracing & Telemetry](#Distributed-Tracing-&-Telemetry))
accept              | Request    | application/json
If-Match            | Request    | "67ab43" or * (no quotes) (see [Conditional Requests](#Conditional-Requests))
If-None-Match       | Request    | "67ab43" or * (no quotes) (see [Conditional Requests](#Conditional-Requests))
If-Modified-Since   | Request    | Sun, 06 Nov 1994 08:49:37 GMT (see [Conditional Requests](#Conditional-Requests))
If-Unmodified-Since | Request    | Sun, 06 Nov 1994 08:49:37 GMT (see [Conditional Requests](#Conditional-Requests))
date                | Both       | Sun, 06 Nov 1994 08:49:37 GMT (see [RFC7231, Section 7.1.1.2](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.1.2))
_content-type_      | Both       | application/merge-patch+json
_content-length_    | Both       | 1024
_x-ms-request-id_   | Response   | [see Customer Support](http://TODO:link-goes-here)
ETag                | Response   | "67ab43" (see [Conditional Requests](#Conditional-Requests))
last-modified       | Response   | Sun, 06 Nov 1994 08:49:37 GMT
_x-ms-error-code_   | Response   | (see [Handling Errors](#Handling-Errors))
retry-after         | Response   | 180 (see [RFC 7231, Section 7.1.3](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.3))

:white_check_mark: **DO** support all headers shown in _italics_

:white_check_mark: **DO** specify headers using kebab-casing

:white_check_mark: **DO** compare request header names using case-insensitivity

:white_check_mark: **DO** compare request header values using case-sensitivity if the header name requires it

:white_check_mark: **DO** accept and return date values in headers using the HTTP Date format as defined in [RFC 1123, Section 5.2.14](https://datatracker.ietf.org/doc/html/rfc1123#page-55), e.g. "Sun, 06 Nov 1994 08:49:37 GMT".

Note: RFC 1123 defines the date format as a modification on top of the date format in [RFC 822, Section 5](https://datatracker.ietf.org/doc/html/rfc822#section-5) to support either a 2 or 4 digit year, and further recommending that 4 digit year always be used.

:no_entry: **DO NOT** fail a request that contains an unrecognized header. Headers may be added by API gateways or middleware and this must be tolerated

:no_entry: **DO NOT** use "x-" prefix for custom headers, unless the header already exists in production [[RFC 6648](https://datatracker.ietf.org/doc/html/rfc6648)].

#### Additional References
- [StackOverflow - Difference between http parameters and http headers](https://stackoverflow.com/questions/40492782)
- [Standard HTTP Headers](https://httpwg.org/specs/rfc7231.html#header.field.registration)
- [Why isn't HTTP PUT allowed to do partial updates in a REST API?](https://stackoverflow.com/questions/19732423/why-isnt-http-put-allowed-to-do-partial-updates-in-a-rest-api)

### REpresentational State Transfer (REST)
REST is an architectural style with broad reach that emphasizes scalability, generality, independent deployment, reduced latency via caching, and security. When applying REST to your API, you define your service’s resources as a collections of items.
These are typically the nouns you use in the vocabulary of your service. Your service's [URLs](#URLS) determine the hierarchical path developers use to perform CRUD (create, read, update, and delete) operations on resources. Note, it's important to model resource state, not behavior.
There are patterns, later in these guidelines, that describe how to invoke behavior on your service. See [this article in the Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design) for a more detailed discussion of REST API design patterns.

When designing your service, it is important to optimize for the developer using your API.

:white_check_mark: **DO** focus heavily on clear & consistent naming

:white_check_mark: **DO** ensure your resource paths make sense

:white_check_mark: **DO** simplify operations with few required query parameters & JSON fields

:white_check_mark: **DO** establish clear contracts for string values

:white_check_mark: **DO** use proper response codes/bodies so customer can diagnose their own problems and fix them without contacting Azure support or the service team

#### JSON Resource Schema & Field Mutability

:white_check_mark: **DO** use the same JSON schema for PUT request/response, PATCH response, GET response, and POST request/response on a given URL path. The PATCH request schema should contain all the same fields with no required fields. This allows one SDK type for input/output operations and enables the response to be passed back in a request.

:white_check_mark: **DO** think about your resource's fields and how they are used:

Field Mutability | Service Request's behavior for this field
-----------------| -----------------------------------------
**Create** | Service honors field only when creating a resource. Minimize create-only fields so customers don't have to delete & re-create the resource.
**Update** | Service honors field when creating or updating a resource
**Read**   | Service returns this field in a response. If the client passed a read-only field, the service **MUST** fail the request unless the passed-in value matches the resource's current value

In addition to the above, a field may be "required" or "optional". A required field is guaranteed to always exist and will typically _not_ become a nullable field in a SDK's data structure. This allows customers to write code without performing a null-check.
Because of this, required fields can only be introduced in the 1st version of a service; it is a breaking change to introduce required fields in a later version. In addition, it is a breaking change to remove a required field or make an optional field required or vice versa.

:white_check_mark: **DO** make fields simple and maintain a shallow hierarchy.

:white_check_mark: **DO** use camel case for all JSON field names. Do not upper-case acronyms; use camel case.

:white_check_mark: **DO** treat JSON field names with case-sensitivity.

:white_check_mark: **DO** treat JSON field values with case-sensitivity. There may be some exceptions (e.g. GUIDs) but avoid if at all possible.

:white_check_mark: **DO** use GET for resource retrieval and return JSON in the response body

:white_check_mark: **DO** create and update resources using PATCH [RFC5789] with JSON Merge Patch [(RFC7396)](https://datatracker.ietf.org/doc/html/rfc7396) request body.

:white_check_mark: **DO** use PUT with JSON for wholesale create/update operations. **NOTE:** If a v1 client PUTs a resource; any fields introduced in V2+ should be reset to their default values (the equivalent to DELETE followed by PUT).

:white_check_mark: **DO** use DELETE to remove a resource.

:white_check_mark: **DO** fail an operation with `400-Bad Request` if the request is improperly-formed or if any JSON field name or value is not fully understood by the specific version of the service. Return an error response as described in [Handling errors](#Handling-errors) indicating what is wrong so customer can diagnose the issue and fix it themselves.

:heavy_check_mark: **YOU MAY** return secret fields via POST **if absolutely necessary**.

:no_entry: **DO NOT** return secret fields via GET. For example, do not return `administratorPassword` in JSON.

:no_entry: **DO NOT** add fields to the JSON if the value is easily computable from other fields to avoid bloating the body.

##### Create / Update / Replace Processing Rules

:white_check_mark: **DO** follow the processing below to create/update/replace a resource:

When using this method | if this condition happens | use&nbsp;this&nbsp;response&nbsp;code
---------------------- | ------------------------- | ----------------------
PATCH/PUT | Any JSON field name/value not known/valid | `400-Bad Request`
PATCH/PUT | Any Read field passed (client can't set Read fields) | `400-Bad Request`
| **If&nbsp;the&nbsp;resource&nbsp;does&nbsp;not&nbsp;exist** |
PATCH/PUT | Any mandatory Create/Update field missing | `400-Bad Request`
PATCH/PUT | Create resource using Create/Update fields | `201-Created`
| **If&nbsp;the&nbsp;resource&nbsp;already&nbsp;exists** |
PATCH | Any Create field doesn't match current value (allows retries) | `409-Conflict`
PATCH | Update resource using Update fields | `200-OK`
PUT | Any mandatory Create/Update field missing | `400-Bad Request`
PUT | Overwrite resource entirely using Create/Update fields | `200-OK`

#### Handling Errors
There are 2 kinds of errors:
- An error where you expect customer code to gracefully recover at runtime
- An error indicating a bug in customer code that is unlikely to be recoverable at runtime; the customer must just fix their code

:white_check_mark: **DO** return an `x-ms-error-code` response header with a string error code indicating what went wrong.

*NOTE: Error code values are part of your API contract (because customer code is likely to do comparisons against them) and cannot change in the future.*

:white_check_mark: **DO** carefully craft `x-ms-error-code` string values for errors that are recoverable at runtime.

:white_check_mark: **DO** ensure that the top-level `code` field's value is identical to the `x-ms-error-code` header's value.

:white_check_mark: **DO** document the service's error code strings; they are part of the API contract.

:white_check_mark: **DO** provide a response body with the following structure:

**ErrorResponse** : Object

Property | Type | Required | Description
-------- | ---- | :------: | -----------
`error` | ErrorDetail | ✔ | The error object.

**ErrorDetail** : Object

Property | Type | Required | Description
-------- | ---- | :------: | -----------
`code` | String | ✔ | One of a server-defined set of error codes.
`message` | String | ✔ | A human-readable representation of the error.
`target` | String |  | The target of the error.
`details` | ErrorDetail[] |  | An array of details about specific errors that led to this reported error.
`innererror` | InnerError |  | An object containing more specific information than the current object about the error.

**InnerError** : Object

Property | Type | Required | Description
-------- | ---- | :------: | -----------
`code` | String |  | A more specific error code than was provided by the containing error.
`innererror` | InnerError |  | An object containing more specific information than the current object about the error.

Example:
```json
{
  "error": {
    "code": "InvalidPasswordFormat",
    "message": "Human-readable description",
    "target": "target of error",
    "innererror": {
      "code": "PasswordTooShort",
      "minLength": 6,
    }
  }
}
```

:heavy_check_mark: **YOU MAY** group common customer code errors into a few `x-ms-error-code` string values.

:heavy_check_mark: **YOU MAY** treat the other fields as you wish as they are _not_ considered part of your service's API contract and customers should not take a dependency on them or their value. They exist to help customers self-diagnose issues.


### JSON
Services, and the clients that access them, may be written in multiple languages. To ensure interoperability, JSON establishes the "lowest common denominator" type system, which is always sent over the wire as UTF-8 bytes. This system is very simple and consists of three types:

 Type | Description
 ---- | -----------
 Boolean | true/false (always lowercase)
 Number  | Signed floating point (IEEE-754 binary64; int range: -2<sup>53</sup>+1 to +2<sup>53</sup>-1)
 String  | Used for everything else

:white_check_mark: **DO** use integers within the acceptable range of JSON number.

:white_check_mark: **DO** establish a well-defined contract for the format of strings. For example, determine maximum length, legal characters, case-(in)sensitive comparisons, etc. Where possible, use standard formats, e.g. RFC3339 for date/time.

:white_check_mark: **DO** use strings formats that are well-known and easily parsable/formattable by many programming languages, e.g. RFC3339 for date/time.

:white_check_mark: **DO** ensure that information exchanged between your service and any client is "round-trippable" across multiple programming languages.

:white_check_mark: **DO** use [RFC3339](https://datatracker.ietf.org/doc/html/rfc3339) for date/time.

:white_check_mark: **DO** use [RFC4122](https://datatracker.ietf.org/doc/html/rfc4122) for UUIDs.

:heavy_check_mark: **YOU MAY** use JSON objects to group sub-fields together.

:heavy_check_mark: **YOU MAY** use JSON arrays if maintaining an order of values is required. Avoid arrays in other situations since arrays can be difficult and inefficient to work with, especially with JSON Merge Patch where the entire array needs to be read prior to any operation being applied to it.

:ballot_box_with_check: **YOU SHOULD** use JSON objects instead of arrays whenever possible.

#### Enums & SDKs (Client libraries)
It is common for strings to have an explicit set of values. These are often reflected in the OpenAPI definition as enumerations. These are extremely useful for developer tooling, e.g. code completion, and client library generation.

However, it is not uncommon for the set of values to grow over the life of a service. For this reason, Microsoft's tooling uses the concept of an "extensible enum," which indicates that the set of values should be treated as only a _partial_ list.
This indicates to client libraries and customers that values of the enumeration field should be effectively treated as strings and that undocumented value may returned in the future. This enables the set of values to grow over time while ensuring stability in client libraries and customer code.

:ballot_box_with_check: **YOU SHOULD** use extensible enumerations unless you are positive that the symbol set will NEVER change over time.

:white_check_mark: **DO** document to customers that new values may appear in the future so that customers write their code today expecting these new values tomorrow.

:no_entry: **DO NOT** remove values from your enumeration list as this breaks customer code.

#### Polymorphic types
:warning: **YOU SHOULD NOT** use polymorphic JSON types because they greatly complicate the customer code due to runtime dynamic casts and the introduction of new types in the future.

If you can't avoid them, then follow the guideline below.

:white_check_mark: **DO** define a `kind` field indicating the kind of the resource and include any kind-specific fields in the body.

Below is an example of JSON for a Rectangle and Circle:
**Rectangle**
```json
{
   "kind": "rectangle",
   "x": 100,
   "y": 50,
   "width": 10,
   "length": 24,
   "fillColor": "Red",
   "lineColor": "White",
   "subscription": {
      "kind": "free"
   }
}
```

**Circle**
 ```json
{
   "kind": "circle",
   "x": 100,
   "y": 50,
   "radius": 10,
   "fillColor": "Green",
   "lineColor": "Black",
   "subscription": {
      "kind": "paid",
      "expiration": "2024",
      "invoice": "123456"
   }
}
```
Both Rectangle and Circle have common fields: `kind`, `fillColor`, `lineColor`, and `subscription`. A Rectangle also has `x`, `y`, `width`, and `length` while a Circle has `x`, `y`, and `radius`. The `subscription` is a nested polymorphic type. A `free` subscription has no additional fields and a `paid` subscription has `expiration` and `invoice` fields.

## Common API Patterns

### Performing an Action
The REST specification is used to model the state of a resource, and is primarily intended to handle CRUD (Create, Read, Update, Delete) operations. However, many services require the ability to perform an action on a resource, e.g. getting the thumbnail of an image, sending an SMS message.  It is also sometimes useful to perform an action on a collection.

:white_check_mark: **DO** pattern your URL like this to perform an action on a resource
**URL Pattern**
```http
https://.../<resource-collection>/<resource-id>:<action>?<input parameters>
```

**Example**
```http
https://.../users/Bob:send-sms?text="Hello"
```

**Equivalent to (in C#)**
```users["Bob"].SendSms("Hello")```

:white_check_mark: **DO** pattern your URL like this to perform an action on a collection
**URL Pattern**
```http
https://.../<resource-collection>:<action>?<input parameters>
```

**Example**
```http
https://.../users:grant?access=read
```

Note: To avoid potential collision of actions and resource ids, you should disallow the use of the ":" character in resource ids.

:white_check_mark: **DO** use a POST operation for any action on a resource or collection.

:white_check_mark: **DO** support the Repeatability-Request-ID & Repeatability-First-Sent request headers if the action needs to be idempotent if retries occur.

:white_check_mark: **DO** return a `200-OK` when the action completes synchronously and successfully.

:ballot_box_with_check: **YOU SHOULD** use a verb to name your action.


### Collections
:white_check_mark: **DO** structure the response to a list operation as an object with a top-level array field containing the set (or subset) of resources.

:ballot_box_with_check: **YOU SHOULD** support paging today if there is ever a chance in the future that the number of items can grow to be very large.
> NOTE: It is a breaking change to add paging in the future

:heavy_check_mark: **YOU MAY** expose an operation that lists your resources by supporting a GET method with a URL to a resource-collection (as opposed to a resource-id).

**Example Response Body**
```json
{
    "value": [
       { "id": "Item 01", "etag": "0xabc", "price": 99.95, "sizes": null },
       { … },
       { … },
       { "id": "Item 99", "etag": "0xdef", "price": 59.99, "sizes": null }
    ],
    "nextLink": "{opaqueUrl}"
 }
```

:white_check_mark: **DO** include the _id_ field and _etag_ field (if supported) for each item as this allows the customer to modify the item in a future operation.

:white_check_mark: **DO** clearly document that resources may be skipped or duplicated across pages of a paginated collection unless the operation has made special provisions to prevent this (like taking a time-expiring snapshot of the collection).

:white_check_mark: **DO** return a `nextLink` field with an absolute URL that the client can GET in order to retrieve the next page of the collection.

:ballot_box_with_check: **YOU SHOULD** use `value` as the name of the top-level array field unless a more appropriate name is available.

:no_entry: **DO NOT** return the `nextLink` field at all when returning the last page of the collection.

:no_entry: **DO NOT** ever return a `nextLink` field with a value of null.

#### Query options
:heavy_check_mark: **YOU MAY** support the following query parameters allowing customers to control the list operation:

Parameter&nbsp;name | Type | Description
------------------- | ---- | -----------
`filter`       | string            | an expression on the resource type that selects the resources to be returned
`orderby`      | string&nbsp;array | a list of expressions that specify the order of the returned resources
`skip`         | integer           | an offset into the collection of the first resource to be returned
`top`          | integer           | the maximum number of resources to return from the collection
`maxpagesize`  | integer           | the maximum number of resources to include in a single response
`select`       | string&nbsp;array | a list of field names to be returned for each resource
`expand`       | string&nbsp;array | a list of the related resources to be included in line with each resource

:white_check_mark: **DO** return an error if the client specifies any parameter not supported by the service.

:white_check_mark: **DO** treat these query parameter names as case-sensitive.

:white_check_mark: **DO** apply `select` or `expand` options after applying all the query options in the table above.

:white_check_mark: **DO** apply the query options to the collection in the order shown in the table above.

:no_entry: **DO NOT** prefix any of these query parameter names with "$" (the convention in the [OData standard](http://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html#sec_QueryingCollections)).

#### `filter`

:heavy_check_mark: **YOU MAY** support `filter`ing of the results of a list operation with the `filter` query parameter.

The value of the `filter` option is an expression involving the fields of the resource that produces a Boolean value. This expression is evaluated for each resource in the collection and only items where the expression evaluates to true are included in the response.

:white_check_mark: **DO** omit all resources from the collection for which the `filter` expression evaluates to false or to null, or references properties that are unavailable due to permissions.

Example: return all Products whose Price is less than $10.00

```http
GET https://api.contoso.com/products?`filter`=price lt 10.00
```

##### `filter` operators

:heavy_check_mark: **YOU MAY** support the following operators in `filter` expressions:

Operator                 | Description           | Example
--------------------     | --------------------- | -----------------------------------------------------
**Comparison Operators** |                       |
eq                       | Equal                 | city eq 'Redmond'
ne                       | Not equal             | city ne 'London'
gt                       | Greater than          | price gt 20
ge                       | Greater than or equal | price ge 10
lt                       | Less than             | price lt 20
le                       | Less than or equal    | price le 100
**Logical Operators**    |                       |
and                      | Logical and           | price le 200 and price gt 3.5
or                       | Logical or            | price le 3.5 or price gt 200
not                      | Logical negation      | not price le 3.5
**Grouping Operators**   |                       |
( )                      | Precedence grouping   | (priority eq 1 or city eq 'Redmond') and price gt 100

:white_check_mark: **DO** respond with an error message as defined in the [Handling Errors](handling-errors) section if a client includes an operator in a `filter` expression that is not supported by the operation.

:white_check_mark: **DO** use the following operator precedence for supported operators when evaluating `filter` expressions. Operators are listed by category in order of precedence from highest to lowest. Operators in the same category have equal precedence and should be evaluated left to right:

| Group           | Operator | Description
| ----------------|----------|------------
| Grouping        | ( )      | Precedence grouping   |
| Unary           | not      | Logical Negation      |
| Relational      | gt       | Greater Than          |
|                 | ge       | Greater than or Equal |
|                 | lt       | Less Than             |
|                 | le       | Less than or Equal    |
| Equality        | eq       | Equal                 |
|                 | ne       | Not Equal             |
| Conditional AND | and      | Logical And           |
| Conditional OR  | or       | Logical Or            |

> :heavy_check_mark: **YOU MAY** support orderby and `filter` functions such as concat and contains. For more information, see [odata Canonical Functions](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part2-url-conventions.html#_Toc31360979).

##### Operator examples
The following examples illustrate the use and semantics of each of the logical operators.

Example: all products with a name equal to 'Milk'

```http
GET https://api.contoso.com/products?`filter`=name eq 'Milk'
```

Example: all products with a name not equal to 'Milk'

```http
GET https://api.contoso.com/products?`filter`=name ne 'Milk'
```

Example: all products with the name 'Milk' that also have a price less than 2.55:

```http
GET https://api.contoso.com/products?`filter`=name eq 'Milk' and price lt 2.55
```

Example: all products that either have the name 'Milk' or have a price less than 2.55:

```http
GET https://api.contoso.com/products?`filter`=name eq 'Milk' or price lt 2.55
```

Example: all products that have the name 'Milk' or 'Eggs' and have a price less than 2.55:

```http
GET https://api.contoso.com/products?`filter`=(name eq 'Milk' or name eq 'Eggs') and price lt 2.55
```

#### orderby

:heavy_check_mark: **YOU MAY** support sorting of the results of a list operation with the `orderby` query parameter.
*NOTE: It is unusual for a service to support `orderby` because it is very expensive to implement as it requires sorting the entire large collection before being able to return any results.*

The value of the `orderby` parameter is a comma-separated list of expressions used to sort the items.
A special case of such an expression is a property path terminating on a primitive property.

Each expression in the `orderby` parameter value may include the suffix "asc" for ascending or "desc" for descending, separated from the expression by one or more spaces.

:white_check_mark: **DO** sort the collection in ascending order on an expression if "asc" or "desc" is not specified.

:white_check_mark: **DO** sort NULL values as "less than" non-NULL values.

:white_check_mark: **DO** sort items by the result values of the first expression, and then sort items with the same value for the first expression by the result value of the second expression, and so on.

:white_check_mark: **DO** use the inherent sort order for the type of the field. For example, date-time values should be sorted chronologically and not alphabetically.

:white_check_mark: **DO** respond with an error message as defined in the [Handling Errors](#Handling-errors) section if the client requests sorting by a field that is not supported by the operation.

For example, to return all people sorted by name in ascending order:
```http
GET https://api.contoso.com/people?orderby=name
```

For example, to return all people sorted by name in descending order and a secondary sort order of hireDate in ascending order.
```http
GET https://api.contoso.com/people?orderby=name desc,hireDate
```

Sorting MUST compose with `filter`ing such that:
```http
GET https://api.contoso.com/people?`filter`=name eq 'david'&orderby=hireDate
```
will return all people whose name is David sorted in ascending order by hireDate.

##### Considerations for sorting with pagination

:white_check_mark: **DO** use the same `filter`ing options and sort order for all pages of a paginated list operation response.

##### skip
:white_check_mark: **DO** define the `skip` parameter as an integer with a default and minimum value of 0.

:heavy_check_mark: **YOU MAY** allow clients to pass the `skip` query parameter to specify an offset into collection of the first resource to be returned.
##### top

:heavy_check_mark: **YOU MAY** allow clients to pass the `top` query parameter to specify the maximum number of resources to return from the collection.

If supporting `top`:
:white_check_mark: **DO** define the `top` parameter as an integer with a minimum value of 1. If not specified, `top` has a default value of infinity.

:white_check_mark: **DO** return the collection's `top` number of resources (if available), starting from `skip`.

##### maxpagesize

:heavy_check_mark: **YOU MAY** allow clients to pass the `maxpagesize` query parameter to specify the maximum number of resources to include in a single page response.

:white_check_mark: **DO** define the `maxpagesize` parameter as an optional integer with a default value appropriate for the collection.

:white_check_mark: **DO** make clear in documentation of the `maxpagesize` parameter that the operation may choose to return fewer resources than the value specified.

### API Versioning

Azure services need to change over time. However, when changing a service, there are 2 requirements:
 1. Already-running customer workloads must not break due to a service change
 2. Customers can adopt a new service version without requiring any code changes (Of course, the customer must modify code to leverage any new service features.)

*NOTE: the [Azure Breaking Change Policy](http://aka.ms/AzBreakingChangesPolicy/) has tables (section 5) describing what kinds of changes are considered breaking. Breaking changes are allowable (due to security/compliance/etc.) if approved by the [Azure Breaking Change Reviewers](mailto:azbreakchangereview@microsoft.com) but only following ample communication to customers and a lengthy deprecation period.*

:white_check_mark: **DO** review any API changes with the Azure API Stewardship Board

:white_check_mark: **DO** use an `api-version` query parameter with a date value

```http
PUT https://service.azure.com/users/Jeff?api-version=2021-06-04
```

:white_check_mark: **DO** use a later date for each new preview version

When releasing a new preview, the service team may completely retire any previous preview versions after giving customers at least 90 days to upgrade their code

:white_check_mark: **DO** use a later date for successive preview versions.

:no_entry: **DO NOT** introduce any breaking changes into the service.

:no_entry: **DO NOT** include a version number segment in any operation path.

:no_entry: **DO NOT** use the same date when transitioning from a preview API to a GA API. If the preview `api-version` is '2021-06-04-preview', the GA version of the API **must be** a date later than 2021-06-04

:no_entry: **DO NOT** keep a preview feature in preview for more than 1 year; it must go GA (or be removed) within 1 year after introduction.

#### Use Extensible Enums

While removing a value from an enum is a breaking change, adding value to an enum can be handled with an _extensible enum_.  An extensible enum is a string value that has been marked with a special marker - setting `modelAsString` to true within an `x-ms-enum` block.  For example:

```json
"createdByType": {
   "type": "string",
   "description": "The type of identity that created the resource.",
   "enum": [
      "User",
      "Application",
      "ManagedIdentity",
      "Key"
   ],
   "x-ms-enum": {
      "name": "createdByType",
      "modelAsString": true
   }
}
```

> :ballot_box_with_check: **You SHOULD** use extensible enums unless you are positive that the symbol set will **NEVER** change over time.

#### Version Discovery

Simpler clients may be hardcoded to a single version of a service. Since Azure services offer each version for a well-known period of time, a client that’s regularly maintained can be always operational without further complexity as long as during regular maintenance the client is moved forward to new versions in advance of older ones being retired.

API version discovery is needed when either a given hosted service may expose a different API version to different clients (e.g. latest API version only available in certain regions or to certain tenants) or the service itself may exist in different instances (e.g. a service that may be run on Azure or hosted on-premises).
In both of those cases clients may get ahead of services in the API version they use. In might also be possible for a client version to ship ahead of its corresponding service update, leading to the same situation. Lastly, version discovery is useful for clients that want to warn operators that an API they depend on may expire soon.

:white_check_mark: **DO**  support API version discovery, including

1. Support HTTP `OPTIONS` requests against all resources, including the root URL for a given tenant or the global root if no tenant identity is tracked or not a multi-tenant service

2. Include the `api-supported-versions` header, containing a comma-separated list of versions conforming to the Azure versioning scheme. This list must include all group versions as well as all major-minor versions supported by the target resource. For cases where no specific version applies (e.g. sometimes the root resource), the list still must contain the group versions supported by the service.

3. If a given service supports versions of the API that are known to be planned for deprecation in a year or less, it must include those versions (group and major.minor) in the `api-deprecated-versions` header.

4. For services that do rolling updates where there is a point in time where some front-ends are ahead of others version-wise, all front-ends **MUST** report the previous version as the latest version until the rolling update covers all instances and only then switch over to reporting the new latest version. This ensures that clients will not detect a version and then get load-balanced into a front-end that does not support it yet.

:ballot_box_with_check: **YOU SHOULD** support the following for version discovery:

1. In addition to the functionality described here, services should support HTTP `OPTIONS` requests for other purposes such as further discovery, CORS, etc.

2. Services should allow unauthenticated HTTP `OPTIONS` requests. When doing so, authors need to consider whether HTTP `OPTIONS` requests against non-existing resources result in 404s and whether that is leaking sensitive information. Certain scenarios, such as support for CORS pre-flight requests, require allowing unauthenticated HTTP `OPTIONS` requests.

3. If using OData and addressing an expanded resource, the HTTP `OPTIONS` request should return the group versions that are supported across the expanded set.

Example request to discover API versions (blob storage container list API):

```text
OPTIONS /?comp=list HTTP/1.1
host: accountname.blob.core.azure.net
```

Example response:

```text
200 OK
api-supported-versions: 2011-08,2012-02,1.1,2.0
api-deprecated-versions: 2009-04,1.0
Content-Length: 0
```

Clients that use version discovery are expected to cache version information. Since there’s a year of lead time after an API version shows in the `api-deprecated-versions` before it’s removed, checking once a week should provide sufficient lead time to client authors or operators.
In the rare case where a server rolls back a version that clients are already using, the service will reject requests because they are ahead of the latest version supported. Whenever a client sees a `version-too-new` error, it should re-execute its version discovery procedure.

### Repeatability of requests

The ability to retry failed requests for which a client never received a response greatly simplifies the ability to write resilient distributed applications. While HTTP designates some methods as safe and/or idempotent (and thus retryable), being able to retry other operations such as create-using-POST-to-collection is desirable.

:ballot_box_with_check: **YOU SHOULD** support repeatable requests according as defined in [OASIS Repeatable Requests Version 1.0](https://docs.oasis-open.org/odata/repeatable-requests/v1.0/repeatable-requests-v1.0.html).

- The tracked time window (difference between the `Repeatability-First-Sent` value and the current time) **MUST** be at least 5 minutes.
- A service advertises support for repeatability requests by adding the `Repeatability-First-Sent` and `Repeatability-Request-ID` to the set of headers for a given operation.
- When understood, all endpoints co-located behind a DNS name **MUST** understand the header. This means that a service **MUST NOT** ignore the presence of a header for any endpoints behind the DNS name, but rather fail the request containing a `Repeatability-Request-ID` header if that particular endpoint lacks support for repeatable requests. Such partial support **SHOULD** be avoided due to the confusion it causes for clients.

<!-- Open API Spec -->
[OpenAPI Specification]: https://github.com/OAI/OpenAPI-Specification/blob/main/versions/2.0.md

### Long-Running Operations & Jobs

When the processing for an operation may take a significant amount of time to complete, it should be
implemented as a _long-running operation (LRO)_. This allows clients to continue running while the
operation is being processed. The client obtains the outcome of the operation at some later time
through another API call.
See the [Long Running Operations section](./ConsiderationsForServiceDesign.md#long-running-operations) in
Considerations for Service Design for an introduction to the design of long running operations.

:white_check_mark: **DO** implement an operation as an LRO if the 99th percentile response time is greater than 1s.

In rare instances where an operation may take a _very long_ time to complete, e.g. longer than 15 minutes,
it may be better to expose this as a first class resource of the API rather than as an operation on another resource.

There are two basic patterns that can be used for long-running operations:
1. Resource-based long-running operations (RELO)
2. Long-running operations with status monitor

:white_check_mark: **DO** use the RELO pattern when the operation is on a resource that contains a "status" property that can be used to obtain the outcome of the operation.

:ballot_box_with_check: **YOU SHOULD** only use the status monitor LRO pattern when the RELO pattern is not applicable.

#### Resource-based long-running operations

Some common situations where the RELO pattern should be used:
1. A "create" operation (PUT, PATCH, or POST) for a resource where the basic structure of the resource is created immediately and includes a status field that indicates when the create has completed, e.g. "provisioning" -> "active".
2. An action operation for a resource where both the initiation of the action and the completion of the action cause a change to the "status" property of the resource.

:white_check_mark: **DO** return a `200-OK` response, `201-Created` for create operations, from the request that initiates the operation.  The response body should contain a representation of the resource that clearly indicates that the operation has been accepted or started.

:white_check_mark: **DO** support a get method on the resource that returns a representation of the resource including the status field that indicates when the operation has completed.

:white_check_mark: **DO** define the "status" field of the resource as an enum with all the values it may contain including the "terminal" values "Succeeded", "Failed", and "Canceled".

:ballot_box_with_check: **YOU SHOULD** use the name `status` for the "status" field of the resource.

#### Long-running operations with status monitor

In a long-running operation with status monitor, the client makes a request to initiate the operation processing and receives a URL in the response where it can obtain the operation results. The [HTTP specification](https://datatracker.ietf.org/doc/html/rfc7231#section-6.3.3) calls the target of this URL a "status monitor".

:white_check_mark: **DO** return a `202-Accepted` status code from the request that initiates an LRO with status monitor if the processing of the operation was successfully initiated.

:white_check_mark: **DO** perform as much validation of the initial request as practical and return an error response immediately when appropriate (without starting the operation).

:white_check_mark: **DO** return the status monitor URL in the `Operation-Location` response header.

:white_check_mark: **DO** support the `get` method on the status monitor endpoint that returns a `200-OK` response with a response body that contains the completion status of the operation with sufficient information to diagnose any potential failures.

:white_check_mark: **DO** include a field in the status monitor resource named `status` indicating the operation's status. This field should be a string with well-defined values. Indicate the terminal state using "Succeeded", "Failed", or "Canceled".

:white_check_mark: **DO** include a field in the status monitor named `error` to contain error information -- minimally `code` and `message` fields -- when an operation fails.

:white_check_mark: **DO** retain the status monitor resource for some documented period of time (at least 24 hours) after the operation completes.

:white_check_mark: **DO** include a `Retry-After` header in the response to the initiating request and requests to the operation-location URL. The value of this header should be an integer number of seconds to wait before making the next request to the operation-location URL.

:heavy_check_mark: **YOU MAY** support a `get` method on the status monitor collection URL that returns a list of status monitors for all recently initiated operations.

:warning: **YOU SHOULD NOT** return any other `2xx` status code from the initial request of a status-monitor LRO -- return `202-Accepted` and a status monitor URL even if processing was completed before the initiating request returns.

:no_entry: **DO NOT** return any data in the response body of a `202-Accepted` response.

Previous Azure guidelines specified "Azure-AsyncOperation" as the name of the response header containing the status monitor URL.

:white_check_mark: **DO** return **both** `Azure-AsyncOperation` and `Operation-Location` headers if your service previously returned `Azure-AsyncOperation`, even though they are redundant, so that existing clients will continue to operate.

:white_check_mark: **DO** return the same value for **both** headers.

:white_check_mark: **DO** look for **both** headers in client code, preferring the `Operation-Location` header.

### Bring your own Storage
When implementing your service, it is very common to store and retrieve data and files. When you encounter this scenario, avoid implementing your own storage strategy and instead use Azure Bring Your Own Storage (BYOS). BYOS provides significant benefits to service implementors, e.g. security, an aggressively optimized frontend, uptime, etc.
While Azure Managed Storage may be easier to get started with, as your service evolves and matures, BYOS will provide the most flexibility and implementation choices. Further, when designing your APIs, be cognizant of expressing storage concepts and how clients will access your data. For example, if you are working with blobs, then you should not expose the concept of folders, nor do they have extensions.

:white_check_mark: **DO** use Azure Bring Your Own Storage.

:white_check_mark: **DO** use a blob prefix

:no_entry: **DO NOT** require a fresh container per operation

#### Authentication
How you secure and protect the data and files that your service uses will not only affect how consumable your API is, but also, how quickly you can evolve and adapt it. Implementing Role Based Access Control [RBAC](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview) is the recommended approach.
It is important to recognize that any roles defined in RBAC essentially become part of your API contract. For example, changing a role's permissions, e.g. restricting access, could effectively cause existing clients to break, as they may no longer have access to necessary resources.

:white_check_mark: **DO** Add RBAC roles for every service operation that requires accessing Storage scoped to the exact permissions.

:white_check_mark: **DO** Ensure that RBAC roles are backward compatible, and specifically, do not take away permissions from a role that would break the operation of the service. Any change of RBAC roles that results in a change of the service behavior is considered a breaking change.

##### Handling 'downstream' errors
It is not uncommon to rely on other services, e.g. storage, when implementing your service. Inevitably, the services you depend on will fail. In these situations, you can include the downstream error code and text in the inner-error of the response body. This provides a consistent pattern for handling errors in the services you depend upon.

:white_check_mark: **DO** include error from downstream services as the 'inner-error' section of the response body.

#### Working with files
Generally speaking, there are two patterns that you will encounter when working with files; single file access, and file collections.

##### Single file access
Desiging an API for accessing a single file, depending on your scenario, is relatively straight forward.

:heavy_check_mark: **YOU MAY** use a Shared Access Signature [SAS](https://docs.microsoft.com/en-us/azure/storage/common/storage-sas-overview) to provide access to a single file. SAS is considered the minimum security for files and can be used in lieu of, or in addition to, RBAC.

:ballot_box_with_check: **YOU SHOULD** if using HTTP (not HTTPS) document to users that all information is sent over the wire in clear text.

:ballot_box_with_check: **YOU SHOULD** support managed identity using Azure Storage by default (if using Azure services).

###### File Versioning
Depending on your requirements, there are scenarios where users of your service will require a specific version of a file. For example, you may need to keep track of configuration changes over time to be able to rollback to a previous state. In these scenarios, you will need to provide a mechanism for accessing a specific version.

:white_check_mark: **DO** Enable the customer to provide an ETag to specify a specific version of a file.
##### File Collections
When your users need to work with multiple files, for example a document translation service, it will be important to provide them access to the collection, and its contents, in a consistent manner. Because there is no industry standard for working with containers, these guidelines will recommend that you leverage Azure Storage. Following the guidelines above, you also want to ensure that you don't expose file system constructs, e.g. folders, and instead use storage constructs, e.g. blob prefixes.

:white_check_mark: **DO** When using a Shared Access Signature (SAS), ensure this is assigned to the container and that the permissions apply to the content as well.

:white_check_mark: **DO** When using managed identity, ensure the customer has given the proper permissions to access the file container to the service.

A common pattern when working with multiple files is for your service to receive requests that contain the location(s) of files to process, e.g. "input" and a location(s) to place the any files that result from processing, e.g. "output." (Note: the terms "input" and "output" are just examples and terms more relevant to the service domain are more appropriate.)

For example, in a request payload may look similar to the following:

```json
{
"input":{
    "location": "https://mycompany.blob.core.windows.net/documents/english/?<sas token>",
    "delimiter":"/"
    },
"output":{
    "location": "https://mycompany.blob.core.windows.net/documents/spanglish/?<sas token>",
    "delimiter":"/"
    }
}
```
Note: How the service gets the request body is outside the purview of these guidelines.

Depending on the requirements of the service, there can be any number of "input" and "output" sections, including none. However, for each of the "input" sections the following apply:

:white_check_mark: **DO** include a JSON object that has string values for "location" and "delimiter."

:white_check_mark: **DO** use a URL to a blob prefix with a container scoped SAS on the end with a minimum of `listing` and `read` permissions.

For each of the "output" sections the following apply:

:white_check_mark: **DO** use a URL to a blob prefix with a container scoped SAS on the end with a minimum of `write` permissions

### Conditional Requests
When designing an API, you will almost certainly have to manage how your resource is updated. For example, if your resource is a bank account, you will want to ensure that one transaction--say depositing money--does not overwrite a previous transaction.
Similarly, it could be very expensive to send a resource to a client. This could be because of its size, network conditions, or a myriad of other reasons. To enable this level of control, services should leverage an `ETag` header, or "entity tag," which will identify the 'version' or 'instance' of the resource a particular client is working with.
An `ETag` is always set by the service and will enable you to _conditionally_ control how your service responds to requests, enabling you to provide predictable updates and more efficient access.

:ballot_box_with_check: **YOU SHOULD** return an `ETag` with any operation returning the resource or part of a resource or any update of the resource (whether the resource is returned or not).

:ballot_box_with_check: **YOU SHOULD** use `ETag`s consistently across your API, i.e. if you use an `ETag`, accept it on all other operations.

You can learn more about conditional requests by reading [RFC7232](https://datatracker.ietf.org/doc/html/rfc7232).

#### Cache Control
One of the more common uses for `ETag` headers is cache control, also referred to a "conditional GET." This is especially useful when resources are large in size, expensive to compute/calculate, or hard to reach (significant network latency). That is, using the value of the `ETag` , the server can determine if the resource has changed. If there are no changes, then there is no need to return the resource, as the client already has the most recent version.

Implementing this strategy is relatively straightforward. First, you will return an `ETag` with a value that uniquely identifies the instance (or version) of the resource. The [Computing ETags](#computing-ETags) section provides guidance on how to properly calculate the value of your `ETag`.
In these scenarios, when a request is made by the client an `ETag` header is returned, with a value that uniquely identifies that specific instance (or version) of the resource. The `ETag` value can then be sent in subsequent requests as part of the `If-None-Match` header.
This tells the service to compare the `ETag` that came in with the request, with the latest value that it has calculated. If the two values are the same, then it is not necessary to return the resource to the client--it already has it. If they are different, then the service will return the latest version of the resource, along with the updated `ETag` value in the header.

:ballot_box_with_check: **YOU SHOULD** implement conditional read strategies

When supporting conditional read strategies:
:white_check_mark: **DO** adhere to the following table for guidance:

| GET Request | Return code | Response                                    |
|:------------|:------------|:--------------------------------------------|
| ETag value = `If-None-Match` value   | `304-Not Modified` | no additional information   |
| ETag value != `If-None-Match` value  | `200-OK`           | Response body include the serialized value of the resource (typically JSON)    |

For more control over caching, please refer to the `cache-control` [HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control).

#### Optimistic Concurrency
An `ETag` should also be used to reflect the create, update, and delete policies of your service. Specifically, you should avoid a "pessimistic" strategy where the 'last write always wins." These can be expensive to build and scale because avoiding the "lost update" problem often requires sophisticated concurrency controls.
Instead, implement an "optimistic concurrency" strategy, where the incoming state of the resource is first compared against what currently resides in the service. Optimistic concurrency strategies are implemented through the combination of `ETags` and the [HTTP Request / Response Pattern](#http-request--response-pattern).

:warning: **YOU SHOULD NOT**  implement pessimistic update strategies, e.g. last writer wins.

When supporting optimistic concurrency:
:white_check_mark: **DO** adhere to the following table for guidance:

| Operation   | Header        | Value | ETag check | Return code | Response       |
|:------------|:--------------|:------|:-----------|:------------|----------------|
| PATCH / PUT | `If-None-Match` | *     | check for _any_ version of the resource ('*' is a wildcard used to match anything), if none are found, create the resource. | `200-OK` or </br> `201-Created` </br> | Response header MUST include the new `ETag` value. Response body SHOULD include the serialized value of the resource (typically JSON).  |
| PATCH / PUT | `If-None-Match` | *     | check for _any_ version of the resource, if one is found, fail the operation |  `412-Precondition Failed` | Response body SHOULD return the serialized value of the resource (typically JSON) that was passed along with the request.|
| PATCH / PUT | `If-Match` | value of ETag     | value of `If-Match` equals the latest ETag value on the server, confirming that the version of the resource is the most current | `200-OK` or </br> `201-Created` </br> | Response header MUST include the new `ETag` value. Response body SHOULD include the serialized value of the resource (typically JSON).  |
| PATCH / PUT | `If-Match` | value of ETag     | value of `If-Match` header DOES NOT equal the latest ETag value on the server, indicating a change has ocurred since after the client fetched the resource|  `412-Precondition Failed` | Response body SHOULD return the serialized value of the resource (typically JSON) that was passed along with the request.|
| DELETE      | `If-None-Match` | value of ETag     | value does NOT match the latest value on the server | `412-Preconditioned Failed` | Response body SHOULD be empty.|
| DELETE      | `If-None-Match` | value of ETag     | value matches the latest value on the server | `204-No Content` | Response body SHOULD be empty.  |

#### Computing ETags
The strategy that you use to compute the `ETag` depends on its semantic. For example, it is natural, for resources that are inherently versioned, to use the version as the value of the `ETag`. Another common strategy for determining the value of an `ETag` is to use a hash of the resource. If a resource is not versioned, and unless computing a hash is prohibitively expensive, this is the preferred mechanism.

:ballot_box_with_check: **YOU SHOULD**, if using a hash strategy, hash the entire resource.

:heavy_check_mark: **YOU MAY** use or, include, a timestamp in your resource schema. If you do this, the timestamp shouldn't be returned with more than subsecond precision, and it SHOULD be consistent with the data and format returned, e.g. consistent on milliseconds.

:heavy_check_mark: **YOU MAY** consider Weak ETags if you have a valid scenario for distinguishing between meaningful and cosmetic changes or if it is too expensive to compute a hash.

### Distributed Tracing & Telemetry
Azure SDK client guidelines specify that client libraries must send telemetry data through the `User-Agent` header, `X-MS-UserAgent` header, and Open Telemetry.
Client libraries are required to send telemetry and distributed tracing information on every  request. Telemetry information is vital to the effective operation of your service and should be a consideration from the outset of design and implementation efforts.

:white_check_mark: **DO** follow the Azure SDK client guidelines for supporting telemetry headers and Open Telemetry.

:no_entry: **DO NOT** reject a call if you have custom headers you don't understand, and specifically, distributed tracing headers.
#### Additional References
- [Azure SDK client guidelines](https://azure.github.io/azure-sdk/general_azurecore.html)
- [Azure SDK User-Agent header policy](https://azure.github.io/azure-sdk/general_azurecore.html#azurecore-http-telemetry-x-ms-useragent)
- [Azure SDK Distributed tracing policy](https://azure.github.io/azure-sdk/general_azurecore.html#distributed-tracing-policy)
- [Open Telemetry](https://opentelemetry.io/)

## Final thoughts
These guidelines describe the upfront design considerations, technology building blocks, and common patterns that Azure teams encounter when building an API for their service. There is a great deal of information in them that can be difficult to follow. Fortunately, at Microsoft, there is a team committed to ensuring your success.

The Azure REST API Stewardship board is a collection of dedicated architects that are passionate about helping Azure service teams build interfaces that are intuitive, maintainable, consistent, and most importantly, delight our customers.
Because APIs affect nearly all downstream decisions, you are encouraged to reach out to the Stewardship board early in the development process.
These architects will work with you to apply these guidelines and identify any hidden pitfalls in your design. For more information on how to part with the Stewardship board, please refer to [Considerations for Service Design](.\ConsiderationsForServiceDesign.md).
