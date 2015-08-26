# Brinkbit HTTP Filesystem API
> A generalized RESTful API for managing a remote file system

# Table of Contents

1. [Overview](#overview)
1. [Resources](#resources)
1. [Request](#request)
1. [Response](#response)
1. [Actions](#actions)
  1. [Get](#get)
    1. [Read](#1.1)
    1. [Help](#1.2)
    1. [Search](#1.3)
    1. [Inspect](#1.4)
    1. [Download](#1.5)
  1. [Post](#post)
    1. [Create](#2.1)
    1. [Bulkupload](#2.2)
    1. [Copy](#2.3)
  1. [Put](#post)
    1. [Update](#3.1)
    1. [Move](#3.2)
    1. [Rename](#3.3)
  1. [Delete](#delete)
    1. [Delete](#4.1)

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
The structure of the file system is as follows:

TODO: replace with graphic.

+ animals/
  + cats/
    + siamese.jpg
    + kindof_pretty_cat.png
    + very_pretty_cat.gif
    + more_pretty_cats/
  + dogs/
    + golden_retriever.bmp
    + loyal_companion.png


**[⬆ back to top](#table-of-contents)**

# Resources

Our filesystem has two types of resources, files and directories.
Both can be manipulated with the same actions, but each behave a little differently.

TODO: expand explanation

**[⬆ back to top](#table-of-contents)**

# Request

Requests are made on a uri with one of the following four [HTTP methods](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html): "GET", "POST", "PUT", "DELETE".

A uri represents the resource to take action on.
For example the uri `/cats/siamese.jpg`

A JSON object MUST be at the root of every JSON API request containing data.
This object defines a document's "top level".
Every request containing a JSON object MUST have a `data` field at the top level.

A request MAY specify an `action` field within the `data` field, and it MAY have an associated `parameters` field. If no action is specified, then the associated default behavior occurs. Each `action` field has a set of accepted parameters, and the API MUST return a 400 status if a mismatch occurs.
Note that each method's default has it's own alias, for consistency.

Following our example file system, to move the `siamese.jpg` resource from the `/cats/` directory to the `/dogs/` directory, you would send the following request:

```http
PUT /cats/siamese.jpg HTTP/1.1
Content-Type: application/fs.api+json
Accept: application/fs.api+json

{
  "data": {
    "action": "Move",
    "parameters": {
      "destination": "/dogs/"
    }
  }
}
```
**[⬆ back to top](#table-of-contents)**

# Response

Similar to the request syntax, every response WILL have a JSON object at the root of the response when it contains data, which defines the response's "top level".
The only exceptions to this are the `read` and `download` GET requests, which contain ONLY the resource's data.
Every other response MUST have a `data` field at the top level on success, or an `errors` field on failure, but NEVER both.
There is an optional `metadata` field that would contain relevant or more descriptive data.

An error response contains an array of at least one error object, and will take the following format:

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

`Action` is an optional parameter under `data` which gives direction for what operation will take place on the specified resource, or set of resources.
Each action has an associated [HTTP method](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html), and only the actions associated with that HTTP method are accepted.
If a mismatch between HTTP method and action occurs, the API MUST return a 400 status.
The list of actions per method are as follows:

GET: [`Read`](#1.1), [`Help`](#1.2), [`Search`](#1.3), [`Inspect`](#1.4), [`Download`](#1.5)

POST: [`Create`](#2.1), [`Bulkupload`](#2.2), [`Copy`](#2.3)

PUT: [`Update`](#3.1), [`Move`](#3.2), [`Rename`](#3.3)

DELETE: [`Delete`](#4.1)

### Get

- [1.1](#1.1) <a name='1.1'></a> **Read** *default*
  > Request file or directory contents.
    Default GET action

  Parameters
  > *none*

  File:
  ```http
  GET /cats/siamese.jpg HTTP/1.1
  Accept: application/vnd.api+json
  ```

  Response:

  `Raw image data`

  Directory:
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
      "morePrettyCats/" // a directory, signified by the trailing /
    ]
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist


- [1.2](#1.2) <a name='1.2'></a> **Help**
  > Request detailed information for a given HTTP method and action

  Parameters
  + `method` `string` -
    an http method i.e. Get, Put, Post, or Delete
  + `action` `string` - *optional*
    one of the associated actions to the specified method.

  Returns a raw text help message
  ```http
  GET /cats/siamese.jpg HTTP/1.1
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "Help",
      "parameters": {
        "method": "Get",
        "action": "Help"
      }
    }
  }
  ```

  Response
  ```javascript
  {
    data: {
      message: "Returns file and directory contents. Default GET action.\nParameters: none\n"
    }
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist


- [1.3](#1.3) <a name='1.3'></a> **Search**
  > Run a query on the requested resource.
    Note, only valid on directories

TODO: redo this section

  Parameters
  + `query` `regex` -
    a regular expression to run against the specified fields on the requested resource
  + `sort` `[{ field: string, descending: boolean }]`
    *default: { field: 'name' }* -
    either an object or an array of objects.
    The fields on which to sort and their direction from highest to lowest priority.
    - `field` - the field on which to sort.
      Valid fields:
      - `'type'` - either 'file' or 'directory'
      - `'size'` - number of bytes
      - `'name'` - the name of the resource
      - `'dateCreated'` - unix timestamp when the resource was created
      - `'lastModified'` - unix timestamp when the resource was last modified
    - `descending` *default: true* - if the sort should descend
  + `fields` `[string]` *default: ['name']* -
    an array of strings.
    The fields against which the query will be run.
    Valid fields:
    - `'type'` - either 'file' or 'directory'
    - `'size'` - number of bytes
    - `'name'` - the name of the resource
    - `'dateCreated'` - unix timestamp when the resource was created
    - `'lastModified'` - unix timestamp when the resource was last modified
    - `'contents'` - the contents of the file
  + `type` `string` *default: 'all'* -
    filter by resource type, either 'directory', 'file', or 'all'
  + `recursive` `boolean` *default false* -
    if true, will deep search, otherwise shallow

Returns an array of matching resources:
```http
GET /cats/
Accept: application/vnd.api+json

{
  "data": {
    "action": "Search",
    "parameters": {
      "query": "/pretty/gi"
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

  Request only directories:
  ```http
  GET /cats/
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "Search",
      "parameters": {
        "query": "/pretty/gi",
        "type": "directory"
      }
    }
  }
  ```

Response:
```javascript
{
  "data": [
    "/cats/more_pretty_cats/"
  ]
}
```

  Errors
  + `404` - Invalid path / Resource does not exist
  + `501` - Invalid action / Action-Resource type conflict


- [1.4](#1.4) <a name='1.4'></a> **Inspect**
  > Request detailed information about a resource

  Parameters
  + `fields` `array` -
    an array of strings for each requested fields.
    Valid fields:
    - `type` `string` - either `'file'` or `'directory'`
    - `size` `number` - number of bytes
    - `name` `string` - the name of the resource
    - `parent` `string` - the parent directory
    - `dateCreated` `number` - unix timestamp when the resource was created
    - `lastModified` `number` - unix timestamp when the resource was last modified

  Example:
  ```http
  GET /cats/kindof_pretty_cat.png
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "Inspect",
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


  - [1.5](#1.5) <a name='1.5'></a> **Download**
    > Zip and download requested resource

    Parameters
    > none

    Example:
    ```http
    GET /cats/kindof_pretty_cat.png
    Accept: application/vnd.api+json

    {
      "data": {
        "action": "Download"
      }
    }
    ```

    Response:

    ```Zipped file contents```

    Errors
    + `404` - Invalid path / Resource does not exist

**[⬆ back to top](#table-of-contents)**

### Post

- [2.1](#2.1) <a name='2.1'></a> **Create** *default*
  > Create a resource with optional initial data.
    Default POST action.

  Parameters 
  + `data` `FormData` -
    the initial data to store in the resource

  ```http
  POST /cats/new_cat_picture.img
  Accept: application/vnd.api+json

  {
    "data": {
      "action": "Download",
      "parameters": {
        "data": formdata,
        "processData": false,
        "contentType": false
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


  TODO: "bulkupload"


- [2.2](#2.2) <a name='2.2'></a> **Copy**
  > Copy a resource

  Parameters
  + `destination` `string` - *default: the resource's full identifier*
    the full path where the copy should be created
  + `makeUnique` `boolean` *default: true* -
    if true, will auto-rename to unique value

  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/mycats/Fluffy.img',
    method: 'POST',
    data: {
      action: 'Copy'
    }
  });

  // response
  {
    code: 200,
    data: 'http://cats.com/fs/mycats/Fluffy_1.img'
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist
  + `409` - Invalid destination / Resource already exists


**[⬆ back to top](#table-of-contents)**

### Put

- [3.1](#3.1) <a name='3.1'></a> **Update** *default*
  > Modify an existing resource.
    Default PUT action

  Parameters
  + `data` `FormData` -
    the data to store in the resource

  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/mycats/cat_names.txt',
    method: 'PUT',
    data: {
      data: 'Fluffy, Furry, Fuzzy'
    }
  });

  // response
  {
    code: 200
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist
  + `413` - Request data too large



- [3.2](#3.2) <a name='3.2'></a> **Move**
  > Relocate an existing resource

  Parameters
  + `destination` `string` -
    the path to which the resource will be moved

  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/mycats/cat_names.txt',
    method: 'PUT',
    data: {
      action: 'Move',
      destination: 'http://cats.com/fs/ourcats/cat_names.txt'
    }
  });

  // response
  {
    code: 200
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist
  + `409` - Invalid destination / Resource already exists

- [3.3](#3.3) <a name='3.3'></a> **Rename**
  > Rename an existing resource

  Parameters
  + `name` `string` - the new name to give the resource

  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/mycats/cat_names.txt',
    method: 'PUT',
    data: {
      action: 'Rename',
      destination: 'animal_names.txt'
    }
  });

  // response
  {
    code: 200
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist
  + `409` - Invalid destination / Resource already exists

**[⬆ back to top](#table-of-contents)**

### Delete

- [4.1](#4.1) <a name='4.1'></a> **Delete** *default*
  > Destroy an existing resource

  Parameters
  > none

  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/mycats/cat_names.txt',
    method: 'DELETE'
  });

  // response
  {
    code: 200
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist



**[⬆ back to top](#table-of-contents)**

# TODOs

- document global error responses
- mention resources mapping to URIs
- mention json data type
- outline response format
- outline request format
- outline what an action is (i.e. Open, Save, Copy)
- outline how to contribute
