[« previous](07-collections.md) | [next »](09-wiring.md)

## 8. ApiV1
In this chapter we are going to build a http package to be able to handle http requests in a standardized way.
Then we are going to provide the api version 1 package, which does require this http adapter.

### 8.1 A standardized way to handle http requests
To fire some http requests, we should define how a http request,
a http response and its handler should look like.

Therefore, let's create a new `http` package by creating its interfaces.
Let's define the interface for the request and its factory function first:

```typescript jsx
// src/packages/core/http/request.ts

import { v4 } from 'uuid';

export type RequestMethod = 'get' | 'post' | 'put' | 'patch' | 'delete';

export type Request<Body = any, Qry = any, Headers = any> = {
    id: string;
    url: string;
    method: 'get' | 'post' | 'put' | 'patch' | 'delete';
    headers: Headers;
    queryParameters: Qry;
    body: Body;
};

type CreationSettings = Pick<Request, 'url' | 'method'> & Partial<Omit<Request, 'url' | 'method'>>;
export function createRequest(settings: CreationSettings): Request {
    return {
        id: v4(),
        body: {},
        headers: {},
        queryParameters: undefined,
        ...settings,
    };
}
```

Well, now that we have the request defined, we also need to define a way to handle it.
We should keep in mind that the request does not reach the server in every case.
Let's imagine you have some connection problems.
In such a case we should have some extra information about the request-response-process.
I think it's a good idea that the feedback of this process should provide the initial request with the response.
So let's wrap the request and its response together in an object called `RequestResponse`.
Furthermore, it would be nice to have the possibility to see how the progress of the request is in percentage.
We should also be possible to cancel our requests and see such information in our `RequestResponse` object.

Uff this is a lot! Let's try to write these requirements down as code:

```typescript jsx
// src/packages/core/http/requestHandler.ts

import { Request } from './request';

export type Response<Body = any, Headers extends object = any> = {
    status: number;
    headers: Headers;
    body: Body;
};

export type RequestResponse<Res extends Response = any, Req extends Request = any> = {
    request: Req;
    response?: Res;
    hasRequestBeenCancelled: boolean;
};

export type RequestExecutionConfig = {
    request: Request;
    onProgress?: (percentage: number) => void;
};

export type RequestHandler = {
    executeRequest: (config: RequestExecutionConfig) => Promise<RequestResponse>;
    cancelRequestById(requestId: string): void;
};
```

With that `RequestResponse` object, we know whether we cancelled the request or not, when the `response` property is `undefined`.
If `hasRequestBeenCancelled` is false and `response` is `undefined`, we know that there were some connection problems.

Don't forget to export these parts:

```typescript jsx
// src/packages/core/http/index.ts

export * from './request';
export * from './requestHandler';
```

### 8.2 Axios request handler
There are several libraries available which support easier http request handling in JS.
One of these libraries is [Axios](https://www.npmjs.com/package/axios).
As of today (year 2022) Axios is the most common library.
One might argue, that the native [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
could be taken as well but [older browsers do not support this](https://caniuse.com/?search=fetch).
So let's write an axios implementation for the `RequestHandler` interface, defined before:

```typescript jsx
// src/packages/core/http/axiosRequestHandler.ts

import axios, { AxiosRequestConfig, CancelTokenSource, Method, AxiosResponse } from 'axios';
import { Request } from './request';
import { RequestExecutionConfig, RequestHandler, RequestResponse } from './requestHandler';

function createRequestResponse(
    request: Request,
    response: undefined | AxiosResponse,
    hasRequestBeenCancelled: boolean
): RequestResponse {
    return {
        hasRequestBeenCancelled,
        request,
        response: !response
            ? undefined
            : {
                status: response.status,
                headers: response.headers,
                body: response.data,
            },
    };
}

function createAxiosConfig(generalReqCfg: AxiosRequestConfig, config: RequestExecutionConfig): AxiosRequestConfig {
    const { request } = config;
    let requestConfig: AxiosRequestConfig = {
        ...generalReqCfg,
        method: request.method as Method,
        url: request.url,
    };
    if (request.headers) {
        requestConfig.headers = request.headers;
    }
    if (request.body) {
        requestConfig.data = request.body;
    }
    if (request.queryParameters) {
        requestConfig.params = request.queryParameters;
    }
    if (config.onProgress) {
        requestConfig.onUploadProgress = (progressEvent) => {
            if (!config.onProgress) {
                return;
            }
            config.onProgress(Math.round((progressEvent.loaded * 100) / progressEvent.total));
        };
    }
    return requestConfig;
}

type RequestIdToCancelTokenSourceMapping = {
    [requestId: string]: CancelTokenSource;
};

export class AxiosRequestHandler implements RequestHandler {
    private readonly generalRequestConfig: AxiosRequestConfig;
    private readonly requestIdToCancelTokenSourceMapping: RequestIdToCancelTokenSourceMapping;

    constructor(generalRequestConfig: AxiosRequestConfig = {}) {
        this.generalRequestConfig = generalRequestConfig;
        this.requestIdToCancelTokenSourceMapping = {};
    }

    executeRequest(config: RequestExecutionConfig): Promise<RequestResponse> {
        const { request } = config;
        const cancelTokenSource = axios.CancelToken.source();
        const requestIdToCancelTokenSourceMapping = this.requestIdToCancelTokenSourceMapping;
        const axiosRequestCfg: AxiosRequestConfig = {
            ...createAxiosConfig(this.generalRequestConfig, config),
            cancelToken: cancelTokenSource.token,
        };
        requestIdToCancelTokenSourceMapping[request.id] = cancelTokenSource;
        return new Promise((resolve) => {
            axios(axiosRequestCfg)
                .then((response): void => {
                    delete requestIdToCancelTokenSourceMapping[request.id];
                    const requestResponse = createRequestResponse(request, response, false);
                    resolve(requestResponse);
                })
                .catch((error): void => {
                    delete requestIdToCancelTokenSourceMapping[request.id];
                    if (axios.isCancel(error)) {
                        const requestResponse = createRequestResponse(request, error.response, true);
                        resolve(requestResponse);
                        return;
                    }
                    if (!error.request) {
                        console.error(error);
                        throw new Error('unexpected axios error printed above');
                    }
                    const requestResponse = createRequestResponse(request, error.response, false);
                    resolve(requestResponse);
                });
        });
    }

    cancelRequestById(requestId: string) {
        const cancelTokenSource = this.requestIdToCancelTokenSourceMapping[requestId];
        if (!cancelTokenSource) {
            return;
        }
        cancelTokenSource.cancel();
        delete this.requestIdToCancelTokenSourceMapping[requestId];
    }
}
```

As you might have noticed, we don't `reject` the returned Promise in the `executeRequest` method.
As of this, we are able to handle every case just with the `then` method which got the `RequestResponse` object injected,
so we can save some extra lines of code whenever we want to handle a request.

Don't forget to add the file it in the `index.ts`.
Well done! We can use this http handler in the api version 1 package we are going to write next.

:floppy_disk: [branch 08-apiv1-1](https://github.com/inkognitro/react-app-tutorial-code/compare/07-collections-2...08-apiv1-1)

### 8.3 

:floppy_disk: [branch 08-apiv1-2](https://github.com/inkognitro/react-app-tutorial-code/compare/08-apiv1-1...08-apiv1-2)

[« previous](07-collections.md) | [next »](09-wiring.md)