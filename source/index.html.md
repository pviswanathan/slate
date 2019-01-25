---
title: Swift Partner API Specifications

language_tabs: # must be one of https://git.io/vQNgJ
  - shell: cURL
  - json: JSON
  - http: HTTP

search: true
---

# Introduction

The Swift Medical API follows a RESTful design pattern and provides resource endpoints to interact with our data. The API uses HTTP response codes to indicate the status (success or error) of requests being made, and all reponses are returned in JSON. The API also follows strict security protocol and supports 2-legged and 3-legged authorization methods (via OAuth2).

This doc will specifically point to the relevant API's assisting in the integration of the following categories:

1. [Authorization](#authorization)
2. [ADT](#adt)
3. [Assessments (Wound/Non-Wound)](#assessments)
4. [Images](#images)
5. [Visits](#visits)
6. [Assessment Scheduling](#assessment-scheduling)
7. [Product Catalog](#product-catalog)
8. [Clinical Orders](#clinical-orders)
9. [MDS](#mds)

## Base URL
Unless otherwise specified, the base URL is `https://api.swiftmedical.io/private/api/v2`

All requests must be made via HTTPS. HTTP is not supported. The API supports only the latest TLS ciphers for strong security, for more information see TLS Ciphers in the Appendix.

## Media Types
The API supports XML and JSON-API.

JSON API is hypermedia protocol allowing developers to find the exact location of the related resource(s)

The response type of a request can be set by setting the accepts header or by appending .json or .xml to end of a request such as /api/v2/patients/123.json

All responses support gzip, and all text fields support UTF-8.

## Pagination
All request that can potentially return a multiple results support pagination. The following attributes are supported for every such endpoint.

Attribute | Description |
--------- | ----------- |
`limit` | Limits the number of returned results, if not specified the default is `20`
`after` | Returns results after the object of specified ID.
`since` | Returns objects updated since `version_id` or `timestamp`.
`sort_by` | Returns results ordered by the specified key.
`order` | Whether to return the results by the sort key ascneding or descending.

## Errors
### Response Codes
The API uses the following status codes for errors:

Status | Description | Explanation |
-------|-------------|-------------|
`401` | Unauthorized | Returned when the current user cannot perform the action such as logging in.
`403` | Forbidden | Returned when the current user is not allowed to access to requested resource.
`404` | Not Found | Returned when the requested resource cannot be found.
`422` | Unprocessable Entity | Returned when the request cannot be processed due to client error. Typically this is returned when request violates `bus`iness logic.
`500` | Internal Server Error | Returned when the API is unable to process the request due to a platform error.

### Response Body
The API uses JSON-API format for returning errors. An example of an error would be as follows:

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/vnd.api+json

{
  "errors": [
    {
      "source": { "pointer": "/data/attributes/first-name" },
      "title": "Invalid Attribute",
      "detail": "First name must contain at least three characters."
    },
    {
      "source": { "pointer": "/data/attributes/first-name" },
      "title": "Invalid Attribute",
      "detail": "First name cannot be blank"
    }
  ]
}
```


# Authorization
The Swift API implements JSON Web Tokens (JWT) for authentication and you can pass the token to the API in the HTTP Authorization Header using  `Bearer`.

## Getting a new access token
Please use the following example as a guide to get a new access token:

```
curl -k -i https://api.swiftmedical.io/private/api/v2/oauth/token \
-F grant_type="client_credentials" \
-F scope="{scopes}" \
-u {client_id}:{client_secret}
```

> The response is as follows:

```
HTTP/1.1 200 OK

{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJ1c2VyX2lkIjoiNzhjZGVjNzgtZjNmNy00N2MwLTg2YTgtMzljN2Y2MjZiY2I2IiwiY3JlYXRlZF9hdCI6IjIwMTctMDQtMTBUMDM6MTY6NDMuNzg1WiIsImV4cGlyZXNfaW4iOjcyMDB9.EpvTYLRMBRWN5ie3_dyTswquDuTI6YUrGLKsFD6jKaktqSKkVfoq3_kE_LPwldIx9vHV6UIwUiitdGcqMDHHfw","token_type":"bearer","expires_in":7200,"refresh_token":"cfba0c8bc2c7fd2883afb6c6a27994020beacb7f400b00b99d1a5519a2270d73","scope":"read","created_at":1491794203}
```

## Refreshing
Upon getting a new access token, the response provides a refresh token which can be used to get a new access token. Please use the following example as a guide to refresh a token:

```
curl -k -i https://api.swiftmedical.io/private/api/v2/oauth/token \
-F grant_type="refresh_token" \
-F refresh_token="{refresh_token}"
```

> The response is as follows:

```
HTTP/1.1 200 OK

{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJ1c2VyX2lkIjoiNzhjZGVjNzgtZjNmNy00N2MwLTg2YTgtMzljN2Y2MjZiY2I2IiwiY3JlYXRlZF9hdCI6IjIwMTctMDQtMTBUMDM6MTY6NDMuNzg1WiIsImV4cGlyZXNfaW4iOjcyMDB9.EpvTYLRMBRWN5ie3_dyTswquDuTI6YUrGLKsFD6jKaktqSKkVfoq3_kE_LPwldIx9vHV6UIwUiitdGcqMDHHfw","token_type":"bearer","expires_in":7200,"refresh_token":"cfba0c8bc2c7fd2883afb6c6a27994020beacb7f400b00b99d1a5519a2270d73","scope":"read","created_at":1491794203}
```

## Making an Authorized Request
Please use the following example as a guide to perform an authorized request:

```
curl -k -i -H "Authorization: Bearer {access_token}" \
-H "Content-Type: application/json" \
-H "X-Client-ID": {client_id}
https://api.swiftmedical.io/private/api/v2/images/{image_id}?include_measurements={include_measurements}&format={format}
```

# ADT
The API supports the standard [HL7](https://corepointhealth.com/resource-center/hl7-resources/hl7-adt/) ADT (Admission Discharge Transfer) format for synchronizing patient statuses.

The Platform supports the following ADT Event Types:

Message ID | Name | Description |
-----------|------|-------------|
`A01` | Admission Admit a new patient.
`A08` | Readmission Admit a patient that was previously discharged.
`A02` | Transfer  Transfer an admitted patient.
`A03` | Discharge Discharge an admitted patient.
`A21` | Temporary Leave Temporarily discharge a patient.
`A11` | Cancel Admit  correct a previous admission message.
`A13` | Cancel Discharge  Correct a previous discharge message.
`A52` | Cancel Leave  Correct a previous `Leave of Absence` message.
`A53` | Cancel Return Correct a previous `Return From Leave` message.

<aside class="notice">
The API also supports historical events. Messages that come logically before other messages will be inserted into the event stream and dependent data will be automatically re-computed. This allows you to correct reports in a reliable and audible way.</aside>

<aside class="notice">Swift can also consume ADT via the Swift HL7 interface. Please contact integrations@swiftmedical.com to learn more.</aside>

## Submitting an ADT Message
ADT messages can be posted to the API in the following way:


> POST https://api.swiftmedical.io/private/api/v2/adt

```json
{
  "ADTMessageType": {
    "header": {
      "messageId": 0,
      "messageMode": "P",
      "messageCode": "ADT",
      "triggerEvent": "A01",
      "eventDateTime": "2015-12-21T09:35:20.000-05:00",
      "receivingApplication": 123,
      "receivingFacility": 123,
      "sendingApplication": 123,
      "sendingFacility": 123,
      "facID": 8,
      "orgID": 37512001,
      "createdDate": "2015-12-21T09:35:20.458-05:00",
      "facilityCode": "1AZ31",
      "operatorID": "daiglj",
      "operatorLongName": "Jonathan Daigle",
      "facilityNPI": 1619058120
    },
    "residentInformation": {
      "demographicInformation": {
        "residentName": {
          "namePrefix": "Dr.",
          "firstName": "Catherin",
          "lastName": "Ali"
        },
        "residentContactInformation": {
          "address1": "433 Waldgrave Building",
          "city": "Phoenix",
          "country": "United States",
          "county": "Maricopa",
          "postalZipCode": 85004,
          "provState": "AZ",
          "homePhone": "(506) 471-5555"
        },
        "residentIdentifiersList": {
          "MRNumber": "ALPCODE7",
          "ssnSin": "653-48-0964",
          "medicareNumber": "555779980Q",
          "clientID": 444319,
          "clientNumber": 3134887,
          "healthCardNumber": "ALPCODE7"
        },
        "sex": "F",
        "language": "English",
        "maritalStatus": "Widowed",
        "dateOfBirth": "1926-01-31T00:00:00.000-05:00"
      },
      "stayInformation": {
        "residentClass": "I",
        "admissionType": "3 - Elective",
        "admitDateTime": "2015-12-21T09:35:20.000-05:00",
        "admitSource": "1 - Physician Referral",
        "residentLocation": {
          "bed": "B",
          "floor": 1,
          "room": 8117,
          "unit": "Monarch Via",
          "unitID": 1192
        },
        "quarterlyReviewCycle": 1,
        "attendingPhysicianInformation": {
          "FirstName": "Mara",
          "LastName": "Greathouse",
          "NPINumber": 1609102110,
          "businessPhone": "(602) 555-8775",
          "description": "UNSPECIFIED SYSTOLIC (CONGESTIVE) HEART FAILURE"
        },
        "actionCode": "U",
        "reportedDateTime": "2015-01-23T00:00:00.000-05:00",
        "resolvedDateTime": "2015-12-08T00:00:00.000-05:00",
        "rankOrder": 2,
        "therapy": false
      }
    }
  }
}
```

# Assessments
The API supports various assessments such as BRADEN, PURS, Waterlow, Norton, Wound as well as custom assessments. Internally, assessments are immutable allowing us to easily audit changes, rollback mistakes, and support offline use.

<aside class="notice">Assessments created/updated in the Swift system will be pushed to the partner as per the partner's API specifications.</aside>

## CREATE
For assessments created in the Partner system, they may be pushed to Swift using this method.

> POST https://api.swiftmedical.io/private/api/v2/assessments

```json
{
  "assessmentId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "userId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "patientId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "resourceId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "resourceType": "Study",
  "answersJson": {
    "nutrition": "veryPoor",
    "sensory": "completelyLimited",
    "moisture": "constantlyMoist",
    "friction": "problem",
    "mobility": "completelyImmobile",
    "activity": "bedfast",
    "header": {
      "title": "Braden Assessment",
      "version": "1",
      "type": "braden",
      "$schema": "https://api.swiftmedical.io/private/v2/assessments/86ce6a41-6a1b-4162-ae06-41a7b51fe1a7/schema#"
    }
  },
  "studyId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "score": "18",
  "completed": true,
  "frequency": "null",
  "frequencyValue": "0",
  "dueDateTime": "2015-01-23T00:00:00.000-05:00",
  "createdDateTime": "2015-01-23T00:00:00.000-05:00",
  "updatedDateTime": "2015-01-23T00:00:00.000-05:00"
}
```

## UPDATE
For assessments updated in the Partner system, they may be synced to Swift using this method.

> PUT https://api.swiftmedical.io/private/api/v2/assessments/{assessment_id}

```json
{
  "assessmentId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "userId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "patientId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "resourceId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "resourceType": "Study",
  "answersJson": {
    "nutrition": "veryPoor",
    "sensory": "completelyLimited",
    "moisture": "constantlyMoist",
    "friction": "problem",
    "mobility": "completelyImmobile",
    "activity": "bedfast",
    "header": {
      "title": "Braden Assessment",
      "version": "1",
      "type": "braden",
      "$schema": "https://api.swiftmedical.io/private/v2/assessments/86ce6a41-6a1b-4162-ae06-41a7b51fe1a7/schema#"
    }
  },
  "studyId": "9c24ecd0-ac11-4f67-9b91-a2cb6c45dd4b",
  "score": "18",
  "completed": true,
  "frequency": "null",
  "frequencyValue": "0",
  "dueDateTime": "2015-01-23T00:00:00.000-05:00",
  "createdDateTime": "2015-01-23T00:00:00.000-05:00",
  "updatedDateTime": "2015-01-23T00:00:00.000-05:00"
}
```

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`assessment_id` | `uuid` | Represents the Swift Assessment ID

# Images
One of the core components of the Swift platform is advanced imaging. Within the mobile application, users are able to capture high-resolution wound images, which help to tell a story of how the wound progresses over time.

<aside class="notice">Images created in the Swift system will be pushed to the partner as per the partner's API specifications.</aside>

## GET
> GET https://api.swiftmedical.io/private/api/v2/images/{image_id}?include_measurements={include_measurements}&format={format}

> This returns a redirect to the image:

```http
HTTP/1.1 302 Found
Location: https://swiftmedical-images.s3.amazonaws.com/ObjectName?AWSAccessKeyId=xxxx&Expires=xxxxx&Signature=xxxxxxx
```

Used to fetch an image

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`image_id` | `uuid` | Represents the Swift Image ID

### Query Params

Parameter | Data Type | Default | Description |
--------- | --------- | --------|-------------|
`include_measurements` | `boolean` | `false` | True if you want an overlay of the measurements, false otherwise
`format` | `string` | `medium` | The image size, other possible values include `original` and `thumbnail`


# Visits
In outpatient settings, patient visit scheduling is a fundamental component of the workflow. This API supports scheduling patients for upcoming visits. The Swift mobile application is able to leverage this information to render the most relevant patient list to the user to stream-line the mobile workflow.

<aside class="notice">Swift can also consume encounters via the Swift HL7 interface (SIU). Please contact integrations@swiftmedical.com to learn more.</aside>

## CREATE
Used to create a patient visit. This endpoint is based off of the HL7 [SIU-S12](https://corepointhealth.com/resource-center/hl7-resources/hl7-siu-message/) specification.

> POST https://api.swiftmedical.io/private/api/v2/patients/{patient_id}/visits

```json
{
  "HL7MessageType" : {
    "id" : null,
    "partnerId" : "1d4f62ea-a0cd-4107-a78b-c6074ab765ea",
    "messageType" : "SIU",
    "header" : {
      "messageId" : "1496176905888",
      "eventDateTime" : "",
      "receivingApplication" : "WOUND360",
      "receivingFacility" : "WOUND360",
      "sendingApplication" : "IHEAL",
      "sendingFacility" : "HEALOGICS",
      "facID" : "4187",
      "orgID" : 0,
      "triggerEvent" : "S12",
      "createdDate" : null,
      "facilityCode" : "",
      "facilityNPI" : null
    },
    "patient" : {
      "id" : "2809383",
      "schedule" : {
        "id" : null,
        "appointmentTiming" : {
          "startDateTime" : "20170530210000",
          "duration" : "75",
          "durationUnits" : "minutes",
          "startDateTimeOffset" : "-0500"
        }
      },
      "visit" : {
        "id" : "56425078",
        "class" : "outpatient"
      }
    },
    "residentInformation" : {
      "demographicInformation" : {
        "residentName" : {
          "namePrefix" : null,
          "firstName" : "Swift",
          "lastName" : "Test"
        },
        "residentContactInformation" : {
          "address1" : null,
          "city" : null,
          "country" : null,
          "county" : null,
          "postalZipCode" : null,
          "provState" : null,
          "homePhone" : null
        },
        "residentIdentifiersList" : {
          "MRNumber" : "979864",
          "ssnSin" : null,
          "medicareNumber" : null,
          "clientID" : "2809383",
          "clientNumber" : null,
          "healthCardNumber" : null
        },
        "religion" : null,
        "sex" : "M",
        "language" : null,
        "maritalStatus" : null,
        "dateOfBirth" : "19450131"
      }
    },
    "stayInformation" : {
      "dischargeDateTime" : "",
      "admitDateTime" : "20170530210000",
      "residentLocation" : {
        "bed" : "",
        "floor" : "",
        "floorId" : null,
        "room" : "",
        "unit" : null,
        "description" : "",
        "unitID" : ""
      },
      "attendingPhysicianInformation" : {
        "FirstName" : null,
        "LastName" : null,
        "NPINumber" : null,
        "businessPhone" : null,
        "description" : null
      }
    }
  }
}
```

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`patient_id` | `uuid` | Represents the Swift Patient ID

## UPDATE
Used to update an existing patient visit. This endpoint mimics the HL7 [SIU-S14/SIU-S15](https://corepointhealth.com/resource-center/hl7-resources/hl7-siu-message/) specification.

> PUT https://api.swiftmedical.io/private/api/v2/patients/{patient_id}/visits/{visit_id}

```json
{
  "HL7MessageType" : {
    "id" : null,
    "partnerId" : "1d4f62ea-a0cd-4107-a78b-c6074ab765ea",
    "messageType" : "SIU",
    "header" : {
      "messageId" : "1496176905888",
      "eventDateTime" : "",
      "receivingApplication" : "WOUND360",
      "receivingFacility" : "WOUND360",
      "sendingApplication" : "IHEAL",
      "sendingFacility" : "HEALOGICS",
      "facID" : "4187",
      "orgID" : 0,
      "triggerEvent" : "S14",
      "createdDate" : null,
      "facilityCode" : "",
      "facilityNPI" : null
    },
    "patient" : {
      "id" : "2809383",
      "schedule" : {
        "id" : null,
        "appointmentTiming" : {
          "startDateTime" : "20170530210000",
          "duration" : "75",
          "durationUnits" : "minutes",
          "startDateTimeOffset" : "-0500"
        }
      },
      "visit" : {
        "id" : "56425078",
        "class" : "outpatient"
      }
    },
    "residentInformation" : {
      "demographicInformation" : {
        "residentName" : {
          "namePrefix" : null,
          "firstName" : "Swift",
          "lastName" : "Test"
        },
        "residentContactInformation" : {
          "address1" : null,
          "city" : null,
          "country" : null,
          "county" : null,
          "postalZipCode" : null,
          "provState" : null,
          "homePhone" : null
        },
        "residentIdentifiersList" : {
          "MRNumber" : "979864",
          "ssnSin" : null,
          "medicareNumber" : null,
          "clientID" : "2809383",
          "clientNumber" : null,
          "healthCardNumber" : null
        },
        "religion" : null,
        "sex" : "M",
        "language" : null,
        "maritalStatus" : null,
        "dateOfBirth" : "19450131"
      }
    },
    "stayInformation" : {
      "dischargeDateTime" : "",
      "admitDateTime" : "20170530210000",
      "residentLocation" : {
        "bed" : "",
        "floor" : "",
        "floorId" : null,
        "room" : "",
        "unit" : null,
        "description" : "",
        "unitID" : ""
      },
      "attendingPhysicianInformation" : {
        "FirstName" : null,
        "LastName" : null,
        "NPINumber" : null,
        "businessPhone" : null,
        "description" : null
      }
    }
  }
}
```

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`patient_id` | `string` | Represents the Partner Patient ID
`visit_id` | `string` | Represents the Partner Visit ID

# Assessment Scheduling
Commonly used in the inpatient setting, assessment scheduling is used to ensure that patients' wounds are assessed at a regular frequency to ensure gold standard care. This API allows Partners to push an assessment schedule to Swift, which in turn allows the Swift mobile application to present an optimized practioner workflow to address scheduled assessments as a priority.

## CREATE
Used to create an assessment schedule.

> POST https://api.swiftmedical.io/private/api/v2/assessments/{assessment_id}/visits

```json
{
  "patientId": "112233",
  "scheduleId": "100", 
  "assessmentId": "23453"
  "assessDueDate": "2015-01-23T00:00:00.000-05:00"
}
```

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`assessment_id` | `uuid` | Represents the Swift Assessment ID

## UPDATE
Used to update an assessment schedule.

> PUT https://api.swiftmedical.io/private/api/v2/assessments/{assessment_id}/schedules/{schedule_id}

```json
{
  "assessDueDate": "2015-01-23T00:00:00.000-05:00"
}
```

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`assessment_id` | `uuid` | Represents the Swift Assessment ID
`schedule_id` | `string` | Represents the Partner Schedule ID

# Product Catalog
Product selection is configurable section of the Swift mobile workflow. If the partner so desires, they may push product (formulary) information to Swift so that this list can be offered for selection to users performing the mobile workflow.

<aside class="notice">Swift can also consume product catalogs directly from the partner via CSV.</aside>

## CREATE
Used to create a product listing.

> POST https://api.swiftmedical.io/private/api/v2/product/

```json
{
  "id" : "00326925",
  "din" : null,
  "sku" : "320-DYND30261",
  "brand" : "Medline",
  "name" : "Biohazard Lab Specimen Bag",
  "imageURL" : "",
  "descriptionHTML" : "<ul><li>Unique two-pouch design isolates patient specimen documents, helping to reduce possibilities for contamination or accidental separation of specimen and paperwork</li><li>Zip-style closure</li></ul>",
  "externalDescriptionURL" : "",
  "quantity" : "1000",
  "dimensions" : "6x9"
}
```

## UPDATE
Used to update an existing product listing.

> PUT https://api.swiftmedical.io/private/api/v2/product/{product_id}

```json
{
  "id" : "00326925",
  "din" : null,
  "sku" : "320-DYND30261",
  "brand" : "Medline",
  "name" : "Biohazard Lab Specimen Bag",
  "imageURL" : "",
  "descriptionHTML" : "<ul><li>Unique two-pouch design isolates patient specimen documents, helping to reduce possibilities for contamination or accidental separation of specimen and paperwork</li><li>Zip-style closure</li></ul>",
  "externalDescriptionURL" : "",
  "quantity" : "1000",
  "dimensions" : "6x9"
}
```

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`product_id` | `string` | Represents the Partner Product ID

# Clinical Orders
Swift offers a Clinical Orders API to allow partners to push Clinical Orders to Swift. This data is used to generate alerts and facilitate a stream-lined workflow within the Swift mobile application.

<aside class="notice">Swift can also consume Clinical Orders via the Swift HL7 interface (ORM). Please contact integrations@swiftmedical.com to learn more.</aside>

## CREATE
Used to create a clinical order.

> POST https://api.swiftmedical.io/private/api/v2/patients/{patient_id}/clinical-orders/

```json
{
  "HL7MessageType" : {
    "id" : null,
    "partnerId" : "1d4f62ea-a0cd-4107-a78b-c6074ab765ea",
    "messageType" : "ORM",
    "header" : {
      "messageId" : "1496176905888",
      "eventDateTime" : "",
      "receivingApplication" : "WOUND360",
      "receivingFacility" : "WOUND360",
      "sendingApplication" : "IHEAL",
      "sendingFacility" : "HEALOGICS",
      "facID" : "4187",
      "orgID" : 0,
      "triggerEvent" : "S14",
      "createdDate" : null,
      "facilityCode" : "",
      "facilityNPI" : null
    },
    "patient" : {
      "id" : "2809383",
      "schedule" : {
        "id" : null,
        "appointmentTiming" : {
          "startDateTime" : "20170530210000",
          "duration" : "75",
          "durationUnits" : "minutes",
          "startDateTimeOffset" : "-0500"
        }
      },
      "visit" : {
        "id" : "56425078",
        "class" : "outpatient"
      }
    },
    "residentInformation" : {
      "demographicInformation" : {
        "residentName" : {
          "namePrefix" : null,
          "firstName" : "Swift",
          "lastName" : "Test"
        },
        "residentContactInformation" : {
          "address1" : null,
          "city" : null,
          "country" : null,
          "county" : null,
          "postalZipCode" : null,
          "provState" : null,
          "homePhone" : null
        },
        "residentIdentifiersList" : {
          "MRNumber" : "979864",
          "ssnSin" : null,
          "medicareNumber" : null,
          "clientID" : "2809383",
          "clientNumber" : null,
          "healthCardNumber" : null
        },
        "religion" : null,
        "sex" : "M",
        "language" : null,
        "maritalStatus" : null,
        "dateOfBirth" : "19450131"
      }
    },
    "stayInformation" : {
      "dischargeDateTime" : "",
      "admitDateTime" : "20170530210000",
      "residentLocation" : {
        "bed" : "",
        "floor" : "",
        "floorId" : null,
        "room" : "",
        "unit" : null,
        "description" : "",
        "unitID" : ""
      },
      "attendingPhysicianInformation" : {
        "FirstName" : null,
        "LastName" : null,
        "NPINumber" : null,
        "businessPhone" : null,
        "description" : null
      }
    }
    "order" : {
      "universalServiceIdentifier" : {
        "id" : "003038",
        "text" : "Urinalysis",
        "nameOfCodingSystem" : "L"
      }
      "quantityTiming" : {
        "quantity" : "1"
      }
      "orderDateTime" : "20170530210000"
    }
  }
}
```

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`patient_id` | `string` | Represents the Partner Patient ID

## UPDATE
Used to update an existing clinical order.

> PUT https://api.swiftmedical.io/private/api/v2/patients/{patient_id}/clinical-orders/{clinical_order_id}

```json
{
  "HL7MessageType" : {
    "id" : null,
    "partnerId" : "1d4f62ea-a0cd-4107-a78b-c6074ab765ea",
    "messageType" : "ORM",
    "header" : {
      "messageId" : "1496176905888",
      "eventDateTime" : "",
      "receivingApplication" : "WOUND360",
      "receivingFacility" : "WOUND360",
      "sendingApplication" : "IHEAL",
      "sendingFacility" : "HEALOGICS",
      "facID" : "4187",
      "orgID" : 0,
      "triggerEvent" : "S14",
      "createdDate" : null,
      "facilityCode" : "",
      "facilityNPI" : null
    },
    "patient" : {
      "id" : "2809383"
    },
    "residentInformation" : {
      "demographicInformation" : {
        "residentName" : {
          "namePrefix" : null,
          "firstName" : "Swift",
          "lastName" : "Test"
        },
        "residentContactInformation" : {
          "address1" : null,
          "city" : null,
          "country" : null,
          "county" : null,
          "postalZipCode" : null,
          "provState" : null,
          "homePhone" : null
        },
        "residentIdentifiersList" : {
          "MRNumber" : "979864",
          "ssnSin" : null,
          "medicareNumber" : null,
          "clientID" : "2809383",
          "clientNumber" : null,
          "healthCardNumber" : null
        },
        "religion" : null,
        "sex" : "M",
        "language" : null,
        "maritalStatus" : null,
        "dateOfBirth" : "19450131"
      }
    },
    "stayInformation" : {
      "dischargeDateTime" : "",
      "admitDateTime" : "20170530210000",
      "residentLocation" : {
        "bed" : "",
        "floor" : "",
        "floorId" : null,
        "room" : "",
        "unit" : null,
        "description" : "",
        "unitID" : ""
      },
      "attendingPhysicianInformation" : {
        "FirstName" : null,
        "LastName" : null,
        "NPINumber" : null,
        "businessPhone" : null,
        "description" : null
      }
    }
    "order" : {
      "universalServiceIdentifier" : {
        "id" : "003038",
        "text" : "Urinalysis",
        "nameOfCodingSystem" : "L"
      }
      "quantityTiming" : {
        "quantity" : "1"
      }
      "orderDateTime" : "20170530210000"
    }
  }
}
```

### Path Params

Parameter | Data Type | Description |
--------- | ----------|-------------|
`patient_id` | `string` | Represents the Partner Patient ID
`clinical_order_id` | `string` | Represents the Partner Clinical Order ID


<aside class="notice">Swift can also consume clinical orders via the Swift HL7 interface. Please contact integrations@swiftmedical.com to learn more.</aside>

# MDS
The API supports MDS (Minimal Data Set) versions 2.0 and 3.0.

## GET MDS Listing

> GET https://api.swiftmedical.io/api/private/v2/mds/{version}/responses?patient_id={patient_id}&ard={ard}&a0310a={a0310a}&a0310b={a0310b}&a0310c={a0310c}&a0310f={a0310f}&isc={isc}&primary_reason={primary_reason}

This returns the following:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "orgCode": "canOrg",
  "facId": "1",
  "patientId": 1,
  "ard": "2015-11-09",
  "firstName": "Shad",
  "lastName": "Darden",
  "birthDate": "1925-05-01",
  "mdsResponse": {
    "questionResponseList": [
      {
        "mdsQuestionKey": "D2a",
        "responseValue": "1"
      }
    ]
  }
}
```

### Path Params

Parameter | Data Type | Example | Description |
--------- | ----------|---------|-------------| 
`version` | `number` | `2.0` | 2.0 (represents MDSver. 2.0, Canada only) 3.0 (represents MDSver. 3.0, U.S. only). Any other value will trigger a 404 (Not Found).

### Query Params

Parameter | Data Type | Example | Description |
--------- | ----------|---------|-------------|
`patient_id` | `uuid` | `8ddc8ca5-980f-4191-82cb-e1721e31122b` | Represents the Swift Patient ID
`ard` | `string` | `"2016-01-01"` | The Assessment Reference Date Assessment Reference Date in yyyy-MM-dd format
`a0310a` | `string` | `"01"` | Federal OBRA reason for assessment (sent in a MDS 3.0 request only). Assessment code is sent representing the type of assessment requested.
`a0310b` | `string` | `"99"` | PPS reason for assessment (sent in a MDS 3.0 request only). Assessment code is sent representing the type of assessment requested.
`a0310c` | `string` | `"0"` | PPS other Medicare required assessment (sent in a MDS 3.0 request only). Assessment code is sent representing the type of assessment requested.
`a0310f` | `string` | `"99"` | Entry/Discharge reporting (sent in a MDS 3.0 request only). Assessment code is sent representing the type of assessment requested.
`isc` | `string` | `"NC"` | The MDS 3.0 Item Subset Code (ISC for MDS 3.0 requests only).
`primary_reason` | `string` | `"02"` | MDS question key AA8a reason for assessment (for MDS 2.0 only).