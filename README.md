# Brinkbit HTTP Filesystem API
> A generalized RESTful API for managing a remote file system

# Table of Contents

1. [Overview](#overview)
1. [Resources](#resources)
1. [Request](#request)
1. [Response](#response)
1. [Actions](#actions)
  1. [Get](#get)
    1. [read](#1.1)
    1. [search](#1.2)
    1. [inspect](#1.3)
    1. [download](#1.4)
  1. [Post](#post)
    1. [create](#2.1)
    1. [bulk](#2.2)
    1. [copy](#2.3)
  1. [Put](#post)
    1. [update](#3.1)
    1. [move](#3.2)
    1. [rename](#3.3)
  1. [Delete](#delete)
    1. [delete](#4.1)

**[⬆ back to top](#table-of-contents)**

# Overview

This API is intended to be the standard for communication between the client and server for the filesystem component of the Brinkbit IDE.
While our use case is specific, we hope to develop this as a generalized implementation-agnostic standard for managing remote file systems via HTTP requests.

The http-fs-api follows the [RESTful API](http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm) systems architecture style.
What this means in practice is that all actions are mapped to four [HTTP methods](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html) (POST, GET, PUT, DELETE) which in turn represent the four [CRUD commands](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) (Create, Read, Update, Delete).
The requested url contains a uri component which determines what resource will be manipulated.

While the API is designed to handle paths, directories, and files as though the backend were an actual file system, it's important to note that your backend implementation does not need to be an actual filesystem.
In fact, we recommend that you not store the manipulated resources on an actual filesystem but use some other form of data storage i.e. a database or S3 Buckets.

All examples contained in this document are written as HTTP requests, with the fictitious `http://animals.com` as the domain.
In all the examples, `pets/` will be the route on which the file system API is defined.
The structure of the file system is as follows:

TODO: replace with graphic.

+ pets/
  + cats/
    + siamese.jpg
    + cat_adjectives.txt
    + kindof_pretty_cat.png
    + very_pretty_cat.gif
    + cat_names.txt
    + more_pretty_cats/
  + dogs/
    + golden_retriever.bmp
    + loyal_companion.png


**[⬆ back to top](#table-of-contents)**

# Resources

Our filesystem has two types of resources, files and directories.
Both can be manipulated with the same actions, but each behave a little differently.

Files are discrete, independent resources, containing metadata specifically to itself and their own data.
Files must live either at the top level of the file system, or within a directory.

Directories are containers for files and other directories, containing metadata for itself, as well as aggregated data concerning it's contents. They are denoted by a trailing slash, following their name, e.g., "more_pretty_cats/"

**[⬆ back to top](#table-of-contents)**

# Request

Requests are made on a uri with one of the following four [HTTP methods](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html): "GET", "POST", "PUT", "DELETE".

A uri represents the resource to take action on.
For example the uri `/cats/siamese.jpg`

A JSON object MUST be at the root of every JSON API request containing data.
This object defines a document's "top level".
Every request containing a JSON object MUST have a `data` field at the top level

```javascript
{
  "data": {
    //...
  }
}
```

A request MAY specify an `action` string field within the `data` field.
If no action is specified, then the associated default behavior occurs.
Each `action` field has a set of accepted parameters, and the API MUST return a 400 status if a mismatch occurs.
Note that each method's default has it's own alias, for consistency.

```javascript
{
  "data": {
    "action": "move",
    //...
  }
}
```

The request MAY have a `parameters` field with the `data` field, which contains specific parameters for the given `action`. If there is no `action` field, the entire `parameters` field is ignored.

```javascript
{
  "data": {
    "action": "move",
    "parameters": {
      "destination": "dogs/",
      //...
    }
  }
}
```

The `parameters` field MAY also contain a `flags` field,
which is an array of strings representing aliased flags which alter the specified action.
The current list of flags are:

`force` / `f`: force operation

`recursive` / `r`: recursive

`unique` / `u`: force uniqueness.

Finishing out the example, to move the `siamese.jpg` resource from the `/cats/` directory to the `/dogs/` directory, overwriting any resource there that shares the name, you would send the following request:

```http
PUT /cats/siamese.jpg HTTP/1.1
Content-Type: application/fs.api+json
Accept: application/fs.api+json

{
  "data": {
    "action": "move",
    "parameters": {
      "destination": "dogs/",
      "flags": [ "force" ]
    },
  }
}
```

**[⬆ back to top](#table-of-contents)**

# Response

Similar to the request syntax, most responses SHOULD have a JSON object at the root of the response when it contains data, which defines the response's "top level".
The only exceptions to this are the successful `read` and `download` responses, which MUST only contain the resource's data.
Every other response MUST have a `data` field at the top level on success, or an `errors` field on failure.
It MUST NOT have both.
The root level response MAY contain a metadata field that contains relevant or more descriptive data.
The root level of a JSON response MUST NOT contain any fields besides `data`, `error`, and `metadata`.

A successful response will have all relevant data inside of a top-level `data` array, e.g.:
```javascript
{
  "data": [
    "file1",
    "file2",
    "folder/"
  ]
}
```

An error response WILL have a JSON object at the root, and WILL have an `errors` field at the top level.
The `errors` field isan array of at least one JSON object containing only a  `status` field and `message` field. The `status` field should be a numerical status code representing the error condition, and the 'message' field should be a human-readable string giving additional information. E.g.:

```javascript
{
  "errors": [
    {
      "status": 404,
      "message": "File not found."
    }
  ]
}
```

**[⬆ back to top](#table-of-contents)**

## Actions

`action` is an optional parameter under `data` which gives direction for what operation will take place on the specified resource, or set of resources.
Each action has an associated [HTTP method](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html), and only the actions associated with that HTTP method are accepted.
If a mismatch between HTTP method and action occurs, the API MUST return a 400 status.
The list of actions per method are as follows:

GET: [`read`](#1.1), [`search`](#1.2), [`inspect`](#1.3), [`download`](#1.4)

POST: [`create`](#2.1), [`bulk`](#2.2), [`copy`](#2.3)

PUT: [`update`](#3.1), [`move`](#3.2), [`rename`](#3.3)

DELETE: [`delete`](#4.1)

### Get

- [1.1](#1.1) <a name='1.1'></a> **read** *default*
  > Request file or directory contents.
    Default GET action

  Parameters
  > *none*

  Flags
  > *none*

  Request file:
  ```http
  GET /cats/siamese.jpg HTTP/1.1
  Accept: application/vnd.api+json
  ```

  Response:

  `Raw image data`

  Request directory:
  ```http
  GET /cats/ HTTP/1.1
  Accept: application/vnd.api+json
  ```

  Response:
  ```javascript
  {
    "data": [
      "siamese.jpg",
      "kindof_pretty_cat.png",
      "very_pretty_cat.gif",
      "morePrettyCats/"
    ]
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist


- [1.2](#1.2) <a name='1.2'></a> **search**
  > Run a query on the requested directory, and returns an array of matching resources.

  Parameters
  + `query` `string` -
    an implementation-specific query syntax supported by the API, that defines on which fields to search, and the search to be made.
  + `sort` `string` *optional* -
    an implementation-specific sorting syntax supported by the API, defining which fields to sort on.
    If omitted, results will be returned in the manner determined by the query string.

  Flags
  + `recursive` - Deep search. Defaults to shallow.

  The implementation MUST support querying the following fields:
  + `name`
  + `parent`

  The implementation MAY support querying the following fields:
  + `type`
  + `size`
  + `dateCreated`
  + `lastModified`
  +  other custom fields defined by the implementation

  Request:
  ```http
  GET /cats/
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "search",
      "parameters": {
        "query": "-name \"*pretty*\""
      }
    }
  }
  ```

  Response:
  ```javascript
  {
    "data": [
      "/cats/kindof_pretty_cat.png",
      "/cats/more_pretty_cats/"
    ]
  }
  ```

  Errors
  + `404` - Invalid path
  + `501` - Invalid action / Action-Resource type conflict


- [1.3](#1.3) <a name='1.3'></a> **inspect**
  > Request detailed information about a resource.
  By default, all metadata is returned; specifying fields will return only those fields.

  Parameters
  + `fields` `array` *optional* -
    an array of strings for each requested fields.

    The following fields MUST be supported:
    + `name` `string` - the name of the resource
    + `parent` `string` - the parent directory

    The following fields MAY be supported:
    + `type` `string` - either `'file'` or `'directory'`
    + `size` `number` - number of bytes
    + `dateCreated` `number` - unix timestamp when the resource was created
    + `lastModified` `number` - unix timestamp when the resource was last modified

  Flags
  > *none*

  Request:
  ```http
  GET /cats/kindof_pretty_cat.png
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "inspect",
      "parameters": {
        "fields": ["type", "size", "name", "parent", "dateCreated", "lastModified"]
      }
    }
  }
  ```

  Response:
  ```javascript
  {
    "data": {
      "type": "file",
      "size": 1234567890,
      "name": "very_pretty_cat.png",
      "parent": "cats/",
      "dateCreated": 1439765335,
      "lastModified": 1439765353
    }
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist


- [1.4](#1.4) <a name='1.4'></a> **download**
  > Zip and download requested resource

  Parameters
  + `compression` `string` -
    Specify the type of compression to be used. Defaults to `zip`

  The following compression formats MUST be supported:
  + Zip

  Flags
  > *none*

  Request:
  ```http
  GET /cats/kindof_pretty_cat.png
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "download"
    }
  }
  ```

  Response:

  ```Zipped file contents```

  Errors
  + `404` - Invalid path / Resource does not exist

**[⬆ back to top](#table-of-contents)**

### Post

- [2.1](#2.1) <a name='2.1'></a> **create** *default*
  > Create a resource with optional initial data.
    Default POST action.

  Parameters
  + `content` `any format` -
    the data to store in the resource

  Flags
  + `force` - overwrites existing resource

  Request:
  ```http
  POST /cats/new_cat_picture.jpg
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "create",
      "parameters": {
        "content": `raw image data`
      }
    }
  }
  ```  

  Response:

  ```HTTP Status 200```

  Errors
  + `409` - Invalid path / Resource already exists
  + `415` - Invalid file type
  + `413` - Request data too large
  + `501` - Invalid action / Action-Resource type conflict


- [2.2](#2.2) <a name='2.2'></a> **bulk**
  > Create multiple resources with optional initial data, in a specified directory.

  Parameters
  + `resources` `json` -
    Hash of names and content of resources to be uploaded.
    `null` specifies no initial content.

  Flags
  + `force` - overwrites existing resource

  Request:
  ```http
  POST /cats/
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "create",
      "parameters": {
        resources: {
          "another_cat_picture": `raw image data`,
          "the_best_cats/": null
        }
      }
    }
  }
  ```  

  Response:

  ```HTTP Status 200```

  Errors
  + `409` - Invalid path / Resource already exists
  + `415` - Invalid file type
  + `413` - Request data too large
  + `501` - Invalid action / Action-Resource type conflict  


- [2.3](#2.3) <a name='2.3'></a> **copy**
  > Copy a resource

  Parameters
  + `destination` `string` - the full path to where the copy should be created.
  Defaults to the specified resource's directory.

  Flags
  + `unique` - renames the resource in the typical "_1" format if there is a naming conflict.

  Request:
  ```http
  POST /cats/fluffy.jpg
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "copy",
      "parameters": {
        "destination": './'
      }
    }
  }
  ```

  Response:
  ``` javascript
  {
    "data": "cats/siamese_1.jpg."
  }
  ```

  Response:
  ```http Status 200```

  Errors
  + `404` - Invalid path / Resource does not exist
  + `409` - Invalid destination / Resource already exists
  + `501` - Invalid action / Action-Resource type conflict


**[⬆ back to top](#table-of-contents)**

### Put

- [3.1](#3.1) <a name='3.1'></a> **update** *default*
  > Modify an existing resource.
    Default PUT action

  Parameters
  + `contents` `any format` -
    the data to store in the resource

  Flags
  > none

  Request:
  ```http
  PUT /cats/cat_adjectives.txt

  Accept: application/vnd.api+json

  {
    "data": {
      "parameters": {
        "data": "Fluffy, Furry, Fuzzy"

      }
    }
  }
  ```

  Response:

  ```HTTP Status 200```


  Errors
  + `404` - Invalid path / Resource does not exist
  + `413` - Request data too large
  + `501` - Invalid action / Action-Resource type conflict



- [3.2](#3.2) <a name='3.2'></a> **move**
  > Relocate an existing resource

  Parameters
  + `destination` `string` - the path to which the resource will be moved

  Flags
  > none

  Request:
  ```http
  PUT /cats/cat_names.txt
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "move",
      "parameters": {
        "destination": "cats/more_pretty_cats/"
      }
    }
  }
  ```

  Response:

  ```HTTP Status 200```


  Errors
  + `404` - Invalid path / Resource does not exist
  + `409` - Invalid destination / Resource already exists
  + `501` - Invalid action / Action-Resource type conflict



- [3.3](#3.3) <a name='3.3'></a> **rename**
  > Rename an existing resource

  Parameters
  + `name` `string` - the new name to give the resource

  Flags
  > none

  Request:
  ```http
  PUT /cats/more_pretty_cats/cat_names.txt
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "rename",
      "parameters": {
        "name": "pretty_cat_names.txt"
      }
    }
  }
  ```

  Response:

  ```HTTP Status 200```  

  Errors
  + `404` - Invalid path / Resource does not exist
  + `409` - Invalid destination / Resource already exists
  + `501` - Invalid action / Action-Resource type conflict

**[⬆ back to top](#table-of-contents)**

### Delete

- [4.1](#4.1) <a name='4.1'></a> **delete** *default*
  > Destroy an existing resource

  Parameters
  > none

  Flags
  + `force` - Removes resource and all child resources.


  Request:
  ```http
  DELETE /cats/more_pretty_cats/pretty_cat_names.txt
  Accept: application/vnd.api+json
  ```

  Response:

  ```HTTP Status 200```

  Errors
  + `404` - Invalid path / Resource does not exist

**[⬆ back to top](#table-of-contents)**

# TODOs

- document global error responses
- mention resources mapping to URIs
- mention json data type
- outline what an action is (i.e. Open, Save, Copy)
- outline how to contribute
