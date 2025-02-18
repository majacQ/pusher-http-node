# Pusher Node.js REST library

In order to use this library, you need to have an account on <http://pusher.com>. After registering, you will need the application credentials for your app.

## Installation

You need to be running Node.js 0.8+ to use this library.

```
$ npm install pusher
```

### Parse Cloud

In order to use the library in a Parse Cloud module, install the module into your `cloud/modules` path:

    $ npm install pusher --prefix cloud/modules

Then to build the file bundle for Parse Cloud:

    $ cd cloud/modules/pusher
    $ npm run parse-build

To import `Pusher`:

```js
var Pusher = require('cloud/modules/node_modules/pusher/parse');
```

## Configuration

There are 3 ways to configure the client. First one is just using the Pusher constructor:

```javascript
var Pusher = require('pusher');

var pusher = new Pusher({
  appId: 'APP_ID',
  key: 'APP_KEY',
  secret: 'SECRET_KEY',
  encrypted: ENCRYPTED, // optional, defaults to false
  host: 'HOST', // optional, defaults to api.pusherapp.com
  port: PORT, // optional, defaults to 80 for unencrypted and 443 for encrypted
  cluster: 'CLUSTER', // optional, if `host` is present, it will override the `cluster` option.
});
```

For specific clusters, you can use the `forCluster` function:

```javascript
var Pusher = require('pusher');

var pusher = Pusher.forCluster("CLUSTER", {
  appId: 'APP_ID',
  key: 'APP_KEY',
  secret: 'SECRET_KEY',
  encrypted: ENCRYPTED, // optional, defaults to false
  port: PORT, // optional, defaults to 80 for unencrypted and 443 for encrypted
});
```

You can also specify auth and endpoint options by passing an URL:

```javascript
var Pusher = require('pusher');

var pusher = Pusher.forURL("SCHEME://APP_KEY:SECRET_KEY@HOST:PORT/apps/APP_ID");
```

You can pass the optional second argument with options, as in `forCluster` function.

This is useful for example on Heroku, which sets the PUSHER_URL environment
variable to such URL, if you have the Pusher addon installed.

#### Additional options

There are a few additional options that can be used in all above methods:

```javascript
var Pusher = require('pusher');

var pusher = new Pusher({
  // you can set other options in any of the 3 ways described above
  proxy: 'HTTP_PROXY_URL', // optional, URL to proxy the requests through
  timeout: TIMEOUT, // optional, timeout for all requests in milliseconds
  keepAlive: KEEP_ALIVE // optional, enables keep-alive, defaults to false
});
```

## Usage

### Callbacks and error handling

#### API requests

Asynchronous methods on the Pusher class (`trigger`, `get` and `post`) take an optional callback as the last argument. After performing the request, the callback is called with three arguments:

- error - if the request can't be performed or returns an error code, error will contain details, otherwise it will be null
- request - the request object
- response - the response object - can be undefined if response was not received

All operational errors are wrapped into a Pusher.RequestError object.

#### WebHooks

In case accessing data for invalid WebHooks, an Pusher.WebHookError exception will be thrown from the called method. It is recommended to validate the WebHook before interpreting it.

### Publishing events

To send an event to one or more channels use the trigger function. Channel names can contain only characters which are alphanumeric, '_' or '-' and have to be at most 200 characters long. Event name can be at most 200 characters long too.

#### Single channel

```javascript
pusher.trigger('channel-1', 'test_event', { message: "hello world" });
```

#### Multiple channels

```javascript
pusher.trigger([ 'channel-1', 'channel-2' ], 'test_event', { message: "hello world" });
```

You can trigger an event to at most 10 channels at once. Passing more than 10 channels will cause an exception to be thrown.

### Excluding event recipients

In order to avoid the client that triggered the event from also receiving it, the `trigger` function takes an optional `socketId` parameter. For more informaiton see: <http://pusher.com/docs/publisher_api_guide/publisher_excluding_recipients>.

```javascript
var socketId = '1302.1081607';
pusher.trigger(channel, event, data, socketId);
```

### Authenticating private channels

To authorise your users to access private channels on Pusher, you can use the `authenticate` function:

```javascript
var auth = pusher.authenticate(socketId, channel);
```

For more information see: <http://pusher.com/docs/authenticating_users>

### Authenticating presence channels

Using presence channels is similar to private channels, but you can specify extra data to identify that particular user:

```javascript
var channelData = {
	user_id: 'unique_user_id',
	user_info: {
	  name: 'Phil Leggetter'
	  twitter_id: '@leggetter'
	}
};
var auth = pusher.authenticate(socketId, channel, channelData);
```

The `auth` is then returned to the caller as JSON.

For more information see: <http://pusher.com/docs/authenticating_users>

### Application state

