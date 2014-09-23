To give you a full example of how to use the Swagger Tools APIs and its middleware, let's create a simple API that will
allow you to get the current weather for a location or zip code.  We will build this API iteratively starting with the
Swagger document _(This example will use Swagger 2.0)_ and then we will move toward the implementation.  The hope is
that you will see how much simpler your implementation can be thanks to the middleware Swagger Tools provides.

## Create Your Project

This process is standard fare for Node.js development but to be thorough, let's do it anyways.  First we need to create
our project directory using and change directories to it using this: `mkdir weather && cd $_`.  Now that you are in your
project's root directory, run `npm init` and fill out however you'd like.  Now that we've got our project skeleton,
let's continue.

## Designing Your API

The first step in our project is to define our API.  The plan is to figure out how we want our API to work and then we
will take that knowledge and put it into a [Swagger][swagger] document for Swagger Tools and others to use.  _(For those
of you not familiar with Swagger, Swagger is an open format for describing RESTful APIs.)_

### The API (In Words)

The simple API we'll be designing is a weather API that uses the [MSN Weather][msn-weather].  _(MSN Weather was chosen
because it was free and didn't require you to go through the process of signing up for an account and generating an API
key.)_  Our API will take the following request arguments:

* **location:** `string` A search location like a city, zip code, etc.  _(Anything that works on MSN Weather)_
* **unit:** `string=F` Dictates whether or not you want to retrieve the results in Fahrenheit `F` or Celsius `C`.

Based on this request, the API will look up the current weather for the location and return a response in the requested
unit.

### The API (In Swagger)

To describe this API as a Swagger document, we would end up with something like this:

```json
{
  "swagger": 2.0,
  "info": {
    "title": "Simple Weather API",
    "description": "API for getting the current weather information.",
    "version": "1.0"
  },
  "produces": ["application/json"],
  "basePath": "/api",
  "paths": {
    "/weather": {
      "get": {
        "description": "Returns the current weather for the requested location using the requested unit.",
        "parameters": [
          {
            "name": "location",
            "in": "query",
            "description": "The MSN Weather location search string.",
            "required": true,
            "type": "string"
          },
          {
            "name": "unit",
            "in": "query",
            "description": "The unit, either 'C' or 'F'.",
            "default": "F",
            "enum": ["C", "F"],
            "required": false,
            "type": "string"
          }
        ],
        "responses": {
          "default": {
            "description": "Invalid request.",
            "schema": {
              "$ref": "Error"
            }
          },
          "200": {
            "description": "Successful request.",
            "schema": {
              "$ref": "Weather"
            }
          }
        }
      }
    }
  },
  "definitions": {
    "CurrentWeather": {
      "properties": {
        "temperature": {
          "type": "string"
        },
        "skycode": {
          "type": "string"
        },
        "skytext": {
          "type": "string"
        },
        "date": {
          "type": "string"
        },
        "observationtime": {
          "type": "string"
        },
        "observationpoint": {
          "type": "string"
        },
        "feelslike": {
          "type": "string"
        },
        "humidity": {
          "type": "string"
        },
        "winddisplay": {
          "type": "string"
        },
        "day": {
          "type": "string"
        },
        "shortday": {
          "type": "string"
        },
        "windspeed": {
          "type": "string"
        },
        "imageUrl": {
          "type": "string"
        }

      },
      "required": ["temperature", "skycode", "skytext", "date", "observationtime", "observationpoint", "feelslike", "humidity", "winddisplay", "day", "shortday", "windspeed", "imageUrl"]
    },
    "Error": {
      "properties": {
        "message": {
          "type": "string"
        }
      },
      "required": ["message"]
    },
    "Forecast": {
      "properties": {
        "low": {
          "type": "string"
        },
        "high": {
          "type": "string"
        },
        "skycodeday": {
          "type": "string"
        },
        "skytextday": {
          "type": "string"
        },
        "date": {
          "type": "string"
        },
        "day": {
          "type": "string"
        },
        "shortday": {
          "type": "string"
        },
        "precip": {
          "type": "string"
        }
      },
      "required": ["low", "high", "skycodeday", "skytextday", "date", "day", "shortday", "precip"]
    },
    "Location": {
      "type": "object",
      "properties": {
        "name": {
          "type": "string"
        },
        "zipcode": {
          "type": "string"
        },
        "lat": {
          "type": "string"
        },
        "long": {
          "type": "string"
        },
        "timezone": {
          "type": "string"
        },
        "alert": {
          "type": "string"
        },
        "degreetype": {
          "type": "string",
          "enum": ["C", "F"]
        },
        "imagerelativeurl": {
          "type": "string"
        }
      },
      "required": ["name", "lat", "long", "timezone", "degreetype"]
    },
    "Weather": {
      "properties": {
        "location": {
          "$ref": "Location"
        },
        "current": {
          "$ref": "CurrentWeather"
        },
        "forecast": {
          "type": "array",
          "items": {
            "$ref": "Forecast"
          }
        }
      },
      "required": ["current", "location", "forecast"]
    }
  }
}
```

