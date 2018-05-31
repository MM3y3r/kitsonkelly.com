---
title: TypeScript and Azure Functions
date: 2018-05-28 14:39:41
tags:
- azure
- microsoft
- typescript
---

I have been some time self educating on Azure Functions and TypeScript, and I want to share my learnings.

<!--more-->

Serverless is becoming big.  I had touched Lambda at one point, but it generally frustrated me to the point where I couldn't quite even get off the ground.  I wanted to understand a bit more about Azure, and in particular wanted to see what it was like writing [Azure Functions](https://azure.microsoft.com/en-gb/services/functions/) in [TypeScript](https://www.typescriptlang.org/).

## General observations

Azure Functions are going through a generation upgrade at the time of this writing.  V1 is what is production by default, but V2 is currently in preview mode.  It is clear there are a _lot_ of improvements in V2, and quite a few positive surprises.  But because this was education, I chose to go with v2.  This led to a lot of confusion on my part though as some of the key documentation was using old v1 syntax for some critical areas, specifically when you start to do bindings that are outside of the core bindings.  Bindings are the main way for an Azure Function to interact with the rest of the world.  More on that later.  There were also quite a few new features that were not clearly documented either, which only lots of googling identified that they even existed.

One of my early frustrations is that I am disappointed that Azure Functions currently lacks first class support for TypeScript.  In the end though, it is possible to reasonably configure a workspace where it is generally transparent.

## Basic structure

