# Use middleware to modify proxy response body

As the `node-http-proxy` automatically responds to the request, we can't `next();` to other middlewares without modifying the module.

Our goal here is to be able to take a JSON response from the proxy, use a middleware to do some things and modify the response before sending it back to the user.

## How

We will create middlewares that will be loaded **before** the proxy. In these functions we will do some things and put a ***transformation*** function in the response, taking the original response of the proxy, and returning a new version.
Using `node-http-proxy` built-in events, we will override the ```res.write()``` function so that it will use our middleware's transformation method if there is one defined.

With this method, the proxy will set the response headers before our function, and the ```Content-Length```header will already be set with the original proxy response length. As we won't be able to modify it, we will switch to ```Transfer-Encoding: chunked``` mode so that if we have a bigger response, it won't be truncated. It's kind of dirty as this mode is used for dynamic files (streaming a video for example) but it works.

*Note: i'm not very proud of this hack but I needed a solution fast.*

## Workflow

- Create your middlewares for specific routes as always
  - We will create a ```res.overwrite``` method taking the response body as a paremeter and returning a new one
- Load them before the proxy
- Initialize the proxy with custom ```onProxyReq``` and ```onProxyRes``` events.
  - Request : we will remove the ```Accept-Encoding``` header from the request so that your proxy won't encode anything.
  - Response : we will switch to chunked mode and call our override function when proxy is answering the original request

## The fake "next" middleware example

```
var express = require('express');
var router = express.Router();

module.exports = function (app) {

  // Takes the proxy response body
  var _marshaller = function (data) {

    // Do some magic stuff here....

    // Modify the response
    var _answer = data;
    _answer.additional_data = true;
    
    // Return the new one
    return _answer;
  };

  app.get('/awesome/route', function (req, res, next) { res.overwrite = _marshaller; next(); });
}

```

## The proxy initialization

```

//
// Load your 'next' routes here (before the proxy)....
//

app.use(BLABLABLABLABLA....)

//
// Init the proxy
//

// Proxy On-Request event
var _proxyOnRequest = function(proxyReq, req, res) {
  console.log("[PROXY] Receiving new request:");

  // I have a JSON body, set the content-length and type headers if necessary...
  if (req.body) {
    var bodyData = JSON.stringify(req.body);
    proxyReq.setHeader('Content-Type','application/json');
    proxyReq.setHeader('Content-Length', Buffer.byteLength(bodyData));
    proxyReq.write(bodyData);
  }

  // Disable proxy encoding
  proxyReq.removeHeader('Accept-Encoding');
};

// Proxy On-Response event
var _proxyOnResponse = function(proxyRes, req, res) {
  console.log("[PROXY] Responding request");

  // Switch to chunked mode
  proxyRes.headers['Transfer-Encoding'] = 'chunked';

  // Save the old write function
  var _write = res.write;

  // Create a buffer (it uses the 'bufferhelper' node package)
  var buffer = new BufferHelper();

  // On data chunk received from the proxy, save it in the buffer
  proxyRes.on('data', function(data) {
    buffer.concat(data);
  });

  // Magic happens here: override the response write function
  res.write = function (data) {
    var body;
    try {
      // Try to parse the JSON response from the proxy
      body = JSON.parse(buffer.toBuffer().toString());

      // If we found the res.overwrite function (we have a 'next' middleware, use it)
      if (res.overwrite) {
        body = res.overwrite(body);
      }
    } catch (e) {
      body = buffer.toBuffer().toString();
    }

    // Transform response into buffer
    body = new Buffer(JSON.stringify(body));

    // Call original write function to answer the request
    _write.call(res, body);
  };
}

var _PROXY_MIDDLEWARE = proxy('/', { 
  target: 'http://proxy-vm:9200',
  onProxyReq: _proxyOnRequest,
  onProxyRes : _proxyOnResponse
});

app.use('/', _PROXY_MIDDLEWARE);

```