As you can see, both of our request parameters (`location` and `unit`) are defined as query parameters and are
restricted based on our requirements.  The response object is modeled after

**Note:** To figure out how we came up with this, please view the [Swagger 2.0 Specification][swagger-2.0-spec].  Also,
there are Swagger validation features we did not use in the model definitions to keep this document brief.  Those too
are documented in the Swagger 2.0 Specification.

## Implementing Your API

Implementing an API typically has a four major parts:

1. Validate the incoming parameters
2. Setup default values for missing incoming parameters
3. Do the real work required to _implement_ the API
4. Send response

Steps `1`, `2` and ,`4` could be considered _boilerplate_ as all of your APIs will have this code and all of your APIs
will have the same needs.  Let's look at how we might implement this.

### Without Swagger Tools

The above being said, without Swagger Tools or some similar tool you'd end up with something point like this:

```javascript
var weather = require('weather-js');
var send400 = function send400 (res, next, msg) {
  res.statusCode = 400;
  return next(msg);
};

module.exports.getWeather = function getWeather (req, res, next) {
  var location = req.params.location;
  var unit = req.params.unit || 'F'; // Default to 'F' if unit is not provided

  // Check that location is provided
  if (typeof location === 'undefined') {
    return send400('location is a required query parameter');
  }

  // Code necessary to check that if unit is provided, it is one of the valid options
  if (['C', 'F'].indexOf(req.param.unit) === -1) {
    return send400('unit must be either C or F');
  }

  // Code necessary to consume the Weather API and respond
  weather.find({search: req.params.location, degreeType: req.params.unit}, function(err, result) {
    if (err) {
      return next(err.message);
    }

    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify(result, null, 2));
  });
};
```

There is nothing necessarily wrong with this but you'll notice in some cases, a lot of the actual code being written is
for validating the request parameters and the rest of the code is pretty simple.  Even if that is not the case for your
API, as there is some complexity behind the route handler, you still have the boilerplate code required to validate your
request parameters, set default values for missing request parameters, etc.  So all of your APIs will have boilerplate
code before you can even implement the business logic behind your API.  There has to be something better...

Let's forget code for a second and think of the API documentation.  Let's say your API changes.  Not only does your
documentation need to be updated to reflect this change but now all code related to the changed API needs to change.
So instead of a single "source of truth", you now have at least two: One for documentation and one for implementation.

Swagger Tools aims to help you with these issues by treating your Swagger document as *the* source of truth and
providing middleware to consume your Swagger document for documentation generation, request validation, etc.  This means
that if you want to update the constraints for an API's request parameters, you update the Swagger document and your
generated documentation reflects this change and so does the request validation portion of the API itself.

### With Swagger Tools

With Swagger Tools, your code would look more like this:

```javascript
var weather = require('weather-js');

module.exports.getWeather = function getWeather (req, res, next) {
  var location = req.swagger.params.location.value;
  var unit = req.swagger.params.unit.value;

  // Code necessary to consume the Weather API and respond
  weather.find({search: location, degreeType: unit}, function(err, result) {
    if (err) {
      return next(err.message);
    }

    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify(result, null, 2));
  });
};
```

You'll notice the code that handles default values for request parameters and the validation of request parameters is
gone.  The reason for this is Swagger Tools ships with a middleware called [swaggerMetadata][swagger-metadata] that
takes your Swagger document, matches the request against an operation in your Swagger document and when the request
matches a defined operation, the information in the Swagger document is used to process the request parameters.  There
is another Swagger Tools middleware called [swaggerValidator][swagger-validator] that uses the information added to the
request by swaggerMetadata to validate the request parameters.

These two middlewares will help you remove a lot of boilerplate from your code.  Not only that but using these two
middlewares allows you to put all of your request parameter validation constraints in one place and have the related
items (documentation, API request validation, etc.) updated automatically.

## Putting it All Together

So now that we've designed our API and we've implemented our API, let's put this all together to have a functional API
using Swagger and Swagger Tools.

### Installing Dependencies

**body-parser** [[npm][body-parser]]

> Node.js body parsing middleware.

`npm install body-parser --save`

**connect** [[npm][connect]]

> High performance middleware framework.

`npm install connect --save`

**parseurl** [[npm][parseurl]]

> Parse a url with memoization

`npm install parseurl --save`

**qs** [[npm][qs]]

> A querystring parser that supports nesting and arrays, with a depth limit.

`npm install qs --save`

**swagger-tools** [[npm][swagger-tools]]

> Various tools for using and integrating with Swagger.

`npm install swagger-tools --save`

### Creating Our Server

The server for our API will be very simple.  Since we are using Swagger Tools, most of the server code will be wiring
that up.  Before that, we should create a place to store our controllers _(grouped route controllers)_ and our Swagger
document.  For this example, our Swagger document will live at `./api/swagger.json` and our controllers will live at
`./controllers/`.

The next thing we need to do is create our server.  This is the code that will wire up our server components, like
Swagger Tools, and then begin listening on a port to begin hosting the API.  Below is an example of such a  server:

```javascript
var app = require('connect')();
var bodyParser = require('body-parser');
var http = require('http');
var parseurl = require('parseurl');
var path = require('path');
var qs = require('qs');
var swaggerTools = require('swagger-tools');
var swaggerMetadata = swaggerTools.middleware.v2.swaggerMetadata;
var swaggerRouter = swaggerTools.middleware.v2.swaggerRouter;
var swaggerValidator = swaggerTools.middleware.v2.swaggerValidator;

var serverPort = 3000;

// swaggerMetadata configuration
var options = {
  controllers: './controllers',
  useStubs: process.env.NODE_ENV === 'development' ? true : false // Conditionally turn on stubs (mock mode)
};

// The Swagger document (require it, build it programmatically, fetch it from a URL, ...)
var swaggerDoc = require(path.join(__dirname, 'api', 'swagger.json'));

// Validate the Swagger document
var result = swaggerTools.specs.v2.validate(swaggerDoc);

if (typeof result !== 'undefined') {
  if (result.errors.length > 0) {
    console.log('The server could not start due to invalid Swagger document...');

    console.log('');

    console.log('Errors');
    console.log('------');

    result.errors.forEach(function (err) {
      console.log('#/' + err.path.join('/') + ': ' + err.message);
    });

    console.log('');
  }

  if (result.warnings.length > 0) {
    console.log('Warnings');
    console.log('--------');

    result.warnings.forEach(function (warn) {
      console.log('#/' + warn.path.join('/') + ': ' + warn.message);
    });
  }

  if (result.errors.length > 0) {
    process.exit(1);
  }
}

// Wire up the middleware required by Swagger Tools (body-parser and qs)
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: false }));
app.use(function (req, res, next) {
  if (!req.query) {
    req.query = req.url.indexOf('?') > -1 ? qs.parse(parseurl(req).query, {}) : {};
  }

  return next();
});

// Interpret Swagger resources and attach metadata to request - must be first in swagger-tools middleware chain
app.use(swaggerMetadata(swaggerDoc));

// Validate Swagger requests
app.use(swaggerValidator());

// Route validated requests to appropriate controller
app.use(swaggerRouter(options));

// Start the server
http.createServer(app).listen(serverPort, function () {
  console.log('Your server is listening on port %d (http://localhost:%d)', serverPort, serverPort);
});
```

