[« previous](08-apiv1.md) | [final words »](10-finalwords.md)

## 9. Wiring
In this chapter we are going to write a mocked api with [Express](https://www.npmjs.com/package/express)
for the register user endpoint. In a next step we wire the form with this endpoint and take the
response's field message for form element error enrichment.

### 9.1 Provide a mock API
Before we can wire our user registration form, we have to create an API endpoint for it.
Let's create an endpoint which validates our inputs and returns some field messages.
One or more general messages would also be good. We could connect these with the toaster in one of the next steps.

To write our mocked API endpoint let's install Express first:
```
npm install --save-dev express cors
```

Then let's add our mock API like so:
```javascript
// src/mockApi.js

const express = require('express');
const cors = require('cors');
const uuid = require('uuid');

const port = 9000;
const app = express();
app.use(express.json())
app.use(cors({ origin: '*' }));

function createErrorMessage(messageId, translationId) {
    return {
        id: messageId,
        severity: 'error',
        translation: {
            id: translationId,
        }
    };
}

app.post('/api/v1/auth/register', function (req, res) {
    res.header("Access-Control-Allow-Origin", "*")

    const gender = req.body.gender;
    const username = req.body.username;
    const email = req.body.email;
    const password = req.body.password;

    const generalMessages = [];
    const fieldMessages = [];

    if (!gender || !['f', 'm', 'o'].includes(gender)) {
        fieldMessages.push({
            path: ['gender'],
            message: createErrorMessage('message-id-01', 'api.v1.invalidValue'),
        });
    }

    if (!username) {
        fieldMessages.push({
            path: ['username'],
            message: createErrorMessage('message-id-02', 'api.v1.invalidValue'),
        });
    }

    if (!email) {
        fieldMessages.push({
            path: ['email'],
            message: createErrorMessage('message-id-03', 'api.v1.invalidValue'),
        });
    }

    if (!password) {
        fieldMessages.push({
            path: ['password'],
            message: createErrorMessage('message-id-04', 'api.v1.invalidValue'),
        });
    }

    const success = fieldMessages.length === 0;

    if (!success) {
        generalMessages.push(createErrorMessage(uuid.v4(), 'api.v1.formErrors'));
        res.status(400);
        res.header('Content-Type', 'application/json');
        res.write(JSON.stringify({
            success,
            generalMessages,
            fieldMessages,
        }));
        res.send();
        return;
    }

    generalMessages.push(createErrorMessage(uuid.v4(), 'api.v1.userWasCreated'));
    res.status(201);
    res.header('Content-Type', 'application/json');
    res.write(JSON.stringify({
        success,
        generalMessages,
        fieldMessages,
        data: {
            apiKey: 'foo',
            user: {
                id: 'fbfe874c-ea8f-4cc1-bd4e-f07bedc30487',
                username,
            },
        }
    }));
    res.send();
});

console.log(`Mock-API is running at: http://localhost:${port}`);
app.listen(port);
```

This file should not be affected by our linting rules, so let's ignore it by adding a `.eslintignore` file inside
our project folder and paste following line in it: `src/mockApi.js`

Let's also add a new script in the `package.json` by enhancing the `scripts` section like so:
```javascript
"scripts": {
    // other scripts...
    "mockApi": "node ./src/mockApi.js"
}
```

Cool! We are now able to run the script and reach the mock API endpoint e.g. by [Postman](https://www.postman.com/).
Before we go to the next step, we should support the translations which could be delivered from the API by adding
following sections in the translations files:

```javascript
// src/components/translations/deCH.json

"api": {
    "v1": {
        "formErrors": "Ihre Formulardaten sind fehlerhaft!",
        "invalidValue": "Ungültiger Wert"
    }
},
```

and
```javascript
// src/components/translations/enUS.json

"api": {
    "v1": {
        "formErrors": "You have some errors in your form!",
        "invalidValue": "Invalid value"
    }
},
```

### 9.3 Provide the ApiV1RequestHandler
To use the `ApiV1RequestHandler` we need to provide it in the `ServiceProvider` like we did also for other services:

```typescript jsx
// src/ServiceProvider.tsx

// Replace following import statement:
import React, { FC, MutableRefObject, PropsWithChildren, useEffect, useRef, useState } from 'react';

// Add following import statements:
import { AxiosRequestHandler } from '@packages/core/http';
import {
    HttpApiV1RequestHandler,
    ScopedApiV1RequestHandler,
    ScopedApiV1RequestHandlerProvider,
} from '@packages/core/api-v1/core';


// Add the ApiV1RequestHandler factory like so:
function createScopedApiV1RequestHandler(currentUserStateRef: MutableRefObject<AuthUser>): ScopedApiV1RequestHandler {
    const httpRequestHandler = new AxiosRequestHandler();
    const apiV1BaseUrl = 'http://localhost:9000/api/v1';
    const httpApiV1RequestHandler = new HttpApiV1RequestHandler(
        httpRequestHandler,
        () => {
            return currentUserStateRef.current.type === 'authenticated' ? currentUserStateRef.current.apiKey : null;
        },
        apiV1BaseUrl
    );
    return new ScopedApiV1RequestHandler(httpApiV1RequestHandler);
}

// Insert following code in the top of the ServiceProvider component:
const currentUserStateRef = useRef<AuthUser>(currentUserState);
useEffect(() => {
    currentUserStateRef.current = currentUserState;
}, [currentUserState, currentUserStateRef]);

// Insert the following code before the return statement of the ServiceProvider component:
const apiV1RequestHandlerRef = useRef(createScopedApiV1RequestHandler(currentUserStateRef));

// Provide the ScopedApiV1RequestHandler with its Provider like so:
return (
    <ScopedApiV1RequestHandlerProvider value={apiV1RequestHandlerRef.current}>
        {/* Other service providers... */}
    </ScopedApiV1RequestHandlerProvider>
);
```

### 9.4 Wire the user registration form
To be able to enrich our form elements with messages, we need to define a `pathPart` for those.
It is important that we define the same `pathPart` on the specific form elements
which is delivered from the API field message, otherwise the form element state is not going to be 
enriched with the specific field message.
Let's define these `pathParts` in the `RegistrationFormState` factory function:

````typescript jsx
// src/pages/auth/RegisterPage.tsx

function createRegistrationFormState(): RegistrationFormState {
    return {
        genderSelection: createSingleSelectionState({ pathPart: ['gender'] }),
        usernameField: createTextFieldState({ pathPart: ['username'] }),
        emailField: createTextFieldState({ pathPart: ['email'] }),
        passwordField: createTextFieldState({ pathPart: ['password'] }),
        agreeCheckbox: createCheckboxState(),
    };
}
````

Next, let's add the logic to communicate with the endpoint and to enrich the `RegistrationFormState`
with the API field messages after we received the response.
As you might have seen in the mockApi code: If there are no errors in the registration form,
we directly receive the access token and user data of the newly registered user.
It would be nice to have the user logged in and redirected to the start page when a successful registration was made.
With all these requirements, we need to make the registration form look like so:

````typescript jsx
// src/pages/auth/RegisterPage.tsx

// Replace the i18n imports like so:
import { T, useTranslator } from '@packages/core/i18n';

// Add following import statements:
import { ApiV1ResponseTypes, useApiV1RequestHandler } from '@packages/core/api-v1/core';
import { registerUser } from '@packages/core/api-v1/auth';
import { useCurrentUserRepository } from '@packages/core/auth';
import { useNavigate } from 'react-router-dom';

// Add the "onSubmit" callback to the RegistrationFormProps, so that the props definition looks like so:
type RegistrationFormProps = {
    data: RegistrationFormState;
    onChangeData: (data: RegistrationFormState) => void;
    onSubmit: () => void; // Don't forget to implement it in the RegistrationForm: <Form onSubmit={props.onSubmit}>
};

// Make the RegisterPage component look like:
export const RegisterPage: FC = () => {
    const { t } = useTranslator();
    const [registrationForm, setRegistrationForm] = useState(createRegistrationFormState());
    const apiV1RequestHandler = useApiV1RequestHandler();
    const currentUserRepo = useCurrentUserRepository();
    const navigate = useNavigate();
    function canFormBeSubmitted(): boolean {
        return registrationForm.agreeCheckbox.value;
    }
    function submitForm() {
        if (!canFormBeSubmitted()) {
            return;
        }
        const newRegistrationFormState = getStateWithEnrichedFormElementStates(registrationForm, {
            messages: [],
            prefixPath: [],
        });
        setRegistrationForm(newRegistrationFormState);
        registerUser(apiV1RequestHandler, {
            username: registrationForm.usernameField.value,
            email: registrationForm.emailField.value,
            gender: registrationForm.genderSelection.chosenOption?.data as GenderId,
            password: registrationForm.passwordField.value,
        }).then((rr) => {
            if (!rr.response) {
                return;
            }
            if (rr.response.type !== ApiV1ResponseTypes.SUCCESS) {
                const newRegistrationFormState = getStateWithEnrichedFormElementStates(registrationForm, {
                    messages: rr.response.body.fieldMessages,
                    prefixPath: [],
                });
                setRegistrationForm(newRegistrationFormState);
                return;
            }
            currentUserRepo.setCurrentUser({
                type: 'authenticated',
                apiKey: rr.response.body.data.apiKey,
                data: {
                    id: rr.response.body.data.user.id,
                    username: rr.response.body.data.user.username,
                },
            });
            navigate('/');
        });
    }
    return (
        <NavBarPage title={t('pages.registerPage.title')}>
            <Typography component="h1" variant="h5">
                {t('pages.registerPage.title')}
            </Typography>
            <RegistrationForm
                data={registrationForm}
                onChangeData={(data) => setRegistrationForm(data)}
                onSubmit={() => submitForm()}
            />
            <Button
                margin="dense"
                variant="outlined"
                color="primary"
                disabled={!canFormBeSubmitted()}
                onClick={() => submitForm()}>
                {t('pages.registerPage.signUp')}
            </Button>
        </NavBarPage>
    );
};
````

Yay! Start the mockApi and the App if not done yet.
Then open the registration form in the browser, skip some form elements and submit the form.
The field error messages delivered with the API endpoint response, should show up at the form elements.

So far so good, but... wasn't there something called "general messages" delivered from the API endpoint?
Let's make these general messages dispatching a toast message for each of them.

:floppy_disk: [branch 09-wiring-1](https://github.com/inkognitro/react-app-tutorial-code/compare/08-apiv1-2...09-wiring-1)

### 9.5 Support any request handler middleware
To show the general messages of an API endpoint response, we need to write something like
a toaster middleware for the request handler. Let's try to define an interface for it:

```typescript
// src/packages/core/api-v1/core/scopedRequestHandler.ts

export type ApiV1RequestHandlerMiddleware = {
    onRequest?: (r: ApiV1Request) => void;
    onRequestResponse?: (rr: ApiV1RequestResponse) => void;
};
```

Next let's support middlewares in the `ScopedApiV1RequestHandler`, so that the class looks like below:
```typescript
// src/packages/core/api-v1/core/scopedRequestHandler.ts

export class ScopedApiV1RequestHandler implements ApiV1RequestHandler {
    private readonly middlewares: ApiV1RequestHandlerMiddleware[];
    private readonly requestHandler: ApiV1RequestHandler;
    private runningRequestIds: string[];

    constructor(requestHandler: ApiV1RequestHandler) {
        this.middlewares = [];
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

    public addMiddleware(middleware: ApiV1RequestHandlerMiddleware) {
        this.middlewares.push(middleware);
    }

    public executeRequest(settings: ApiV1RequestExecutionSettings): Promise<ApiV1RequestResponse> {
        const that = this;
        return new Promise((resolve) => {
            that.runningRequestIds.push(settings.request.id);
            that.middlewares.forEach((m) => {
                if (m.onRequest) {
                    m.onRequest(settings.request);
                }
            });
            that.requestHandler.executeRequest(settings).then((rr): void => {
                that.runningRequestIds = that.runningRequestIds.filter(
                    (requestId) => settings.request.id !== requestId
                );
                that.middlewares.forEach((m) => {
                    if (m.onRequestResponse) {
                        m.onRequestResponse(rr);
                    }
                });
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

### 9.6 Provide the toaster middleware
Now that the `ScopedApiV1RequestHandler` supports middlewares, we can write a middleware for it, which
handles the `ApiV1RequestResponse` and dispatches toast messages when required.
In case of a lost connection, the user should probably be informed about that.
The middleware fulfilling these requirements could look like so:

```typescript
// src/packages/core/api-v1/core/toasterMiddleware.ts

import { createContext, useContext } from 'react';
import { Toaster } from '@packages/core/toaster';
import { Translator } from '@packages/core/i18n';
import { ApiV1RequestHandlerMiddleware } from './scopedRequestHandler';
import { ApiV1Message, ApiV1RequestResponse } from './types';

export class ApiV1ToasterMiddleware implements ApiV1RequestHandlerMiddleware {
    private readonly toaster: Toaster;
    private readonly translator: Translator;

    constructor(toaster: Toaster, translator: Translator) {
        this.toaster = toaster;
        this.translator = translator;
    }

    onRequestResponse(rr: ApiV1RequestResponse) {
        if (rr.hasRequestBeenCancelled) {
            return;
        }
        if (!rr.response) {
            this.toaster.showMessage({
                severity: 'error',
                content: this.translator.t('core.util.connectionToServerFailed'),
            });
            return;
        }
        rr.response.body.generalMessages.map((m: ApiV1Message) =>
            this.toaster.showMessage({
                id: m.id,
                severity: m.severity,
                content: this.translator.t(m.translation.id, m.translation.placeholders),
            })
        );
    }
}

const toasterMiddlewareContext = createContext<null | ApiV1ToasterMiddleware>(null);
export const ApiV1ToasterMiddlewareProvider = toasterMiddlewareContext.Provider;

export function useNullableApiV1ToasterMiddleware(): null | ApiV1ToasterMiddleware {
    return useContext(toasterMiddlewareContext);
}
```

Don't forget to export it in the `index.ts` and to provide the `ApiV1ToasterMiddleware` in the `<ServiceProvider>`!

Furthermore, we need to support the translation for the failed connection.
Just add these translation keys like so:


```javascript
// src/components/translations/deCH.json

"core": {
    "util": {
        "connectionToServerFailed": "Verbindung zum Server konnte nicht hergestellt werden."
    },
    // other translations...
}
```

and
```javascript
// src/components/translations/enUS.json

"core": {
    "util": {
        "connectionToServerFailed": "Connection to server failed."
    },
    // other translations...
}
```

Cool! Now that we have the toaster middleware available, let's use it in the next step.

### 9.6 Add the middleware in `useApiV1RequestHandler`
Whenever we use the ApiV1RequestHandler,
it would be nice to configure whether toast messages from the responses should be shown or not.

Despite I am not a fan of implicitly doing things, I think it makes sense to show the toasts
when no hook configuration was used with the hook.

See the code below, which enhances the `useApiV1RequestHandler` hook, to better understand what I mean:
```typescript
// src/packages/core/api-v1/core/scopedRequestHandler.ts

// other stuff...

type UseApiV1RequestHandlerConfig = {
    showToasts: boolean;
};

const defaultUseApiV1RequestHandlerConfig: UseApiV1RequestHandlerConfig = {
    showToasts: true,
};

export function useApiV1RequestHandler(
    config: UseApiV1RequestHandlerConfig = defaultUseApiV1RequestHandlerConfig
): ApiV1RequestHandler {
    const toasterMiddleware = useNullableApiV1ToasterMiddleware();
    const requestHandler = useContext(scopedApiV1RequestHandlerContext);
    if (!requestHandler) {
        throw new Error('no ScopedApiV1RequestHandler was provided');
    }
    const requestHandlerRef = useRef<ScopedApiV1RequestHandler | null>(null);
    if (!requestHandlerRef.current) {
        const separatedRequestHandler = requestHandler.createSeparated();
        if (toasterMiddleware && config.showToasts) {
            separatedRequestHandler.addMiddleware(toasterMiddleware);
        }
        requestHandlerRef.current = separatedRequestHandler;
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

Congratulations! If you submit the form with skipped form elements, an
error toast will show automatically, because it is listed in the `generalMessages` property of the response body.
If you don't want having toast messages dispatched for the received general messages,
you can just use the hook with the appropriate config: `useApiV1RequestHandler({ showToasts: false });`.

:floppy_disk: [branch 09-wiring-2](https://github.com/inkognitro/react-app-tutorial-code/compare/09-wiring-1...09-wiring-2)

[« previous](08-apiv1.md) | [final words »](10-finalwords.md)