It's possible to query the state of the application using the `pusher.get` function.

```javascript
pusher.get({ path: path, params: params }, callback);
```

The `path` property identifies the resource that the request should be made to and the `params` property should be a map of additional query string key and value pairs.

Params can't include following keys:
- auth_key
- auth_timestamp
- auth_version
- auth_signature
- body_md5

The following example provides the signature of the callback and an example of parsing the result:
```javascript
pusher.get({ path: '/channels', params: {} }, function(error, request, response) {
	if (response.statusCode === 200) {
		var result = JSON.parse(response.body);
		var channelsInfo = result.channels;
	}
});
```

#### Get the list of channels in an application

```javascript
pusher.get({ path: '/channels', params: params }, callback);
```

Information on the optional `params` and the structure of the returned JSON is defined in the [REST API reference](http://pusher.com/docs/rest_api#method-get-channels).

#### Get the state of a channel

```javascript
pusher.get({ path: '/channels/[channel_name]', params: params }, callback);
```

Information on the optional `params` option property and the structure of the returned JSON is defined in the [REST API reference](http://pusher.com/docs/rest_api#method-get-channel).

#### Get the list of users in a presence channel

```javascript
pusher.get({ path: '/channels/[channel_name]/users' }, callback);
```

The `channel_name` in the path must be a [presence channel](http://pusher.com/docs/presence). The structure of the returned JSON is defined in the [REST API reference](http://pusher.com/docs/rest_api#method-get-users).

### WebHooks

The library provides a simple helper for WebHooks, which can be accessed via Pusher instances:

```javascript
var webhook = pusher.webhook(request);
```

Requests must expose following fields:
- headers - object with request headers indexed by lowercase header names
- rawBody - string with the WebHook request body

Since neither Node.js nor express provide the body in the request, your application needs to read it and assign to the request object. See examples/webhook_endpoint.js for a simple webhook endpoint implementation using the express framework.

Headers object must contain following headers:
- x-pusher-key - application key, sent by Pusher
- x-pusher-signature - WebHook signature, generated by Pusher
- content-type - must be set to application/json, what Pusher does

After instantiating the WebHook object, you can use its following methods:

#### isValid

Validates the content type, body format and signature of the WebHook and returns a boolean. Your application should validate incoming webhooks, otherwise they could be faked.

Accepts an optional parameter containing additional application tokens (useful e.g. during migrations):

```javascript
var webhook = pusher.webhook(request);
// will check only the key and secret assigned to the pusher object:
webhook.isValid();
// will also check two additional tokens:
webhook.isValid([{ key: "x1", secret: "y1" }, { key: "x2", secret: "y2" }]);
```

#### getData

Returns the parsed WebHook body. Throws a Pusher.WebHookError if the WebHook is invalid, so please check the `isValid` result before accessing the data.

```javascript
// will return an object with the WebHook data
webhook.getData();
```

Please read [the WebHooks documentation](http://pusher.com/docs/webhooks) to find out what fields are included in the body.

#### getEvents

Returns events included in the WebHook as an array. Throws a Pusher.WebHookError if the WebHook is invalid, so please check the `isValid` result before accessing the events.

```javascript
// will return an array with the events
webhook.getEvents();
```

#### getTime

Returns the Date object for the time when the WebHook was sent from Pusher. Throws a Pusher.WebHookError if the WebHook is invalid, so please check the `isValid` result before accessing the time.

```javascript
// will return a Date object
webhook.getTime();
```

### Generating REST API signatures

If you wanted to send the REST API requests manually (e.g. using a different HTTP client), you can use the `createSignedQueryString` method to generate the whole request query string that includes the auth keys and your parameters.

```javascript
pusher.createSignedQueryString(options);
```

The only argument needs must be an object with following keys:
- method - the HTTP request method
- body - optional, the HTTP request body
- params - optional, the object representing the query params

Query parameters can't contain following keys, as they are used to sign the request:

- auth_key
- auth_timestamp
- auth_version
- auth_signature
- body_md5

## Testing

The tests run using [Mocha](http://visionmedia.github.io/mocha/). Make sure
you've got all required modules installed:

    npm install

### Running the local test suite

You can run local integration tests without setting up a Pusher app:

    node_modules/.bin/mocha tests/integration/**/*.js

### Running the complete test suite

In order to run the full test suite, first you need a Pusher app. When starting
mocha, you need to set the PUSHER_URL environment variable to contain your
app credentials, like following:

    `PUSHER_URL='http://KEY:SECRET@api.pusherapp.com/apps/APP_ID' node_modules/.bin/mocha $(find tests)`

## Credits

This library is based on the work of:
* Christian Bäuerlein and his library pusher.
* Jaewoong Kim and the node-pusher library.

## License

This code is free to use under the terms of the MIT license.