### Testing

At this point, if you run your server using `node .` in my example, you should see the following:

```
Your server is listening on port 3000 (http://localhost:3000)
```

**Note:** If you run into an issue with your server saying your Swagger document fails validation, this is due to a
known bug in the Swagger 2.0 JSON Schema file.  To work around this, remove the `default` and `enum` properties from the
`#/paths/weather/get/parameters/1` object.  _(This will also require that you pass in the `unit` query parameter in all
examples below until this bug can be fixed upstream.)_

To test this API, using whatever tool you want perform a `GET` on `http://localhost:3000/api/weather`.  Since our API
requires the `location` query string, you should get a `400` with an error message dictating that the request requires
the `location` parameter.  So update your `GET` request to include the `location` query parameter.  Here is an example:
`http://localhost:3000?location=95113`.

At this point you should get a `404` because there are no route handlers configured.  If you view the
[Swagger Router (How To Use)][swagger-router-how-to-use], you'll see we can just use the `x-swagger-router-controller`
property on either `#/paths/weather` or the `#/paths/weather/get`.  For our example, add the
`x-swagger-router-controller` property to `#/paths/weather/get` and give it a value of `Weather`.  Once you do this,
the middleware will have a controller wired up to handle `GET` requests for the `/api/weather` path.  To enable it, we
need to restart our server and run Node.js in the `development` environment.  _(The reason for this is in our code above
we have the `useStubs` option for `swaggerRouter` enabled conditionally based on the Node.js environment.)_  To do this,
restart your server using something like: `NODE_ENV=development node .`.  With mock mode enabled, if you perform the
same `GET` on `http://localhost:3000/api/weather`, you should see a mock response that conforms to the `Weather` model
defined in `#/definitions/Weather`.  Here is an example:

```json
{
  "current":{
    "date":"Sample text",
    "day":"Sample text",
    "feelslike":"Sample text",
    "humidity":"Sample text",
    "imageUrl":"Sample text",
    "observationpoint":"Sample text",
    "observationtime":"Sample text",
    "shortday":"Sample text",
    "skycode":"Sample text",
    "skytext":"Sample text",
    "temperature":"Sample text",
    "winddisplay":"Sample text",
    "windspeed":"Sample text"
  },
  "forecast":[
    {
      "date":"Sample text",
      "day":"Sample text",
      "high":"Sample text",
      "low":"Sample text",
      "precip":"Sample text",
      "shortday":"Sample text",
      "skycodeday":"Sample text",
      "skytextday":"Sample text"
    }
  ],
  "location":{
    "alert":"Sample text",
    "degreetype":"C",
    "imagerelativeurl":"Sample text",
    "lat":"Sample text",
    "long":"Sample text",
    "name":"Sample text",
    "timezone":"Sample text",
    "zipcode":"Sample text"
  }
}
```

Now of course, we want to have a **real** API so let's implement the route handler.  Based on the current Swagger
document and the code above, to handle the route we will need to create a Node.js module at `./controllers/Weather.js`.
How does swaggerRouter figure out which method to call in the controller module?  Well, the default is to use the
operation method from `#/paths/weather`, in this case `get`.  What if you do not want to use an HTTP verb as your
JavaScript function name?  Well, you can use the `operationId` of an operation to dictate the method name.  So if you
wanted to name the route handler function `getWeather`, you could add the `operationId` property to the operation
at `#/paths/weather/get` and have its value be `getWeather`.  _(The example code below will assume you've done
this.)_

