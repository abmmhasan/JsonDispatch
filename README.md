# JsonDispatch

![Version](https://img.shields.io/badge/version-1.0-green?style=flat)

A specification for building APIs in JSON based format for application-level communication.

## What? Why?

* **JsonDispatch** is a specification that follows/merges rules for how a JSON response should be formatted while doing communication using REST apis.
* Though there is 1/2 spec available, but after doing several APIs it turns out, every spec (currently observed) lacks something. And the biggest issue is consistency.
* For backend developer (as I'm) it is really hectic to adopt new structure/element everytime (also for "need additional data" thing)
* **Why this?**
  * Better adaptation capability
  * Covering most of the possible scenario without breaking it
  * Cross-referencing for log detection/backtrace
  * Backward compatible (never remove, only add)

## Let's explain with example

### Example

#### Success response

```json
{
    "signature": "60c1bbca-b1c8-49d0-b3ea-fe41d23290bd",
    "version": "1",
    "status": "success",
    "message": "Data prepare successful",
    "data": {
        "type": "articles",
        "source": "self",
        "attributes": {
            "category": 1,
            "sample1": [...],
            "sample2": [...],
        }
    },
    "_references": {
        "category": {
            "1": "One Production",
            "2": "Their Production",
            "3": "Our Production",
            "4": "Out of Sight"
        }
    },
    "_properties": {
        "data": {
            "type": "object",
            "name": "On the road",
            "template": "https://github.com/abmmhasan/sample-link"
        },
        "sample1": {
            "type": "array",
            "name": "Sample",
            "deprecation": "https://demo.link4"
        }
    },
    "_links": {
        "self": "https://demo.link",
        "next": "https://demo.next"
    }
}
```

#### Failed response

```json
{
    "signature": "f4b44a6e-d593-11ec-9d64-0242ac120002",
    "version": "1",
    "status": "fail",
    "message": "Invalid request",
    "data": [
        {
            "status": 403,
            "source": "/data/attributes/secretPowers",
            "title": "Secret Power issue!",
            "detail": "Editing secret powers is not authorized on Sundays."
        },
        {
            "status": 422,
            "source": "/data/attributes/volume",
            "title": "Volume issue!",
            "detail": "Volume does not, in fact, go to 11."
        }
    ],
    "_properties": {
        "data": {
            "type": "array",
            "name": "invalid-resource"
        }
    }
}
```

#### Error response

```json
{
    "signature": "c043e23a-4b26-4a05-96c4-5c60fcc18d50",
    "version": "1",
    "status": "error",
    "message": "someMessageHere",
    "code": "additionalErrorCode",
    "data": [
        {
            "status": 500,
            "source": "reputation",
            "title": "The backend responded with an error",
            "detail": "Reputation service not responding after three requests."
        }
    ],
    "_properties": {
        "data": {
            "type": "array",
            "name": "reputation-resource-error"
        }
    }
}
```

### Primary Parameters

Lets have a look at our parameter assignment per **status**:

|Type| Description| Required Keys | Optional Keys |
| :---:| :---| :---| :---|
|success| All is well! & response body returned as usual!| signature, version, status, data |message, _links, _properties, _references |
|fail| Issue with submitted data (validation,...), Pre-condition failure | signature, version, status, message|data, _properties |
|error| Failed to process request; e.g. intermediate external API issue, exceptions,... | signature, version, status, message | code, data, _properties |

Now lets get familiar by element:

| Name | Type | Use Case | Hint/Example |
| :---: | :---: | :--- |:----|
| signature | string | Provide a unique signature to every response. This will/would be used for tracing. | This signature should be generated during client request & must use in every possible log (even with storage if possible). A [standard UUID implementation](https://github.com/abmmhasan/UUID) would be nice. |
| version** | string | The version number of the structure change when it happened for the first time. Must follow [Semantic Versioning](https://semver.org/). | 2.3, 6.7, ...|
| status | string | Response status |Must be any of **success**, **fail** or **error**. Check above table for criteria.|
| message | string | A simple message explaining current response. |E.g. for Success 'Data retrieve successful' or for fail 'Data validation failure'.|
| code | string | Error code. This is not HTTP error code but a symbolic code explaining what actually happened. |E.g. EIU001 may be an error code returned for non-existing User.|
| data** | array, object, string, bool, number | Main payload. Data type must be unique per version-url-method combo. |Obviously it is what you are sending to the client.|
| _references** | object | Any additional info required for data attribute relation. |E.g. data.human.attitude have id. So under ref we should have data.human.attitude key which enlist possible value for that key as "key: value" attribute. So client can parse data as required.|
| _properties** | object | Data property |This is an object explaining all the Group type.|
| _links** | object | Provide all the required links here, which can be used for reference, pagination, additional info,... |E.g. self, next, previous, author, book, volume etc.|

** These parameters have more in-depth property type-hint. Check below!

## In-depth explanation

### `version`

* Versioning must follow [Semantic Versioning](https://semver.org/).
* The version number of the structure change when it happened for the first time. For example, our project version is 2.5, but the API structure actually changed in version 1.9; so the version number should be **1.9** here.
* As of Semantic Versioning (major.minor.patch), existing structure can be modified according to versioning method:
  * **patch** & **minor** version change means we can fix issue with existing data (like linguistics change, add new Error Code for Code key etc.). Must be backward compatible.
  * **minor** version change means we can add new keys.
  * **major** version may change structure like converting some data.attribute member type from object to array or vice-versa etc.

### `data`

Though it may hold any type of data but you must keep the type/structure same respecting version-url-method (respecting version rule as given above) combination.

* For `success` response it may hold any type of data as defined (array/object/string/bool/number). Though it's use is completely independent but if it is **object** or **array of object** then it is recommended to keep following structure in immediate child objects:

  | Name   | Type | Hint/Example |
  |:----:|:----:|:---|
  |type|string|An unique identifier for the object|
  |source|string|unique identifier for source for data in attribute e.g. self|
  |attributes|array, object|payload|
  |references|object|same as [_references](#_references) but specific to current memeber only (optional)|
  |properties|object|same as [_properties](#_properties) but specific to current memeber only (optional)|

    ```json
    "data": [
        {
            "type": "articles",
            "source": "self",
            "attributes": [
                {
                    "title": "Rails is Omakase",
                    "category": 1
                },
                {
                    "title": "Rails is Omakase2",
                    "category": 4
                },
                {
                    "title": "Rails is Omakase3",
                    "category": 1
                }
            ],
            "references": {
                "category": {
                    "1": "One Production",
                    "2": "Their Production",
                    "3": "Our Production",
                    "4": "Out of Sight"
                }
            },
            "properties": {
                "type": "array",
                "name": "c_title",
                "template":"https://some.link" ,
                "deprecation": "https://demo.link4",
                "count": 3,
                "range": "1000-1200",
                "page": 52
            }
        }
    ]
    ```

* For `fail` response it will hold array of objects explaining the reason (optional), as following:

  | Name   | Type | Hint/Example |
  |:----:|:----:|:---|
  | status | int  | Explain error as HTTP code |
  | source | string  | Path to element from root (glued by slash) where error detected (optional if parsing issue)|
  | title | string | A formal title for the reason (optional) |
  | detail | string | A detail explanation |

    ```json
    "data": [
        {
            "status": 403,
            "source": "/data/attributes/secretPowers",
            "title": "Secret Power issue!",
            "detail": "Editing secret powers is not authorized on Sundays."
        },
        {
            "status": 422,
            "source": "/data/attributes/volume",
            "title": "Volume issue!",
            "detail": "Volume does not, in fact, go to 11."
        }
    ]
    ```

* For `error` response it will hold array of objects explaining the reason (optional), as following:

  | Name   | Type |      Hint/Example       |
  |:----:|:----:| :--- |
  | status | int  | Explain error as HTTP code |
  | source | string  | Provide source of error. E.g. self or google or ... (external source)|
  | title | string | A formal title for the reason (optional) |
  | detail | string | A detail explanation |

    ```json
    "data": [
        {
            "status": 500,
            "source": "reputation",
            "title": "The backend responded with an error",
            "detail": "Reputation service not responding after three requests."
        }
    ]
    ```

### `_references`

Sometimes we may need to send some extra information regarding data parsing. & that is where it comes in. You can also think of it as **normalizer** that holds some defination/key-value resolver to parse data.

* This object will have key which client should be aware of,
* Anywhere the key name matches, will be resolved as key-value pair
* These keys will be resolved as value assigned here. E.g. We have data as given below. So any id/reference we get in attributes.category inside data object should be parsed accordingly. Like, if we get "1" we will parse as "One Production".
* It can even be a group of object / array as well under those keys depending on use case scenario.

    ```json
    "_references": {
        "category": {
            "1": "One Production",
            "2": "Their Production",
            "3": "Our Production",
            "4": "Out of Sight"
        }
    }
    ```

### `_properties`

This actually holds identity/property of groups inside data. Key-value pair is parsable as like `_references` but the value section have several things in it.

| Name   | Type |      Hint/Example       |
|:----:|:----:| :--- |
| type | string  | type of resource(array/object/string/bool/number) under that key |
| name | string  | name for that key (Should be a unique identifier for that specific response section)|
| template | url | if the resource under that key follows some specific template then link to that template (optional) |
| deprecation | url | if the resources under that key gonna deprecate (soon) & will be moved to/under a new link, provide the link here (optional) |
| count | int | if array, provide data count (optional) |
| range | string (hyphened data) | If (paginated) array provide range (optional) |
| page | int | If (paginated) array provide page index (optional) |

** You can add more memebers as you see fit

```json
"_properties": {
    "data": {
        "type": "array",
        "name": "c_title",
        "template":"https://some.link" ,
        "deprecation": "https://demo.link4",
        "count": 10,
        "range": "102-890",
        "page": 52
    }
}
```

### `_links`

Provide all the required links here, which can be used for reference, pagination, additional info, ...

* If pagination links required we can send relative paths as `self`, `next`, `previous`, `first`, `last`
* If we need to send reference e.g. author link, we can send `author`
* & much more as you see fit.

```json
"_links": {
    "self": "https://link.to",
    "next": "https://link.next"
}
```
