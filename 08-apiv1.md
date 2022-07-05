[« previous](07-collections.md) | [next »](09-wiring.md)

## 8. ApiV1
In this chapter we are going to build a http package to handle http requests in a standardized way.
This enables us to provide an api version 1 package which is based on that http package.

### 8.1 A standardized way to handle http requests
To fire some http requests, we should define how a http request,
a http response and its handler should look like.

Therefore, let's create a new `http` package by creating its interfaces.
Let's define the interface for the request and its factory function first:

```typescript
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

```typescript
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

```typescript
// src/packages/core/http/index.ts

export * from './request';
export * from './requestHandler';
```

### 8.2 Axios request handler
There are several libraries available which support easier http request handling in JS.
One of these libraries is [Axios](https://www.npmjs.com/package/axios).
As of today (year 2022), Axios is the most common library for this.
One might argue, that the native [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
could be taken as well, but [older browsers do not support this](https://caniuse.com/?search=fetch).
So let's write an axios implementation for the `RequestHandler` interface, like we have defined before:

```typescript
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
As of this, we are able to handle every case just with the `then` method, which gets the `RequestResponse` object injected.
By ignoring the `catch` method, we can save some extra code whenever we want to handle a request.

Don't forget to add the file it in the `index.ts`.
Well done! We can use this http handler in the api version 1 package we are going to write next.

:floppy_disk: [branch 08-apiv1-1](https://github.com/inkognitro/react-app-tutorial-code/compare/07-collections-2...08-apiv1-1)

### 8.3 Api types
Like the `http` package, we should also define a package which handles stuff of our http API version 1.
Don't worry, we'll mock an endpoint in the next chapter. For now, we are going to define how endpoints
should be structured.

Let's keep in mind the field messages, which we had defined in the `form` package.
In an ideal world these field messages would be delivered from the api itself.
If the messages came with the response in another format, we had to write some mapping code.
Let's go the easy way and make a field message array holding the same kind of objects like defined in the `form` package.
Furthermore, it would be nice to have some general messages available, which have nothing todo with the request
parameters itself. So let's go with the code below:

```typescript
// src/packages/core/api-v1/core/types.ts

import { Request, RequestMethod, RequestResponse, Response } from '@packages/core/http';

export type ApiV1TranslationPlaceholders = {
    [key: string]: string;
};

export type ApiV1Translation = {
    id: string;
    placeholders?: ApiV1TranslationPlaceholders;
};

export type ApiV1Message = {
    id: string;
    severity: 'info' | 'success' | 'warning' | 'error';
    translation: ApiV1Translation;
};

export type ApiV1FieldMessagePath = (string | number)[];

export type ApiV1FieldMessage = {
    path: ApiV1FieldMessagePath;
    message: ApiV1Message;
};

export type ApiV1EndpointId = {
    method: RequestMethod;
    path: string;
};

export type ApiV1Request<Payload = any> = {
    id: string;
    endpointId: ApiV1EndpointId;
    payload: Payload;
};

export type ApiV1RequestExecutionSettings<R extends ApiV1Request = any> = {
    request: R;
    transformer: ApiV1EndpointTransformer;
    onProgress?: (percentage: number) => void;
};

export type ApiV1ResponseBodyBase = {
    success: boolean;
    fieldMessages: ApiV1FieldMessage[];
    generalMessages: ApiV1Message[];
};

export enum ApiV1ResponseTypes {
    SUCCESS = 'success',
    ERROR = 'error',
}

export type ApiV1Response<T extends ApiV1ResponseTypes, Body extends object = {}> = { type: T } & Response<
    Body & ApiV1ResponseBodyBase
>;

export type ApiV1RequestResponse<Req extends ApiV1Request = any, Res extends ApiV1Response<any> = any> = {
    request: Req;
    response: undefined | Res;
    hasRequestBeenCancelled: boolean;
};

export type ApiV1EndpointTransformer<Req extends ApiV1Request = any, Res extends ApiV1Response<any> = any> = {
    endpointId: ApiV1EndpointId;
    createHttpRequest: (request: Req) => Request;
    createRequestResponse: (rr: RequestResponse, request: Req) => ApiV1RequestResponse<Req, Res>;
};
```

Like we have done in the http package, we should also handle our API requests with a request handler.
With the `ApiV1ResponseTypes`, like in the `AuthUser` object,
TS will be able to automatically cast the response after a `ApiV1Response.type` check.

Next, let's define the `ApiV1RequestHandler` interface like so:

```typescript
// src/packages/core/api-v1/core/requestHandler.ts

import { ApiV1RequestExecutionSettings } from './types';

export type ApiV1RequestHandler = {
    executeRequest: (settings: ApiV1RequestExecutionSettings) => Promise<any>;
    cancelAllRequests: () => void;
    cancelRequestById: (requestId: string) => void;
};
```

Before we create our first endpoint, we should provide some factory functions of our previously defined types.
These helper functions can be used for our endpoint definitions. Having those extracted will save us a lot of code:

```typescript
// src/packages/core/api-v1/core/factory.ts

import { v4 } from 'uuid';
import { Response, createRequest, Request } from '@packages/core/http';
import { ApiV1EndpointId, ApiV1FieldMessage, ApiV1Message, ApiV1Request, ApiV1ResponseBodyBase } from './types';

type AnyResponse = Response<{
    success: boolean;
    generalMessages?: ApiV1Message[];
    fieldMessages?: ApiV1FieldMessage[];
}>;

export function createApiV1BasicResponseBody(response: Response): ApiV1ResponseBodyBase {
    const r = response as AnyResponse;
    return {
        success: r.body.success,
        fieldMessages: r.body.fieldMessages ?? [],
        generalMessages: r.body.generalMessages ?? [],
    };
}

type PathParams = { [paramName: string]: string };
type QueryParams = object;
type BodyParams = object;

function createUrl(urlWithVars: string, pathParams: PathParams): string {
    let url = urlWithVars;
    for (let paramName in pathParams) {
        const value = pathParams[paramName];
        url = url.replaceAll('{' + paramName + '}', value);
    }
    return url;
}

type HttpRequestCreationOptions = {
    pathParams?: PathParams;
    queryParams?: QueryParams;
    bodyParams?: BodyParams;
};

export function createHttpRequestFromRequest(request: ApiV1Request, options: HttpRequestCreationOptions = {}): Request {
    return createRequest({
        url:
            options && options.pathParams
                ? createUrl(request.endpointId.path, options.pathParams)
                : request.endpointId.path,
        method: request.endpointId.method,
        id: request.id,
        queryParameters: options.queryParams,
        body: options.bodyParams,
    });
}

export type RequestBase = Pick<ApiV1Request, 'id' | 'endpointId'>;
export function createRequestBase(endpointId: ApiV1EndpointId): RequestBase {
    return {
        id: v4(),
        endpointId: endpointId,
    };
}
```

### 8.4 Register user endpoint
We are ready to define our first endpoint. Every endpoint should have a transformer.
The transformer's task is to transform an `ApiV1Request` into a `Request` from the http request,
as well as to transform a `RequestResponse` object from the `http` package into a `ApiV1RequestResponse` object.
which transforms received data into the expected return type format:
In the types we have defined before.

So let's assume the register user endpoint looks like below:
```typescript
// src/packages/core/api-v1/auth/registerUser.ts

import {
    ApiV1RequestHandler,
    ApiV1RequestResponse,
    ApiV1Response,
    createApiV1BasicResponseBody,
    createHttpRequestFromRequest,
    createRequestBase,
    ApiV1EndpointTransformer,
    ApiV1EndpointId,
    ApiV1Request,
    ApiV1ResponseTypes,
} from '../core';
import { RequestResponse, Response as HttpResponse } from '../../http';

const endpointId: ApiV1EndpointId = { method: 'post', path: '/auth/register' };

type AuthUser = {
    apiKey: string;
    user: {
        id: string;
        username: string;
    };
};

type RegisterUserResponse =
    | ApiV1Response<ApiV1ResponseTypes.SUCCESS, { data: AuthUser }>
    | ApiV1Response<ApiV1ResponseTypes.ERROR>;

type RegisterUserPayload = {
    gender: 'f' | 'm' | 'o';
    email: string;
    username: string;
    password: string;
};

type RegisterUserRequest = ApiV1Request<RegisterUserPayload>;

function createRegisterUserRequest(payload: RegisterUserPayload): RegisterUserRequest {
    return {
        ...createRequestBase(endpointId),
        payload,
    };
}

const registerUserTransformer: ApiV1EndpointTransformer<RegisterUserRequest, RegisterUserResponse> = {
    endpointId: endpointId,
    createHttpRequest: (request) => {
        return {
            ...createHttpRequestFromRequest(request),
            body: request.payload,
        };
    },
    createRequestResponse: (rr: RequestResponse, request) => {
        if (!rr.response) {
            return {
                request,
                hasRequestBeenCancelled: rr.hasRequestBeenCancelled,
                response: undefined,
            };
        }
        if (rr.response.status === 201) {
            const realSuccessResponse = rr.response as HttpResponse<{ data: AuthUser }>;
            return {
                request,
                hasRequestBeenCancelled: rr.hasRequestBeenCancelled,
                response: {
                    ...realSuccessResponse,
                    type: 'success',
                    body: {
                        ...createApiV1BasicResponseBody(realSuccessResponse),
                        ...rr.response.body,
                    },
                },
            };
        }
        const realErrorResponse = rr.response;
        return {
            request,
            hasRequestBeenCancelled: rr.hasRequestBeenCancelled,
            response: {
                ...realErrorResponse,
                type: 'error',
                body: createApiV1BasicResponseBody(realErrorResponse),
            },
        };
    },
};

export type RegisterUserRequestResponse = ApiV1RequestResponse<RegisterUserRequest, RegisterUserResponse>;

export function registerUser(
    requestHandler: ApiV1RequestHandler,
    payload: RegisterUserPayload
): Promise<RegisterUserRequestResponse> {
    return requestHandler.executeRequest({
        request: createRegisterUserRequest(payload),
        transformer: registerUserTransformer,
    }) as Promise<RegisterUserRequestResponse>;
}
```

We now are able to use the mapping function with whatever `ApiV1RequestHandler` we receive.
This is also useful for testing purposes.

As usual, export the endpoint like so:
```typescript
// src/packages/core/api-v1/auth/index.ts

export * from './registerUser';
```

> :bulb: Generally, I suggest importing properties from the third nesting level, just to keep things simple:
> `import { Translator } from '@packages/core/i18n';`
> 
> In such an exceptional case like our `api-v1` package, where a lot of contexts might have similar endpoints,
> It might be worth it to add a fourth nesting level: `import { registerUser } from '@packages/core/api-v1/auth';`

### 8.5 ApiV1RequestHandler: Http implementation
Until now, we don't have an implementation for the `ApiV1RequestHandler`.
So let's write one, to handle an `ApiV1Request` with the `http` package like so:

```typescript
// src/packages/core/api-v1/core/httpRequestHandler.ts

import { Request, RequestExecutionConfig, RequestHandler as HttpRequestHandler } from '@packages/core/http';
import { ApiV1RequestExecutionSettings, ApiV1RequestResponse } from './types';
import { ApiV1RequestHandler } from './requestHandler';

type AccessTokenFinder = () => null | string;

export class HttpApiV1RequestHandler implements ApiV1RequestHandler {
    private readonly baseUrl: string;
    private readonly requestHandler: HttpRequestHandler;
    private readonly findAccessToken: AccessTokenFinder;
    private runningRequestIds: string[];

    constructor(requestHandler: HttpRequestHandler, findAccessToken: AccessTokenFinder, baseUrl: string) {
        this.baseUrl = baseUrl;
        this.requestHandler = requestHandler;
        this.findAccessToken = findAccessToken;
        this.runningRequestIds = [];
    }

    public executeRequest(settings: ApiV1RequestExecutionSettings): Promise<ApiV1RequestResponse> {
        const requestFromTransformer = settings.transformer.createHttpRequest(settings.request);
        let request: Request = {
            ...requestFromTransformer,
            url: this.baseUrl + requestFromTransformer.url,
            id: settings.request.id,
        };
        const accessToken = this.findAccessToken();
        if (accessToken) {
            request = {
                ...request,
                headers: {
                    ...request.headers,
                    Authorization: 'Bearer ' + accessToken,
                },
            };
        }
        const requestExecutionCnf: RequestExecutionConfig = {
            onProgress: settings.onProgress,
            request,
        };
        const that = this;
        this.runningRequestIds.push(request.id);
        return new Promise((resolve) => {
            this.requestHandler
                .executeRequest(requestExecutionCnf)
                .then((requestResponse): void => {
                    let rr = settings.transformer.createRequestResponse(requestResponse, settings.request);
                    resolve(rr);
                })
                .finally(() => {
                    that.runningRequestIds = that.runningRequestIds.filter((requestId) => request.id !== requestId);
                });
        });
    }

    public cancelAllRequests() {
        const that = this;
        this.runningRequestIds.forEach((requestId) => that.cancelRequestById(requestId));
    }

    public cancelRequestById(requestId: string) {
        this.requestHandler.cancelRequestById(requestId);
    }
}
```

Don't forget to export it in the `index.ts`.

### 8.6 ApiV1RequestHandler: Scoped implementation
We should also have a handler which only cancels the requests of its own scope when it is unmounted.
Even though we are going to create the hook in the next chapter we can already prepare the
handler itself. Let's define it like below:

```typescript
// src/packages/core/api-v1/core/scopedRequestHandler.ts

import { createContext, useContext, useEffect, useRef } from 'react';
import { ApiV1RequestExecutionSettings, ApiV1RequestResponse } from './types';
import { ApiV1RequestHandler } from './requestHandler';

export class ScopedApiV1RequestHandler implements ApiV1RequestHandler {
    private readonly requestHandler: ApiV1RequestHandler;
    private runningRequestIds: string[];

    constructor(requestHandler: ApiV1RequestHandler) {
        this.requestHandler = requestHandler;
        this.runningRequestIds = [];
        this.createSeparated = this.createSeparated.bind(this);
        this.executeRequest = this.executeRequest.bind(this);
        this.cancelAllRequests = this.cancelAllRequests.bind(this);
        this.cancelRequestById = this.cancelRequestById.bind(this);
    }

    public createSeparated() {
        return new ScopedApiV1RequestHandler(this.requestHandler);
    }

    public executeRequest(settings: ApiV1RequestExecutionSettings): Promise<ApiV1RequestResponse> {
        const that = this;
        return new Promise((resolve) => {
            that.runningRequestIds.push(settings.request.id);
            that.requestHandler.executeRequest(settings).then((rr): void => {
                that.runningRequestIds = that.runningRequestIds.filter(
                    (requestId) => settings.request.id !== requestId
                );
                resolve(rr);
            });
        });
    }

    public cancelAllRequests() {
        this.runningRequestIds.forEach((requestId) => this.requestHandler.cancelRequestById(requestId));
    }

    public cancelRequestById(requestId: string) {
        if (!this.runningRequestIds.includes(requestId)) {
            return;
        }
        this.requestHandler.cancelRequestById(requestId);
    }
}
```

Next let's add a hook in the same file so that we can use the ScopedApiV1RequestHandler in our components.
As mentioned before, we should cancel all running requests on component unmount:

```typescript
// src/packages/core/api-v1/core/scopedRequestHandler.ts

const scopedApiV1RequestHandlerContext = createContext<null | ScopedApiV1RequestHandler>(null);
export const ScopedApiV1RequestHandlerProvider = scopedApiV1RequestHandlerContext.Provider;

export function useApiV1RequestHandler(): ApiV1RequestHandler {
    const requestHandler = useContext(scopedApiV1RequestHandlerContext);
    if (!requestHandler) {
        throw new Error('no ScopedApiV1RequestHandler was provided');
    }
    const requestHandlerRef = useRef<ApiV1RequestHandler | null>(null);
    if (!requestHandlerRef.current) {
        requestHandlerRef.current = requestHandler.createSeparated();
    }
    useEffect(() => {
        return () => {
            if (requestHandlerRef.current) {
                requestHandlerRef.current.cancelAllRequests();
            }
        };
    }, []);
    return requestHandlerRef.current;
}
```

Don't forget to export it in the `index.ts`.

Well done! I hope everything was clear so far. Don't worry that you could not test your code yet.
We are going to wire things in the next chapter and will hopefully recognize that the work
we have done in this chapter will save us a lot of code for features in the future.

:floppy_disk: [branch 08-apiv1-2](https://github.com/inkognitro/react-app-tutorial-code/compare/08-apiv1-1...08-apiv1-2)

[« previous](07-collections.md) | [next »](09-wiring.md)