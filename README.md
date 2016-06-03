# inert

Static file and directory handlers plugin for hapi.js.

[![Build Status](https://secure.travis-ci.org/hapijs/inert.png)](http://travis-ci.org/hapijs/inert)

Lead Maintainer - [Gil Pedersen](https://github.com/kanongil)

**inert** provides new [handler](https://github.com/hapijs/hapi/blob/master/API.md#serverhandlername-method)
methods for serving static files and directories, as well as decorating the [reply](https://github.com/hapijs/hapi/blob/master/API.md#reply-interface)
interface with a `file` method for serving file based resources.

#### Features

 * Files are served with cache friendly `last-modified` and `etag` headers.
 * Generated file listings and custom indexes.
 * Precompressed file support for `content-encoding: gzip` and `content-encoding: br` responses.
 * File attachment support using `content-disposition`  header.

## Index

  - [Examples](#examples)
      - [Static file server](#static-file-server)
      - [Serving a single file](#serving-a-single-file)
      - [Customized file response](#customized-file-response)
  - [Usage](#usage)
      - [Registration options](#registration-options)
      - [`reply.file(path, [options])`](#replyfilepath-options)
      - [The `file` handler](#the-file-handler)
      - [The `directory` handler](#the-directory-handler)

## Examples

**inert** enables a number of common use cases for serving static assets.

### Static file server

The following creates a basic static file server that can be used to serve html content from the
`public` directory on port 3000:

```js
const Path = require('path');
const Hapi = require('hapi');
const Inert = require('inert');

const server = new Hapi.Server({
    connections: {
        routes: {
            files: {
                relativeTo: Path.join(__dirname, 'public')
            }
        }
    }
});
server.connection({ port: 3000 });

server.register(Inert, () => {});

server.route({
    method: 'GET',
    path: '/{param*}',
    handler: {
        directory: {
            path: '.',
            redirectToSlash: true,
            index: true
        }
    }
});

server.start((err) => {

    if (err) {
        throw err;
    }

    console.log('Server running at:', server.info.uri);
});
```

### Serving a single file

You can serve specific files using the `file` handler:

```js
server.route({
    method: 'GET',
    path: '/{path*}',
    handler: {
        file: 'page.html'
    }
});
```

### Customized file response

If you need more control, the `reply.file()` method is available to use inside handlers:

```js
server.route({
    method: 'GET',
    path: '/file',
    handler: function (request, reply) {

        let path = 'plain.txt';
        if (request.headers['x-magic'] === 'sekret') {
            path = 'awesome.png';
        }

        return reply.file(path).vary('x-magic');
    }
});

server.ext('onPostHandler', function (request, reply) {

    const response = request.response;
    if (response.isBoom &&
        response.output.statusCode === 404) {

        return reply.file('404.html').code(404);
    }

    return reply.continue();
});
```

## Usage

After registration, this plugin adds a new method to the `reply` object and exposes the `'file'`
and `'directory'` route handlers.

### Registration options

**inert** accepts the following registration options:

  - `etagsCacheMaxSize` - sets the maximum number of file etag hash values stored in the
    etags cache. Defaults to `10000`.

Note that **inert** uses the custom `'file'` `variety` to signal that the response is a static
file generated by this plugin.

### `reply.file(path, [options])`

Transmits a file from the file system. The 'Content-Type' header defaults to the matching mime
type based on filename extension.:

  - `path` - the file path.
  - `options` - optional settings:
      - `confine` - serve file relative to this directory and returns `403 Forbidden` if the
        `path` resolves outside the `confine` directory.
        Defaults to `true` which uses the `relativeTo` route option as the `confine`.
        Set to `false` to disable this security feature.
      - `filename` - an optional filename to specify if sending a 'Content-Disposition' header,
        defaults to the basename of `path`
      - `mode` - specifies whether to include the 'Content-Disposition' header with the response.
        Available values:
          - `false` - header is not included. This is the default value.
          - `'attachment'`
          - `'inline'`
      - `lookupCompressed` - if `true`, looks for the same filename with the '.gz' suffix for a
        pre-compressed version of the file to serve if the request supports content encoding.
        Defaults to `false`.
      - `etagMethod` - specifies the method used to calculate the `ETag` header response.
        Available values:
          - `'hash'` - SHA1 sum of the file contents, suitable for distributed deployments.
            Default value.
          - `'simple'` - Hex encoded size and modification date, suitable when files are stored
            on a single server.
          - `false` - Disable ETag computation.

Returns a standard [response](https://github.com/hapijs/hapi/blob/master/API.md#response-object) object.

The [response flow control rules](https://github.com/hapijs/hapi/blob/master/API.md#flow-control) **do not** apply.

### The `file` handler

Generates a static file endpoint for serving a single file. `file` can be set to:
  - a relative or absolute file path string (relative paths are resolved based on the
    route [`files`](https://github.com/hapijs/hapi/blob/master/API.md#route.config.files)
    configuration).
  - a function with the signature `function(request)` which returns the relative or absolute
    file path.
  - an object with one or more of the following options:
      - `path` - a path string or function as described above (required).
      - `confine` - serve file relative to this directory and returns `403 Forbidden` if the
        `path` resolves outside the `confine` directory.
        Defaults to `true` which uses the `relativeTo` route option as the `confine`.
        Set to `false` to disable this security feature.
      - `filename` - an optional filename to specify if sending a 'Content-Disposition'
        header, defaults to the basename of `path`
      - `mode` - specifies whether to include the 'Content-Disposition' header with the
        response. Available values:
          - `false` - header is not included. This is the default value.
          - `'attachment'`
          - `'inline'`
      - `lookupCompressed` - if `true`, looks for the same filename with the '.gz' suffix
        for a pre-compressed version of the file to serve if the request supports content
        encoding. Defaults to `false`.
      - `etagMethod` - specifies the method used to calculate the `ETag` header response.
        Available values:
          - `'hash'` - SHA1 sum of the file contents, suitable for distributed deployments.
            Default value.
          - `'simple'` - Hex encoded size and modification date, suitable when files are stored
            on a single server.
          - `false` - Disable ETag computation.

### The `directory` handler

Generates a directory endpoint for serving static content from a directory.
Routes using the directory handler must include a path parameter at the end of the path
string (e.g. `/path/to/somewhere/{param}` where the parameter name does not matter). The
path parameter can use any of the parameter options (e.g. `{param}` for one level files
only, `{param?}` for one level files or the directory root, `{param*}` for any level, or
`{param*3}` for a specific level). If additional path parameters are present, they are
ignored for the purpose of selecting the file system resource. The directory handler is an
object with the following options:
  - `path` - (required) the directory root path (relative paths are resolved based on the
    route [`files`](https://github.com/hapijs/hapi/blob/master/API.md#route.config.files)
    configuration). Value can be:
      - a single path string used as the prefix for any resources requested by appending the
        request path parameter to the provided string.
      - an array of path strings. Each path will be attempted in order until a match is
        found (by following the same process as the single path string).
      - a function with the signature `function(request)` which returns the path string or
        an array of path strings. If the function returns an error, the error is passed back
        to the client in the response.
  - `index` - optional boolean|string|string[], determines if an index file will be served
    if found in the folder when requesting a directory. The given string or strings specify
    the name(s) of the index file to look for. If `true`, looks for 'index.html'. Any falsy
    value disables index file lookup. Defaults to `true`.
  - `listing` - optional boolean, determines if directory listing is generated when a
    directory is requested without an index document.
    Defaults to `false`.
  - `showHidden` - optional boolean, determines if hidden files will be shown and served.
    Defaults to `false`.
  - `redirectToSlash` - optional boolean, determines if requests for a directory without a
    trailing slash are redirected to the same path with the missing slash. Useful for
    ensuring relative links inside the response are resolved correctly. Disabled when the
    server config `router.stripTrailingSlash` is `true. `Defaults to `false`.
  - `lookupCompressed` - optional boolean, instructs the file processor to look for the same
    filename with the '.gz' suffix for a pre-compressed version of the file to serve if the
    request supports content encoding. Defaults to `false`.
  - `etagMethod` - specifies the method used to calculate the `ETag` header response.
    Available values:
      - `'hash'` - SHA1 sum of the file contents, suitable for distributed deployments.
        Default value.
      - `'simple'` - Hex encoded size and modification date, suitable when files are stored
        on a single server.
      - `false` - Disable ETag computation.
  - `defaultExtension` - optional string, appended to file requests if the requested file is
    not found. Defaults to no extension.
