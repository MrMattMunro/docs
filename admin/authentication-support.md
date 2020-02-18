# Authentication Support

API Platform Admin delegates the authentication support to React Admin.
Refer to [the chapter dedicated to authentication in the React Admin documentation](https://marmelab.com/react-admin/Authentication.html)
for more information.

In short, you have to tweak data provider and api documentation parser, like this:

```javascript
// admin/src/App.js

import React from "react";
import { Redirect, Route } from "react-router-dom";
import { HydraAdmin, hydraDataProvider as baseHydraDataProvider, fetchHydra as baseFetchHydra } from "@api-platform/admin";
import parseHydraDocumentation from "@api-platform/api-doc-parser/lib/hydra/parseHydraDocumentation";
import authProvider from "./authProvider";

const entrypoint = process.env.REACT_APP_API_ENTRYPOINT;
const fetchHeaders = { Authorization: `Bearer ${window.localStorage.getItem("token")}` };
const fetchHydra = (url, options = {}) => baseFetchHydra(url, {
    ...options,
    headers: new Headers(fetchHeaders),
});
const apiDocumentationParser = entrypoint => parseHydraDocumentation(entrypoint, { headers: new Headers(fetchHeaders) })
    .then(
        ({ api }) => ({ api }),
        (result) => {
            switch (result.status) {
                case 401:
                    return Promise.resolve({
                        api: result.api,
                        customRoutes: [
                            <Route path="/" render={() => {
                                return window.localStorage.getItem("token") ? window.location.reload() : <Redirect to="/login" />
                            }} />
                        ],
                    });

                default:
                    return Promise.reject(result);
            }
        },
    );
const dataProvider = baseHydraDataProvider(entrypoint, fetchHydra, apiDocumentationParser);

export default () => (
    <HydraAdmin
        dataProvider={ dataProvider }
        authProvider={ authProvider }
        entrypoint={ entrypoint }
    />
);
```

The above example also requires you to define an authProvider. A very basic example authProvider using the [LexikJWTAuthenticationBundle](https://api-platform.com/docs/core/jwt/) package, a [Doctrine entity user provider](https://symfony.com/doc/current/security/user_provider.html#entity-user-provider) provided by Symfony and the [React Admin default login page](https://marmelab.com/react-admin/Authentication.html#login-configuration) could look like this:

```javascript
// admin/src/authProvider.js

export default {
    // called when the user attempts to log in
    login: ({ username, password }) => {
        const request = new Request('https://localhost:8443/authentication_token', {
            method: 'POST',
            body: JSON.stringify({ 'email': username, 'password': password }),
            headers: new Headers({ 'Content-Type': 'application/json' }),
        });
        return fetch(request)
            .then(response => {
                if (response.status < 200 || response.status >= 300) {
                    throw new Error(response.statusText);
                }
                return response.json();
            })
            .then(({ token }) => {
                localStorage.setItem('token', token);
            });        
    },
    // called when the user clicks on the logout button
    logout: () => {
        localStorage.removeItem('token');
        return Promise.resolve();
    },
    // called when the API returns an error
    checkError: ({ status }) => {
        if (status === 401 || status === 403) {
            localStorage.removeItem('token');
            return Promise.reject();
        }
        return Promise.resolve();
    },
    // called when the user navigates to a new location, to check for authentication
    checkAuth: () => {
        return localStorage.getItem('token')
            ? Promise.resolve()
            : Promise.reject();
    },
    // called when the user navigates to a new location, to check for permissions / roles
    getPermissions: () => Promise.resolve(),
};
```
