# http-fs-api
> A generalized RESTful API for managing a remote file system.

# Overview

This API is intended to be the standard for communication between the client and server for the filesystem component of the Brinkbit IDE. While our use case is specific, we hope to develop this as a generalized implementation-agnostic standard for managing remote file systems via HTTP requests.

The http-fs-api follows the [RESTful API](https://en.wikipedia.org/wiki/Representational_state_transfer) systems architecture style. What this means in practice is that all actions are mapped to four [HTTP methods](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html) (POST, GET, PUT, DELETE) which in turn represent the four [CRUD commands](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) (Create, Read, Update, Delete). The url requested determines what resource will be manipulated.

While the API is designed to handle paths, directories, and files as though the backend were an actual file system, it's important to note that your backend implementation does not need to be an actual filesystem. In fact, we recommend that you not store the manipulated resources on an actual filesystem but use some other form of data storage i.e. a database or S3 Buckets.

All examples contained in this document are written as [ajax requests](http://api.jquery.com/jquery.ajax/) and use the fictitious `http://cats.com` as the domain. In all the examples, `/fs/` will be the route on which the file system API is defined.

# Resources

Our filesystem has two types of resources, files and directories. Both can be manipulated with the same actions, but each behave a little differently.

# Request

Requests are made on a uri with one of the following four [HTTP methods](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html): "GET", "POST", "PUT", "DELETE".

A uri represents the resource to take action on. For example the uri `/img.png`

Every request accepts an optional JSON object as a request parameter.

Every request can optionally specify an action field in the request parameters. For example:

```javascript
$.ajax({
  url: 'http://cats.com/fs/Siamese.img'
  method: 'PUT',
  data: {
    action: 'Move',
    destination: 'Siberian.img'
  }
})
```

# Response

# Actions

#### GET
- Open (default)
- Search

#### POST
- New (default)
- Copy

#### PUT
- Save (default)
- Move

#### DELETE
- Delete (default)



TODO: arrange following TODOs into a hierarchal readable outline

TODO: mention files and directories as the two different resource types

TODO: mention resources mapping to URIs

TODO: mention json data type

TODO: describe HTTP commands

TODO: outline response format

TODO: outline request format

TODO: outline what an action is (i.e. Open, Save, Copy)

TODO: outline the different actions and their HTTP command mappings

TODO: outline how to contribute