Tutorials often give you the step by step, and then only in retrospect you learn what you are doing and why.  Also there was a focus on the [Azure Portal](https://portal.azure.com/), which tried to set you up and guide you through.  The challenge is that once you want to go off and do something that isn't a quick start, you start to really learn how things hang together.

Since I focused on TypeScript, it meant at run-time I was still doing NodeJS/JavaScript functions.  So the whole of this post assumes that.

Azure Functions are deployed in a Function App.  The Function App is a specialised version of the Auzure App Service.  The main descriptor for the Function App is the `host.json`.  It can take a lot of host related settings, but in my experiments yet, I have not found any need to add anything.  Locally, you can have a `local.settings.json` which will mirror the Function App settings when deployed.  There is tooling to syncronise the server settings with your local settings.

For each Function there is in an App, there is a separate directory (which by default is the path to the function) and in that directory, there is a `function.json` which is the descriptor for the function.  For a basic HTTP request trigger Function, it would look like this:

```json
{
  "disabled": false,
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ]
}
```

This establishes that the function is not disabled, and describes the bindings of the function, how the function will communicate with the outside world.  In this case, it specifies that it is _triggered_ by an HTTP request, and that information about the request will be bound to the `req` property in the function's context.  It also specifies that the `res` property will contain the response from the function to be sent back to the requestor.

The other thing that is required is an index file for function, typically `index.js`.  This should be a CommonJS module which exports a function, or has a function name `run` which it exports (the latter doesn't seem to be obviously documented).

The function should take at least a single argument, which is the entire context of the request, and can then optionally take each additional binding above.  The bindings are also available in the context object, which I find to be a lot more straight forward to use.

When the function is finished processing, it can call the `.done()` method on the context object, or it can return a `Promise` or be `async` in which when the function resolves, it will process the results.  The ability to use `async` functions is also not obviously documented as part of v2.

So in TypeScript a basic function would look something like this:

```ts
export async function run(context: any) {
  const { req } = context;
  if (req.query.name || (req.body && req.body.name)) {
    context.res = {
      // status: 200, /* Defaults to 200 */
      body: 'Hello ' + (req.query.name || req.body.name)
    };
  } else {
    context.res = {
      status: 400,
      body: 'Please pass a name on the query string or in the request body'
    };
  }
}
```

While it is also not very well documented, when an Azure Function app is deployed, An npm install in production mode will occur, therefore you can add packages to your Function App via having them in the `package.json`. 

## Typings

There were two attempts at trying to create some sort of types for Azure Functions, one by the Azure team and one by an individual contributor.  Neither of them were part of DefinitelyTyped, and therefore aren't part of `@types`.  Even then I had some arguments with them.  I have been trying to get some [types](https://github.com/DefinitelyTyped/DefinitelyTyped/pull/24961) available, and I am sure I will get them landed one day, but I am still learning and will want to get them right first.  At the moment, I am using the following types locally though:

```ts
// Type definitions for Azure Functions 2.0.X
// Project: https://github.com/Azure/azure-functions-core-tools
// Definitions by: Christopher Anderson <https://github.com/christopheranderson>,
//                 Chris Feijoo <https://github.com/kube>,
//                 Kitson Kelly <https://github.com/kitsonk>
// Definitions: https://github.com/DefinitelyTyped/DefinitelyTyped

// TypeScript Version: 2.2

/// <reference types="node" />

export interface Bindings {
  response?: FunctionResponse;
  [key: string]: any;
}

export interface Context<B = Bindings> {
  bindings: B;
  bindingData: any;
  invocationId: string;

  log: ContextLog;
  done(err?: any, output?: object): void;
}

export interface ContextLog {
  (message?: any, ...optionalParams: any[]): void;
  error(message?: any, ...optionalParams: any[]): void;
  warn(message?: any, ...optionalParams: any[]): void;
  info(message?: any, ...optionalParams: any[]): void;
  verbose(message?: any, ...optionalParams: any[]): void;
}

export interface FunctionRequest {
  /**
   * An object that contains the body of the request.
   */
  body: any;

  /**
   * An object that contains the request headers.
   */
  headers: { [name: string]: string };

  /**
   * The HTTP method of the request.
   */
  method: HttpMethod;

  /**
   * The URL of the request.
   */
  originalUrl: string;

  /**
   * An object that contains the routing parameters of the request.
   */
  params: { [param: string]: string };

  /**
   * An object that contains the query parameters.
   */
  query: { [key: string]: string };

  /**
   * The body of the message as a string.
   */
  rawBody: any;
}

export interface FunctionResponse {
  /**
   * An object that contains the body of the response.
   */
  body?: any;

  /**
   * An object that contains the response headers.
   */
  headers?: {
    'content-type'?: string;
    'content-length'?: number;
    'content-disposition'?: string;
    'content-encoding'?: string;
    'content-language'?: string;
    'content-range'?: string;
    'content-location'?: string;
    'content-md5'?: Buffer;
    expires?: Date;
    'last-modified'?: Date;
    [name: string]: any;
  };

  /**
   * Indicates that formatting is skipped for the response.
   */
  isRaw?: boolean;

  /**
   * The HTTP status code of the response.
   */
  status?: HttpStatusCode | number;
}

export interface HttpContext<B = Bindings> extends Context<B> {
  req: FunctionRequest;
  res: FunctionResponse;
}

export type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'HEAD' | 'OPTIONS' | 'TRACE' | 'CONNECT' | 'PATCH';

export enum HttpStatusCode {
  // 1XX Informational
  Continue = 100,
  SwitchingProtocols = 101,
  Processing = 102,
  Checkpoint = 103,

  // 2XX Success
  OK = 200,
  Created = 201,
  Accepted = 202,
  NonAuthoritativeInformation = 203,
  NoContent = 204,
  ResetContent = 205,
  PartialContent = 206,
  MultiStatus = 207,
  AlreadyReported = 208,
  IMUsed = 226,

  // 3XX Redirection
  MultipleChoices = 300,
  MovedPermanently = 301,
  Found = 302,
  SeeOther = 303,
  NotModified = 304,
  UseProxy = 305,
  SwitchProxy = 306,
  TemporaryRedirect = 307,
  PermanentRedirect = 308,

  // 4XX Client Error
  BadRequest = 400,
  Unauthorized = 401,
  PaymentRequired = 402,
  Forbidden = 403,
  NotFound = 404,
  MethodNotAllowed = 405,
  NotAcceptable = 406,
  ProxyAuthenticationRequired = 407,
  RequestTimeout = 408,
  Conflict = 409,
  Gone = 410,
  LengthRequired = 411,
  PreconditionFailed = 412,
  PayloadTooLarge = 413,
  URITooLong = 414,
  UnsupportedMediaType = 415,
  RangeNotSatisfiable = 416,
  ExpectationFailed = 417,
  ImATeapot = 418,
  MethodFailure = 420,
  EnhanceYourCalm = 420,
  MisdirectedRequest = 421,
  UnprocessableEntity = 422,
  Locked = 423,
  FailedDependency = 424,
  UpgradeRequired = 426,
  PreconditionRequired = 428,
  TooManyRequests = 429,
  RequestHeaderFieldsTooLarge = 431,
  LoginTimeout = 440,
  NoResponse = 444,
  RetryWith = 449,
  BlockedByWindowsParentalControls = 450,
  UnavailableForLegalReasons = 451,
  Redirect = 451,
  SSLCertificateError = 495,
  SSLCertificateRequired = 496,
  HttpRequestSentToHttpsPort = 497,
  ClientClosedRequest = 499,
  InvalidToken = 498,
  TokenRequired = 499,
  RequestWasForbiddenByAntivirus = 499,

  // 5XX Server Error
  InternalServerError = 500,
  NotImplemented = 501,
  BadGateway = 502,
  ServiceUnavailable = 503,
  GatewayTimeout = 504,
  HttpVersionNotSupported = 505,
  VariantAlsoNegotiates = 506,
  InsufficientStorage = 507,
  LoopDetected = 508,
  BandwidthLimitExceeded = 509,
  NotExtended = 510,
  NetworkAuthenticationRequired = 511,
  UnknownError = 520,
  WebServerIsDown = 521,
  ConnectionTimedOut = 522,
  OriginIsUnreachable = 523,
  ATimeoutOccurred = 524,
  SSLHandshakeFailed = 525,
  InvalidSSLCertificate = 526,
  SiteIsFrozen = 530
}
```

## TypeScript configuration

Azure Functions v2 supports different versions of the NodeJS runtime.  The actual version can be specified on the Function App Application Settings with the key of `WEBSITE_NODE_DEFAULT_VERSION`.  I have been using the current LTS which is `8.11.1`.  This supports `async`/`await` natively and the rest of ES2017, so the following `tsconfig.json` is what I am currently using.

```json
{
    "compilerOptions": {
        "inlineSources": true,
        "module": "commonjs",
        "sourceMap": true,
        "strict": true,
        "target": "es2017",
        "types": [
            "node"
        ]
    },
    "compileOnSave": true,
    "include": [
        "*/index.ts"
    ]
}
```

This will essentially compile all the `index.ts` in all the subdirectories of the root, which is exactly how the Azure Function App will scan for functions.  It also creates a source map, which can be used both for local and remote debugging of your functions.

## Command line tooling

I have found that the command line tooling is necessary for running and debugging your functions locally.

## Using Visual Studio Code

