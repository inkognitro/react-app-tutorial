[« previous](08-apiv1.md)

## 9. Wiring
In this chapter we are going to write a mocked api with [Express](https://www.npmjs.com/package/express)
for the register user endpoint.
In a next step we wire the form with this endpoint and take the response's field message
for form element error enrichment.

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
To use the `ApiV1RequestHandler` we need to provide it in the `ServiceContainer` like we did also for other services:

```typescript jsx
// src/ServiceContainer.tsx

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
which is delivered from the API field message, otherwise the field message will
not be filled in the state of the specific form element.
Let's define these `pathParts` this in the `RegistrationFormState` factory function:

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
As you might have seen: If there are no errors in the registration form,
we directly receive the access token and user data of the newly registered user.
It would be nice to have the user logged in and be redirected to the start page after a successful registration process.
With all these requirements, we need to take following action:

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

Yay! Start the mockApi and the App, open the registration form in the browser, skip some form elements and
submit the form: The field error messages, which come from the API endpoint, are should show right after the form elements.

So far so good, but... wasn't there something called "general messages" delivered from the API?
Let's make these general messages visible by automatically dispatching a toast message for each of them.

### 9.5 Add the toaster middleware


:floppy_disk: [branch 09-wiring-1](https://github.com/inkognitro/react-app-tutorial-code/compare/08-apiv1-2...09-wiring-1)

:floppy_disk: [branch 09-wiring-2](https://github.com/inkognitro/react-app-tutorial-code/compare/09-wiring-1...09-wiring-2)

[« previous](08-apiv1.md)