[« previous](05-toaster.md) | [next »](07-collections.md)

## 6. Form
Within this chapter we are going to create the basic form elements for the user registration form.
We are also going to consider error messages for these form elements.

### 6.1 Learnings from the past
In my earlier React times, I thought it was a good idea to store fetched endpoint data directly
as component state to finally pass it down to child components.
This came with major drawbacks like bringing the state in the right format in every child component on every render cycle. 
Sometimes the same computation had to be done even multiple times. Not that good for performance and the
[DRY](https://de.wikipedia.org/wiki/Don%E2%80%99t_repeat_yourself) principle.

It got even worse when error messages had to be displayed for some form elements.
These error messages were saved in another state variable and had to be mapped to the right
form element in every render process.
I quickly learned that one should transform received endpoint data to a specific UI state
right after the data has been fetched. So the data is ready on every render cycle and
computation of data can be kept to a minimum while rendering the component tree.

This goes also hand in hand with the
[low coupling principle](https://stackoverflow.com/questions/14000762/what-does-low-in-coupling-and-high-in-cohesion-mean):
If the API changes for any reason, we only need to change the transformer function which takes in the
delivered data from the API and outputs the corresponding UI state.

### 6.2 Defining the `TextFieldState`
According to our learnings in the past it would be nice to have a state for every form element.
Furthermore, we should support a message container on every form element to enrich it with some (error) messages
if required.

Let's start with the message definition for the message container first:
```typescript
// src/packages/core/form/formElementState.ts
import { Translation } from '@packages/core/i18n';

export type Message = {
    id: string;
    severity: 'info' | 'success' | 'warning' | 'error';
    translation: Translation;
};
```

Inspired by [this article](https://medium.com/@k3nn7/structuring-validation-errors-in-rest-apis-40c15fbb7bc3),
it would be nice to have a function to enrich form elements with error, warning or info messages.
This should be able for a state and its nested form elements, wherever those are in the state tree.
This can be reached with a `path` property in the form element and by going down the nested state tree with
checking if a recursively composed `path` of a nested form element equals the `path` of a field message which was
delivered from an endpoint or elsewhere.
When a message `path` of the form element does match with the `path` of a field message, this means that the message
belongs to the form element. So the form element can be enriched with it.

Let's try to write this down as code:

```typescript
// src/packages/core/form/formElementState.ts

import { Translation } from '@packages/core/i18n';

// The message displayed near a form element
export type Message = {
    id: string;
    severity: 'info' | 'success' | 'error';
    translation: Translation;
};

// The (partial) path of a form element or the whole path of a field message
export type FieldMessagePath = (string | number)[];

// The message which belongs to a field or a parameter, delivered by an endpoint or something else
export type FieldMessage = {
    path: FieldMessagePath;
    message: Message;
};
```

Next let's define the state for the text field:
```typescript
// src/packages/core/form/formElementState.ts

// An enum to define the types for all form element types.
// Normally I suggest using a string union type but we need this later as an object.
// The random string after "textField-" is to make sure that the value is recognized
// as a form element type of this package.
export enum FormElementTypes {
    TEXT_FIELD = 'textField-c615d5de',
}

type GenericFormElementState<T extends FormElementTypes, P extends object = {}> = {
    type: T;
    pathPart?: FieldMessagePath;
} & P;

export type TextFieldState = GenericFormElementState<
    FormElementTypes.TEXT_FIELD,
    { value: string; messages: Message[] }
>;
```

To easily create a `TextFieldState` in the future, add a factory function.
```typescript
// src/packages/core/form/formElementState.ts

export function createTextFieldState(partial: Partial<Omit<TextFieldState, 'type'>> = {}): TextFieldState {
    return {
        messages: [],
        value: '',
        ...partial,
        type: FormElementTypes.TEXT_FIELD,
    };
}
```
Before we are going to write this magic message enrichment function, we'll define an additional form element state.

### 6.3 Defining the `CheckboxState`
Let's define an additional state for a checkbox component we are going to create next.

```typescript
// src/packages/core/form/formElementState.ts

// extend the FormElementTypes that it looks like so
export enum FormElementTypes {
    TEXT_FIELD = 'textField-c615d5de',
    CHECKBOX = 'checkbox-c615d5de',
}

export type CheckboxState = GenericFormElementState<FormElementTypes.CHECKBOX, {
    value: boolean;
    messages: Message[];
}>;

// add the factory function like we have done for the TextFieldState
export function createCheckboxState(partial: Partial<Omit<CheckboxState, 'type'>> = {}): CheckboxState {
    return {
        messages: [],
        value: false,
        ...partial,
        type: FormElementTypes.CHECKBOX,
    };
}

// Unify the form element states in one type
export type FormElementState = TextFieldState | CheckboxState;
```

### 6.4 The magic form element enrichment function
To be able to enrich any state which contains one or more `FormElementState`s somewhere in the tree
we need to write a recursive function for the message enrichment.

This one is a bit hard, so don't worry if you don't get it on your first read.
I suggest reading it bottom up to better understand what is going on:

```typescript
// src/packages/core/form/formElementStateEnrichment.ts

import { FieldMessage, FormElementState, FormElementTypes, Message } from './formElementState';

type FieldMessagePath = (string | number)[];

const FormElementTypesArray = Object.keys(FormElementTypes).map((key) => {
    // @ts-ignore
    return FormElementTypes[key];
});

function isOfTypeFormElement(anyState: any): anyState is FormElementState {
    const pseudoFormElement = anyState as FormElementState;
    if (!pseudoFormElement.type) {
        return false;
    }
    return FormElementTypesArray.includes(pseudoFormElement.type);
}

function isOfTypeFieldMessagePath(anyState: any): anyState is FieldMessagePath {
    const pseudoPath = anyState as FieldMessagePath;
    if (!Array.isArray(pseudoPath)) {
        return false;
    }
    for (let index in pseudoPath) {
        const value = pseudoPath[index];
        if (typeof value !== 'number' && typeof value !== 'string') {
            return false;
        }
    }
    return true;
}

function getMessagesByPath(fieldMessages: FieldMessage[], path: FieldMessagePath): Message[] {
    const messages: Message[] = [];
    const requiredPathString = path.join('-');
    for (let index in fieldMessages) {
        const fieldMessage = fieldMessages[index];
        if (fieldMessage.path.join('-') !== requiredPathString) {
            continue;
        }
        messages.push(fieldMessage.message);
    }
    return messages;
}

type EnrichmentSettings = {
    messages: FieldMessage[];
    prefixPath: FieldMessagePath;
};

function getWithMessagesEnrichedFormElementState<S extends FormElementState>(
    state: S,
    settings: EnrichmentSettings
): S {
    switch (state.type) {
        case FormElementTypes.TEXT_FIELD:
        case FormElementTypes.CHECKBOX:
            if (!state.pathPart) {
                return state;
            }
            const requiredPath: FieldMessagePath = [...settings.prefixPath, ...state.pathPart];
            return {
                ...state,
                messages: getMessagesByPath(settings.messages, requiredPath),
            };
        default:
            return state;
    }
}

type AnyState = {
    pathPart?: FieldMessagePath;
};

export function getStateWithEnrichedFormElementStates<S = any>(anyState: S, settings: EnrichmentSettings): S {
    if (typeof anyState !== 'object' || anyState === null) {
        return anyState;
    }
    if (isOfTypeFormElement(anyState)) {
        return getWithMessagesEnrichedFormElementState(anyState, settings);
    }
    let newState = { ...anyState };
    for (let key in anyState) {
        const subState: AnyState = anyState[key];
        const prefixPath: FieldMessagePath =
            typeof subState === 'object' &&
            subState !== null &&
            subState.pathPart &&
            !isOfTypeFieldMessagePath(subState.pathPart)
                ? [...settings.prefixPath, ...subState.pathPart]
                : settings.prefixPath;
        // @ts-ignore
        newState[key] = getStateWithEnrichedFormElementStates(subState, {
            ...settings,
            prefixPath,
        });
    }
    return newState;
}
```
Well done! Let's continue and create some components first.
Later we will test this one out on our first form we are going to create:
The user registration form.

### 6.5 Create form components
First let's create a message container which can be used for our form elements.

```typescript jsx
// src/packages/core/form/Messages.tsx

import { FC } from 'react';
import { Message as MessageState } from './formElementState';
import styled from 'styled-components';
import { T } from '@packages/core/i18n';

const StyledSpan = styled.span`
    &.error {
        color: ${({ theme }) => theme.palette.error.main};
    }
`;

type MessageProps = MessageState;

const Message: FC<MessageProps> = (props) => {
    if (props.severity === 'error') {
        return (
            <StyledSpan className="error">
                <T {...props.translation} />
            </StyledSpan>
        );
    }
    return null;
};

export type MessagesProps = {
    messages: MessageState[];
};

export const Messages: FC<MessagesProps> = (props) => {
    return (
        <>
            {props.messages.map((message) => (
                <Message key={message.id} {...message} />
            ))}
        </>
    );
};
```

Then let's write a text field component.
We are going to define an adapter component for the MUI's `TextField` component.
To easily manage the state by passing one property and one callback to our TextField component,
I suggest using a `data` and `onChangeData` property:

```typescript jsx
// src/packages/core/form/TextField.tsx

import React, { CSSProperties, FC } from 'react';
import { TextFieldState } from './formElementState';
import { TextField as MuiTextField, InputProps, FormControl } from '@mui/material';
import { Messages } from './Messages';

export type TextFieldProps = {
    data: TextFieldState;
    onChangeData?: (data: TextFieldState) => void;
    type: 'text' | 'password' | 'email';
    variant?: 'standard' | 'filled' | 'outlined';
    margin?: 'none' | 'normal' | 'dense';
    label?: string;
    name?: string;
    autoComplete?: string;
    autoFocus?: boolean;
    required?: boolean;
    fullWidth?: boolean;
    readOnly?: boolean;
    disabled?: boolean;
    size?: 'small' | 'medium';
    maxLength?: number;
    style?: CSSProperties;
    inputProps?: InputProps['inputProps'];
};

export const TextField: FC<TextFieldProps> = (props) => {
    const variant = props.variant ?? 'standard';
    const margin = props.margin ?? 'none';
    let inputProps: InputProps['inputProps'] = props.inputProps ?? undefined;
    if (props.readOnly || props.maxLength !== undefined) {
        if (!inputProps) {
            inputProps = {};
        }
        if (props.readOnly) {
            inputProps = { ...inputProps, readOnly: true };
        }
        if (props.maxLength !== undefined) {
            inputProps = { ...inputProps, maxLength: props.maxLength };
        }
    }
    return (
        <FormControl margin={margin} fullWidth={props.fullWidth}>
            <MuiTextField
                style={props.style}
                size={props.size}
                inputProps={inputProps}
                disabled={props.disabled}
                variant={variant}
                required={props.required}
                fullWidth={props.fullWidth}
                label={props.label}
                name={props.name}
                autoComplete={props.autoComplete}
                autoFocus={props.autoFocus}
                type={props.type}
                value={props.data.value}
                onChange={(event) => {
                    if (props.onChangeData) {
                        props.onChangeData({
                            ...props.data,
                            value: event.target.value,
                        });
                    }
                }}
            />
            {!props.data.messages.length ? undefined : <Messages messages={props.data.messages} />}
        </FormControl>
    );
};
```

Let's do the same for the checkbox form element:
```typescript jsx
// src/packages/core/form/Checkbox.tsx

import React, { FC, ReactNode } from 'react';
import { CheckboxState } from './formElementState';
import { Checkbox as MuiCheckbox, FormControl, FormControlLabel } from '@mui/material';
import { Messages } from './Messages';

export type CheckboxProps = {
    data: CheckboxState;
    margin?: 'dense' | 'normal' | 'none';
    onChangeData?: (data: CheckboxState) => void;
    label?: ReactNode;
    name?: string;
    autoComplete?: string;
    autoFocus?: boolean;
    required?: boolean;
    value?: string;
    color?: 'default' | 'primary' | 'secondary';
    readOnly?: boolean;
};

export const Checkbox: FC<CheckboxProps> = (props) => {
    const checkbox = (
        <MuiCheckbox
            readOnly={props.readOnly}
            required={props.required}
            name={props.name}
            autoFocus={props.autoFocus}
            checked={props.data.value}
            onChange={(event) => {
                if (props.onChangeData) {
                    props.onChangeData({
                        ...props.data,
                        value: event.target.checked,
                    });
                }
            }}
            value={props.value}
            color={props.color}
        />
    );
    if (!props.label) {
        return checkbox;
    }
    return (
        <FormControl margin={props.margin}>
            <FormControlLabel control={checkbox} label={props.label} />
            {!props.data.messages.length ? undefined : <Messages messages={props.data.messages} />}
        </FormControl>
    );
};
```

Cool! Now we have our first form elements defined.
Next we are going to add the trappings for these elements: The Form component and a button adapter.

### 6.6 Form component and button adapter
We should wrap our form elements with a `<Form>` to make sure the default behaviour like a form submit is triggered
when we press the enter key on a text field.

```typescript jsx
// src/packages/core/form/Form.tsx

import React, { CSSProperties, FC, ReactNode } from 'react';
import styled from 'styled-components';

// browsers do require a submit button inside the form element for submitting the form on element's enter key press
const InvisibleSubmitButton = styled.button`
    display: none;
`;

const StyledForm = styled.form`
    width: 100%;
`;

export type FormProps = {
    onSubmit?: () => void;
    noValidate?: boolean;
    className?: string;
    style?: CSSProperties;
    children?: ReactNode;
};

export const Form: FC<FormProps> = (props) => {
    return (
        <StyledForm
            style={props.style}
            className={props.className}
            noValidate={props.noValidate}
            onSubmit={(event) => {
                event.preventDefault();
                if (props.onSubmit) {
                    props.onSubmit();
                }
            }}>
            {props.children}
            <InvisibleSubmitButton type="submit">SUBMIT</InvisibleSubmitButton>
        </StyledForm>
    );
};
```

Finally, the user needs to have a button to trigger something. Let's write a MUI adapter for that,
like we have done before:
```typescript jsx
// src/packages/core/form/Button.tsx

import React, { FC } from 'react';
import { Button as MuiButton, ButtonProps as MuiButtonProps, FormControl } from '@mui/material';

export type ButtonProps = MuiButtonProps & {
    margin?: 'dense' | 'normal' | 'none';
    onClick?: () => void;
};

export const Button: FC<ButtonProps> = (props) => {
    let muiButtonProps: ButtonProps = { ...props };
    delete muiButtonProps.margin;
    return (
        <FormControl margin={props.margin} fullWidth={props.fullWidth}>
            <MuiButton {...muiButtonProps} />
        </FormControl>
    );
};
```
Cool! I think we are ready for creating the form, after we exported these parts in the `index.ts`!

:floppy_disk: [branch 06-form-1](https://github.com/inkognitro/react-app-tutorial-code/compare/05-toaster...06-form-1)

### 6.7 Create the registration form
Time to see the previously created components in action! So let's move on to our `RegisterPage` component and
add the registration form to it like so:

```typescript jsx
// src/pages/auth/RegisterPage.tsx

import { FC, useState } from 'react';
import { NavBarPage } from '@components/page-layout';
import { useTranslator, T } from '@packages/core/i18n';
import {
    Button,
    Checkbox,
    CheckboxState,
    createCheckboxState,
    createTextFieldState,
    Form,
    TextField,
    TextFieldState,
} from '@packages/core/form';
import { FunctionalLink } from '@packages/core/routing';
import { Typography } from '@mui/material';

type RegistrationFormState = {
    usernameField: TextFieldState;
    emailField: TextFieldState;
    passwordField: TextFieldState;
    agreeCheckbox: CheckboxState;
};

function createRegistrationFormState(): RegistrationFormState {
    return {
        usernameField: createTextFieldState(),
        emailField: createTextFieldState(),
        passwordField: createTextFieldState(),
        agreeCheckbox: createCheckboxState(),
    };
}

type RegistrationFormProps = {
    data: RegistrationFormState;
    onChangeData: (data: RegistrationFormState) => void;
};

const RegistrationForm: FC<RegistrationFormProps> = (props) => {
    const { t } = useTranslator();
    const termsAndConditionsLabel = (
        <T
            id="pages.registerPage.agreeOnTermsAndConditions"
            placeholders={{
                termsAndConditions: (
                    <FunctionalLink onClick={() => console.log('open terms and conditions')}>
                        {t('pages.registerPage.termsAndConditions')}
                    </FunctionalLink>
                ),
            }}
        />
    );
    return (
        <Form>
            <TextField
                label={t('pages.registerPage.username')}
                data={props.data.usernameField}
                onChangeData={(data) => props.onChangeData({ ...props.data, usernameField: data })}
                type="text"
                maxLength={16}
                variant="outlined"
                margin="dense"
                fullWidth
                name="username"
            />
            <TextField
                label={t('pages.registerPage.email')}
                data={props.data.emailField}
                onChangeData={(data) => props.onChangeData({ ...props.data, emailField: data })}
                type="text"
                maxLength={191}
                variant="outlined"
                margin="dense"
                fullWidth
                name="email"
            />
            <TextField
                label={t('pages.registerPage.password')}
                data={props.data.passwordField}
                onChangeData={(data) => props.onChangeData({ ...props.data, passwordField: data })}
                type="password"
                variant="outlined"
                margin="dense"
                fullWidth
                name="password"
            />
            <Checkbox
                label={termsAndConditionsLabel}
                data={props.data.agreeCheckbox}
                onChangeData={(data) => props.onChangeData({ ...props.data, agreeCheckbox: data })}
                margin="dense"
            />
        </Form>
    );
};

export const RegisterPage: FC = () => {
    const { t } = useTranslator();
    const [registrationForm, setRegistrationForm] = useState(createRegistrationFormState());
    return (
        <NavBarPage title={t('pages.registerPage.title')}>
            <Typography component="h1" variant="h5">
                {t('pages.registerPage.title')}
            </Typography>
            <RegistrationForm data={registrationForm} onChangeData={(data) => setRegistrationForm(data)} />
            <Button margin="dense" variant="outlined" color="primary">
                {t('pages.registerPage.signUp')}
            </Button>
        </NavBarPage>
    );
};
```

Let's define the used translation keys within the `pages.registerPage` property in our language files:

```json
// src/components/translations/deCH.json

"registerPage": {
    "title": "Registrieren",
    "username": "Benutzername",
    "email": "E-Mail Adresse",
    "password": "Passwort",
    "agreeOnTermsAndConditions": "Ich bin mit den {{termsAndConditions}} einverstanden.",
    "termsAndConditions": "AGB",
    "signUp": "Registrieren"
}
```

and
```json
// src/components/translations/enUS.json

"registerPage": {
    "title": "Sign up",
    "username": "Username",
    "email": "Email address",
    "password": "Password",
    "agreeOnTermsAndConditions": "I agree on the {{termsAndConditions}}.",
    "termsAndConditions": "terms and conditions",
    "signUp": "Sign up"
}
```

:floppy_disk: [branch 06-form-2](https://github.com/inkognitro/react-app-tutorial-code/compare/06-form-1...06-form-2)

> :bulb: If you like to check if everything works fine, just add following code to your codebase.
> But keep in mind: We continue the tutorial without commit the following changes:
> [branch 06-form-testout](https://github.com/inkognitro/react-app-tutorial-code/compare/06-form-2...06-form-testout)

[« previous](05-toaster.md) | [next »](07-collections.md)