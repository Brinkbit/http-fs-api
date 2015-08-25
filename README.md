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
    1. [Copy](#2.2)
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

All examples contained in this document are written as [ajax requests](http://api.jquery.com/jquery.ajax/) and use the fictitious `http://cats.com` as the domain.

**[⬆ back to top](#table-of-contents)**

# Resources

Our filesystem has two types of resources, files and directories.
Both can be manipulated with the same actions, but each behave a little differently.

TODO: expand explanation

**[⬆ back to top](#table-of-contents)**

# Request

Requests are made on a uri with one of the following four [HTTP methods](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html): "GET", "POST", "PUT", "DELETE".

A uri represents the resource to take action on.
For example the uri `/img.png`

A JSON object MUST be at the root of every JSON API request and response containing data. This object defines a document's "top level". Every request MUST have a `data` field at the top level.

A request MAY specify an `action` field within the `data` field, and it MAY have an associated `parameters` field.

```http
PUT /animals/cats/Siamese.img HTTP/1.1
Content-Type: application/fs.api+json
Accept: application/fs.api+json

{
  "data": {
    "action": "Move",
    "parameters": {
      "destination": "/animals/"
    }
  }
}
```
**[⬆ back to top](#table-of-contents)**

# Response

Every response WILL have a JSON object at the root of the response when it contains data. The object defines the response's "top level". Every response MUST have a `data` field at the top level. The response will have either a `data` or an `errors` field, but NEVER both. There is an optional `metadata` field that would contain relevant or more descriptive data.

**[⬆ back to top](#table-of-contents)**

## Actions

`Action` is an optional parameter under `data` which gives direction for what operation will take place on the specified resource, or set of resources. Each action has an associated [HTTP method](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html), and only the actions associated with that HTTP method are accepted. If a mismatch between HTTP method and action occurs, the API MUST return a 400 status. The list of actions per method are as follows:

GET: `default`, `help`, `search`, `inspect`, 'download'

POST: `default`, `copy`

PUT: `default`, `move`, `rename`

DELETE: `default`

### Get

- [1.1](#1.1) <a name='1.1'></a> **Read** *default*
  > Request file or directory contents.
    Default GET action

  Parameters
  > *none*

  File:
  ```http
  GET /Siamese.img HTTP/1.1
  Accept: application/vnd.api+json
  ```

  Directory:
  ```http
  GET /Siamese.img HTTP/1.1
  Accept: application/vnd.api+json
  ```

  Response:
  ```javascript
  {
    status: 200,
    data: [
      'kindof_pretty_cat.png',
      'very_pretty_cat.png',
      'morePrettyCats/' // a directory, signified by the trailing /
    ]
  }
  ```
  Errors
  + `404` - Invalid path / Resource does not exist


- [1.2](#1.2) <a name='1.2'></a> **Help**
  > Request detailed information for a given HTTP method

  Parameters
  + `method` `string` *optional* -
    an http method i.e. Get, Put, Post, or Delete

  Returns a raw text help message
  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/Siamese.img',
    data: {
      action: 'Help',
      parameters: {
        method: 'Get',
        action: 'Help'
      }
    }
  });

  // response
  {
    code: 200,
    data: "Returns file and directory contents. Default GET action.\nParameters: none\n"
  }
  ```

  Errors
  + `404` - Invalid path / Resource does not exist


- [1.3](#1.3) <a name='1.3'></a> **Search**
  > Run a query on the requested resource.
    Note, only valid on directories

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
  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/',
    data: {
      action: 'Search',
      query: '/cat/gi'
    }
  });

  // response
  {
    code: 200,
    data: [
      'prettyCats/',
      'prettyCats/kindof_pretty_cat.png',
      'prettyCats/very_pretty_cat.png',
      'prettyCats/morePrettyCats/',
      'ugly/ugly_cat.jpg',
      'catcatcat.jpg'
    ]
  }
  ```
  Request only directories:
  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/',
    data: {
      action: 'Search',
      query: '/cat/gi',
      type: 'directory'
    }
  });

  // response
  {
    code: 200,
    data: [
      'prettyCats/',
      'prettyCats/morePrettyCats/',
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
  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/prettyCats/very_pretty_cat.png',
    data: {
      action: 'Inspect',
      fields: ['type', 'size', 'name', 'parent', 'dateCreated', 'lastModified']
    }
  });

  // response
  {
    code: 200,
    data: {
      type: 'file',
      size: 1234567890,
      name: 'very_pretty_cat.png',
      parent: 'prettyCats/',
      dateCreated: 1439765335,
      lastModified: 1439765353
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
    ```javascript
    // request
    $.ajax({
      url: 'http://cats.com/fs/prettyCats/very_pretty_cat.png',
      data: {
        action: 'Download'
      }
    });

    // returns zip of the file
    ```

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

  ```javascript
  // request
  $.ajax({
    url: 'http://cats.com/fs/mycats/Fluffy.img',
    method: 'POST',
    data: formdata,
    processData: false,
    contentType: false
  });

  // response
  {
    code: 200
  }
  ```

  Errors
  + `409` - Invalid path / Resource already exists
  + `415` - Invalid file type
  + `413` - Request data too large


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

- [4.1](#4.1) <a name='4.1'></a> **Delete**
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
