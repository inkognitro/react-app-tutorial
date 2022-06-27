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

### 9.2 Provide the ApiV1RequestHandler
TBD

:floppy_disk: [branch 09-wiring-1](https://github.com/inkognitro/react-app-tutorial-code/compare/08-apiv1-2...09-wiring-1)

:floppy_disk: [branch 09-wiring-2](https://github.com/inkognitro/react-app-tutorial-code/compare/09-wiring-1...09-wiring-2)

[« previous](08-apiv1.md)