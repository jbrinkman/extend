## Handling Errors from Extensions

Successful execution of an extension typically results in a HTTP 200 response with extension-specific payload in the response body. If you receive a non-200 response, the following information will guide you in understanding the error condition. 

All responses from calling extension URLs contain the `x-wt-response-source` HTTP header. It indicates the component of the Auth0 Extend stack that generated the response. Given this information, the response status code, and the response payload, you can decide how to handle it. These are the possible values of `x-wt-response-source`: 

* **webtask**: the response was generated as a result of executing the code of the extension. You will see this value when the extension completed successfuly but also when it returned an error object through the callback function, or the code of the extension generated an uncaught exception. You can differentiate between success and failure using the status code of the response and the content of the response body. 

* **compiler**: the code of the extension could not be compiled. The most likely cause is a syntax error in the code. In those cases the body of the response contains more information about the location of the error. 

* **proxy**, **network**: these uncommon values indicate an error that occured in Auth0 Extend infrastructure before the execution of the extension code. Typically there will be more details in the body of the response. 

Below are the most common cases of responses you will see. 

### Successful Response

An extension that returns a valid response can look like this: 

```javascript
module.exports = (ctx, cb) => {
  cb(null, { valid: 'response' });
};
```

The response you will see when calling this extension is: 

```
HTTP/1.1 200 OK
Content-type: application/json
x-wt-response-source: webtask

{"valid":"response"}
```

### Application Level Error

An extension that explicitly returns an application error:  

```javascript
module.exports = (ctx, cb) => {
  cb(new Error('Some error'));
};
```

The response you will see when calling this extension is: 

```
HTTP/1.1 400 Bad Request
Content-type: application/json
x-wt-response-source: webtask

{
  "code": 400,
  "error": "Script returned an error.",
  "details": "Error: Some error",
  "name": "Error",
  "message": "Some error",
  "stack": "..."
}

```

### Application Level Error with Custom Status Code

An extension that explicitly returns an application error can override the default HTTP 400 status code with a custom status code value:

```javascript
module.exports = (ctx, cb) => {
  var error = new Error('Some error');
  error.statusCode = 401;
  cb(error);
};
```

Response: 

```
HTTP/1.1 401 Unauthorized
Content-type: application/json
x-wt-response-source: webtask

{
  "code": 401,
  "error": "Script returned an error.",
  "details": "Error: Some error",
  "name": "Error",
  "message": "Some error",
  "stack": "..."
}
```

### Syntax Error in Extension Code

An extension that cannot be compiled due to a syntax error:

```javascript
module.exports = (ctx, cb) => {
  random text
};
```

Response: 

```
HTTP/1.1 400 Bad Request
Content-type: application/json
x-wt-response-source: compiler

{
  "code": 400,
  "message": "Compilation failed: Unexpected identifier",
  "error": "Unexpected identifier",
  "stack": "SyntaxError: Unexpected identifier\n    at ...""
}
```

### Uncaught Synchronous Exception in Extension Code

An extension that generates an uncaught synchronous exception:

```javascript
module.exports = (ctx, cb) => {
  throw new Error('Some error');
};
```

Response:

```
HTTP/1.1 500 Internal Server Error
Content-type: application/json
x-wt-response-source: webtask

{
  "code": 500,
  "error": "Script generated an unhandled synchronous exception.",
  "details": "Error: Some error",
  "name": "Error",
  "message": "Some error",
  "stack": "..."
}
```

### Uncaught Asynchronous Exception in Extension Code

Uncaught asynchronous exceptions thrown by extension code require careful consideration. Such exceptions lead to the termination of the process in which the extension is running. Depending on the isolation scope you have selected, there may be other extensions executing in that process at the same time. For example, if a single tenant defined multiple extensions, and there are multiple concurrent transactions of that tenant which require execution of extensions, an uncaught exception generated by one execution will termite the execution of all other concurrently executing extensions in the same process. Given that, if you receive the "Uncaught asynchronous exception" error, it may as well indicate a problem in the code or exection of another extension. 

This is an extension that generates an uncaught asynchronous exception:

```javascript
module.exports = (ctx, cb) => {
  setTimeout(() => {
    throw new Error('Some error');
  }, 1000);
};
```

Response: 

```
HTTP/1.1 500 Internal Server Error
Content-type: application/json
x-wt-response-source: webtask

{
  "code": 500,
  "error": "Script generated an unhandled asynchronous exception.",
  "details": "Error: Some error",
  "name": "Error",
  "message": "Some error",
  "stack": "..."
}
```