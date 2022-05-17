[« previous](01-setup.md) | [next »](03-routing.md)

## 2. Authentication
To provide an optimal user experience we need to know who our app's current user of is.
Therefore, we need an authentication process, some sort of user object and user cache mechanism
to save the user data in our environment (e.g. browser), to keep the data even after the user has refreshed the page.
This context and storage mechanism we are going to create in this chapter of the tutorial.

### 2.1 Separate packages folder
Not only with testing in mind, but also in consideration of the principle of
[low coupling and high cohesion](https://stackoverflow.com/questions/14000762/what-does-low-in-coupling-and-high-in-cohesion-mean),
I think it makes sense to write the `auth` logic as independent as possible.
Best case scenario would be to be able to reuse the logic for another app.
This other app could be a smaller app for embedding some chosen contents in an iframe
which is rendered by an external website. Let's extract such logic into a separate `packages/` folder.

Sometimes the boundary between app logic Vs. package logic can be really opaque.
In such cases I suggest creating the logic in the app folder and moving the code to packages later,
[when you need it](https://martinfowler.com/bliki/Yagni.html).

In the case of our authentication and authorization logic we can go ahead with a separated `packages/core/auth` folder
without having a bad conscience.

### 2.2 The auth user object
```typescript
type AuthUserType = "anonymous" | "authenticated";
type GenericUser<T extends AuthUserType, Payload extends object = {}> = { type: T } & Payload;
type UserData = { id: string; username: string };
export type AnonymousAuthUser = GenericUser<"anonymous">;
export type AuthenticatedAuthUser = GenericUser<
    "authenticated",
    { apiKey: string; data: UserData }
>;
export type AuthUser = AnonymousAuthUser | AuthenticatedAuthUser;
```

[« previous](01-setup.md) | [next »](03-routing.md)