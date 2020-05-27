# FHIR Transactions

## Overview

Bundles are core to transactions. As a front end dev, I frequently come across them when consuming data to show in a view from a fhir API endpoint, but what are they really?

## Bundles

"A common operation performed with resources is to gather a collection of them into a single instance with containing context... In FHIR this is referred to as "bundling" the resources together."
*"Resource Bundle - Content"*, [HL7](https://www.hl7.org/fhir/bundle.html)

Bundles act like a container for a set of independent FHIR resources. They allow us to group and transmit resources across API requests. I want to emphasize the independent nature of resources in bundles, as opposed to “contained resources”. Contained resources are embedded "in" the container resource. And they can't be requested, updated or deleted separately, or referenced individually from outside its containing resource. On the other hand, resources inside a Bundle get created individually on the server and thus can be accessed via REST api, and acted on via CRUD operations.
 
![FHIR bundle structure]("./img/fhir-bundle.png")

1. Metadata describing the Bundle type-which would include the total amount of entries if it’s a searchset- and a link if you need to retrieve it later.
1. Entry: where the resources that the Bundle has are stored. This also includes URLs which you can use to retrieve the resource individually.
1. Request property: Used when you're uploading Bundles - and where you specify to the server what to do with each resource.
1. Response: Used by the server when it's responding to your request - it tells you how each operation went.

Resource bundles can be used to: 
- Find resources that meet some criteria (as in a searchset)
- Find the version of some resources as part of a history operation
- Send messages with resources from app to app ([FHIR Messaging](https://www.hl7.org/fhir/messaging.html))
- Grouping resources as exchangeable and persistable [documents](https://www.hl7.org/fhir/documents.html)
- CRUD-ing resources in a single operation
- Storing a collection of resources


## Types of Bundles

#### Searchsets
The Bundle collates search results into a single response. As some search results can return a large number of results, and paging separates them into manageable blocks.

#### Document
A document is an immutable bundle set of resources with a fixed presentation that is authored and attested by humans, organizations and devices. The first resource is Composition.

#### Messages
In FHIR messaging, a "request message" is sent from a application to another application when an event happens. The request message consists of a Bundle identified by the type "message", with the first resource in the bundle being a MessageHeader resource. The MessageHeader resource has a code - the message event - that identifies the nature of the request message, and it also carries request metadata. The other resources in the bundle depend on the type of the request.

#### History
This is similar to searchset, but is specialised for `_history` operations, for example, looking at all the edits made to a single Patient resource.

![JSON structure of different bundles]("./img/bundle-types.png")

L: Transaction Bundles
The `entry.request` and `entry.response` property is required for batch, transaction, history and their respective responses.
A fullUrl must be unique in a bundle, or else entries with the same fullUrl must have different meta versionId (except in history bundles).

C: 
A bundle of a searchset for all the Officer Jenny Practitioner elements I put into try.smilecdr.com database. The entry has search property only when it's a search


R: A bundle of resources constituting a FHIR Document
A document must have an identifier with a system and a value, a date, and Composition as the first resource. It also has required references such as a subject, encounter, and author.

#### Operation Outcome

Operation Outcomes are returned as the content of a transaction response. They are a FHIR resource with sets of error, warning and information messages describing the success or failure of an action.

How are they used?

- When a REST operation fails
- As the response on a validation operation to provide information about the outcome
- As part of a message response, if it hasn’t been processed correctly
- As the response to a batch/transaction, when requested
- As part of a search's Bundle response containing information about the search

Operation outcomes that are returned should match the status of the HTTP response code, and should include at least one issue object.

For example, operation outcomes that are successful would accompany an HTTP response status code of 201 for newly created resources, 204 for successful deletion, or 200 for successful `GET` requests.

In this example, I successfully deleted the 8th version of Jenny on try.smilecdr.com.

##### Success
```
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "information",
      "code": "informational",
      "diagnostics": "Successfully deleted 1 resource(s) in 24ms"
    }
  ]
}
```

##### Failure
```
 { "resourceType": "OperationOutcome",
 "issue": [
   {
     "severity": "error",
     "code": "processing",
     "diagnostics": "Unknown resource type 'Patients' - Server knows how to handle: [Appointment, Account, Invoice, CatalogEntry, EventDefinition, DocumentManifest, MessageDefinition, Goal, MedicinalProductPackaged, Endpoint, EnrollmentRequest, Consent, CapabilityStatement, Measure…]”
   }
 ]
}
```
I got this example from try.smilecdr.com by querying `Patients` instead of `Patient` when trying to find all the patients on the server.
The accompanying HTTP status codes for failing operations would be 404 for "Not Found" or 422 for "Unprocessable Entity", which is thrown when an object you post violates FHIR standards.

## History

The history interaction retrieves the history of either a resource, all resources of a given type, or all resources supported by the system. 
Create, Update, Delete actions can create history entries, and have side effects on the `meta.versionId`.
As a result of instance creation, the `response.lastModified` property will always matches `the meta.lastUpdated`.


On success, the response body will contain a JSON-encoded representation of a history bundle with the version history sorted from most recent to the oldest version. 

Errors generated by the FHIR store have a JSON-encoded OperationOutcome describing the issue for the error.

When the Bundle is too big to send in one go, it may have too resources within it or too many to send in one response, and the server will break it up into smaller pieces (called "pages"). To allow for easy navigation from one page to the next the Bundle provides links that set the context for navigating to the next page within it. In addition, there are also links to the previous, first, and last pages provided in each Bundle.

For example, if a search returns 500 resources, the server can return a bundle containing only the first 20 and a link which will return the next 20, and so on until the end etc.

![paging screenshot](paging.png)

## Batch and Transactions

Transactions are complex interactions. It's not expected that every server will implement them, and the server should be configured to do so. Servers that don’t support it will just return a HTTP 400 error, which might include an `OperationOutcome`

Transactions update, create or delete a set of resources as a single transaction.
```
  "interaction": [
    {
      "code": "history-system"
    },
    {
      "code": "transaction"
    }
  ],
```

Batch transactions perform a set of a separate interactions in a single HTTP operation.

`search-system` searches all resources based on some filter criteria.

```
  "interaction": [
    {
      "code": "history-system"
    },
    {
      "code": "transaction"
    }
  ],
```

Batch and Transaction interactions submit a set of actions to perform on a server in a single HTTP request/response.

Multiple actions on multiple resources of the same or different types may be submitted
They may be a mix of interactions like read, search, create, update, delete, etc.

#### Batch vs. Transaction: what's the difference?

##### Batch
- The actions may be performed independently as a "batch"
- Each entry is treated as if an individual interaction or operation

##### Transaction
- A single atomic "transaction" where the entire set of changes is treated as a single entity
- All interactions or operations either succeed or fail together

### How Do We Perform That?

Batch or Transaction is done with an HTTP `POST` command. The Content is a Bundle with `Bundle.type` set to "batch" or "transaction". Each Bundle entry contains the request details.
For `PUT` or `POST`, it also contains the resource.

A batch:
```
{
  "resourceType": "Bundle",
  "id": "bundle-request-simplesummary",
  "type": "batch",
  "entry": [
    {
      "request": {
"method": "GET",
"url": "Patient/8675309"
      }
    },
    {
      "request": {
		...
      }
    },

    ...

  ]
}
```

A transaction:
```
{
  "resourceType": "Bundle",
  "id": "bundle-transaction",
  "type": "transaction",
  "entry": [
    {
      "fullUrl": 
"http://example.org/fhir/Patient/123",
      "resource": {
        "resourceType": "Patient",
        "id": "123",
        "active": true,
        "name": [
          {
            "use": "official",
            "family": "Chalmers",
            "given": [
              "Peter",
              "James"
            ]
          }
        ],
        "gender": "male",
        "birthDate": "1974-12-25"
      },
      "request": {
        "method": "PUT",
        "url": "Patient/123"
      }
    },
    ...
  ]
}
```

### Batch Processing Rules

- You can't have interdependencies between the different entries in the same Bundle.
- Success or failure of one change should not alter the success or failure or resulting content of another change, and the server will validate that.
- The order of execution is the same as for Transactions...but that shouldn’t matter because you followed all the above rules like a good developer!

### Did it Work?

If the HTTP response code is `200 OK`, the batch was processed correctly regardless of the success of the operations within the Batch!
To see the status of the operations, look inside the returned Bundle.
A response code on an entry of other than `2xx` indicates that processing the request in the entry failed.

### Transaction Processing Rules

#### It’s all or Nothing
For transactions, there are only two outcomes:
- Either: Every action within succeeds. Servers accepts all actions and returns a `200 OK`, along with a response bundle
- Or: All actions are rejected. Server returns a HTTP `400` or `500` type response

### Order of Operations
Order of the entries in the Bundle doesn’t matter!
Either way, the actions are processed by the server in this order:

1. Process any DELETE interactions
1. Process any POST interactions
1. Process any PUT or PATCH interactions
1. Process any GET or HEAD interactions
1. Resolve any conditional references

#### Internal References
- A transaction may include references from one resource to another in the bundle
- When the server assigns a new id to any resource in the bundle, it also updates any references to that resource in the same bundle as they are processed
- References to resources that are not part of the bundle are left untouched

#### Conditional References
Sometimes you want to perform a set of operations involving resources that the server doesn’t know about yet.

In a `POST` request 
```
{
  "resourceType": "Patient",
  "identifier": [ { "system": 
"urn:oid:1.2.36.146.595.217.0.1", 
"value": "12345" } ],
  "name": [ {
      "family": "Iantorno",
      "given": [ "Mark", "Brent", 
"Bahlai" ]
  } ],
  "gender": "male",
  "birthDate": "1983-06-23"
}
```

And its response `201 Created`:
```
{
  "resourceType": "Patient",
  "id": "1625",
  "meta": {
    "versionId": "1",
    "lastUpdated": "2020-05-23T09:21:26.065-04:00"
  },
  "identifier": [
    {
      "system": "urn:oid:1.2.36.146.595.217.0.1",
      "value": "12345"
    }
  ],
  "name": [
    {
      "family": "Iantorno",
      "given": [
        "Mark",
        "Brent",
        "Bahlai"
      ]
    }
  ],
  "gender": "male",
  "birthDate": "1974-12-25"
}
```

If we then `GET` the same resource:
```
{
  "resourceType": "Bundle",
  "id": "8b5be330-6138-4894-a3ea-187c7acd010c",
  "meta": {
    "lastUpdated": "2020-05-23T10:06:11.622-04:00"
  },
  "type": "searchset",
  "total": 12,
  "link": [
    {
      "relation": "self",
      "url": "http://localhost:8000/Patient"
    }
  ],
  "entry": [
    {
      "fullUrl": "http://localhost:8000/Patient/1625",
      "resource": {
        "resourceType": "Patient",
        "id": "1625",
        ...
      },
      "search": {
        "mode": "match"
      }
    },
    ...
  ]
}
```

##### What is a Conditional Reference?

When a client is making a bundle, you might not know the logical id of a resource but you know identifying information.
You resolve that identifier to a logical id many ways but it requires an additional REST transaction to the commit.

##### How does it resolve?

In a transaction (and only transactions), references to resources may be replaced by a search URI that describes how to find the correct reference.
It checks all references for search URIs, and uses the search to locate matching resources.
If there are no matches, or multiple matches, the transaction fails, and an error is returned to the user.
If there is a single match, the server replaces the search URI with a reference to the matching resource.