### Creating Our Route Handler

Based on the example above, we will be creating/editing the `./controllers/Weather.js` Node.js module.  We will also be
adding a new Node.js module to our project, [weather-js][weather-js] by running `npm install weather-js --save`.  With
that taken care of, below is an idea of what the controller module might look like to fulfill our needs:

```javascript
var weather = require('weather-js');

module.exports.getWeather = function getWeather (req, res, next) {
  var location = req.swagger.params.location.value;
  var unit = req.swagger.params.unit.value;

  // Code necessary to consume the Weather API and respond
  weather.find({search: location, degreeType: unit}, function(err, result) {
    if (err) {
      return next(err.message);
    }

    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify(result[0] || {}, null, 2));
  });
};

```

When you restart your server and perform at `GET` on `http://localhost:3000/api/weather?location=95113`, instead of the
mock response you should get a **real** response that looks something like this:

```json
{
  "location": {
    "name": "San Jose, CA",
    "zipcode": "95112",
    "lat": "37.3460541",
    "long": "-121.8877563",
    "timezone": "-7",
    "alert": "",
    "degreetype": "F",
    "imagerelativeurl": "http://wst.s-msn.com/i/en-us/"
  },
  "current": {
    "temperature": "70",
    "skycode": "34",
    "skytext": "Mostly Sunny",
    "date": "2014-09-23",
    "observationtime": "11:53:00",
    "observationpoint": "San Jose, San Jose International Airport",
    "feelslike": "70",
    "humidity": "63",
    "winddisplay": "7 mph",
    "day": "Tuesday",
    "shortday": "Tue",
    "windspeed": "7",
    "imageUrl": "http://wst.s-msn.com/i/en-us/law/34.gif"
  },
  "forecast": [
    {
      "low": "60",
      "high": "86",
      "skycodeday": "34",
      "skytextday": "Mostly Sunny",
      "date": "2014-09-23",
      "day": "Tuesday",
      "shortday": "Tue",
      "precip": "0"
    },
    {
      "low": "62",
      "high": "87",
      "skycodeday": "30",
      "skytextday": "Partly Cloudy",
      "date": "2014-09-24",
      "day": "Wednesday",
      "shortday": "Wed",
      "precip": "0"
    },
    {
      "low": "57",
      "high": "80",
      "skycodeday": "30",
      "skytextday": "Partly Cloudy",
      "date": "2014-09-25",
      "day": "Thursday",
      "shortday": "Thu",
      "precip": "0"
    },
    {
      "low": "56",
      "high": "79",
      "skycodeday": "28",
      "skytextday": "Mostly Cloudy",
      "date": "2014-09-26",
      "day": "Friday",
      "shortday": "Fri",
      "precip": "0"
    },
    {
      "low": "56",
      "high": "84",
      "skycodeday": "30",
      "skytextday": "Partly Cloudy",
      "date": "2014-09-27",
      "day": "Saturday",
      "shortday": "Sat",
      "precip": "0"
    }
  ]
}
```

## Conclusion

Swagger Tools provides tooling for ensuring your Swagger documents are valid and with your Swagger documents, scaffold
out and implement a REST API with minimal repetition and boilerplate.  If you have any feedback, please let us know.

[body-parser]: https://www.npmjs.org/package/body-parser
[connect]: https://www.npmjs.org/package/connect
[msn-weather]: http://local.msn.com/weather.aspx
[parseurl]: https://www.npmjs.org/package/parseurl
[qs]: https://www.npmjs.org/package/qs
[swagger]: http://swagger.io
[swagger-2.0-spec]: https://github.com/reverb/swagger-spec/blob/master/versions/2.0.md
[swagger-router]: https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md#swagger-router
[swagger-router-how-to-use]: https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md#how-to-use
[swagger-tools]: https://www.npmjs.org/package/swagger-tools
[weather-js]: https://github.com/cmfatih/weather