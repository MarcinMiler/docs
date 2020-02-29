---
description: >-
  Routing determines how an application responds to a client request to a
  particular endpoint, which is a path and a specific HTTP method (eg. GET,
  POST).
---

# Routing

## Route composition

As we know - every API requires composable routing. Lets assume that we have a separate **User** feature where its API endpoints respond to _GET_ and _POST_ methods on `/user` path.

{% hint style="info" %}
Since Marble.js. v2.0, you can choose between two ways of defining HTTP routes - using [`EffectFactory`](../api-reference/core/core-effectfactory.md) or using [`r.pipe`](../api-reference/core/r.pipe.md) operators. In this example we will stick to the second, newer way, which has a more functional flavor and is more composable.
{% endhint %}

`r.pipe` is an indexed monad builder used for collecting information about Marble REST route details, like: _path_, request _method type_, _middlewares_ and connected _Effect_.

{% tabs %}
{% tab title="user.effects.ts" %}
```typescript
import { combineRoutes, r } from '@marblejs/core';

const getUsers$ = r.pipe(
  r.matchPath('/'),
  r.matchType('GET'),
  r.useEffect(req$ => req$.pipe(
    // ...
  )));

const postUser$: r.pipe(
  r.matchPath('/'),
  r.matchType('POST'),
  r.useEffect(req$ => req$.pipe(
    // ...
  )));

export const user$ = combineRoutes(
  '/user',
  [ getUsers$, postUser$ ],
);
```
{% endtab %}
{% endtabs %}

Defined _HttpEffects_ can be grouped together using `combineRoutes` function, which combines routing for a prefixed path passed as a first argument. Exported group of _Effects_ can be combined with other _Effects_ like in the example below.

{% tabs %}
{% tab title="api.effects.ts" %}
```typescript
import { combineRoutes, r } from '@marblejs/core';
import { user$ } from './user.effects';

const root$ = r.pipe(
  r.matchPath('/'),
  r.matchType('GET'),
  r.useEffect(req$ => req$.pipe(
    // ...
  )));

const foo$ = r.pipe(
  r.matchPath('/foo'),
  r.matchType('GET'),
  r.useEffect(req$ => req$.pipe(
    // ...
  )));

export const api$ = combineRoutes(
  '/api/v1',
  [ root$, foo$, user$ ],
);
```
{% endtab %}
{% endtabs %}

As you can see, the previously defined routes can be combined together, so as a result the routing is built in a much more structured way. If we analyze the above example, the routing will be mapped to the following routing table.

```text
GET    /api/v1
GET    /api/v1/foo
GET    /api/v1/user
POST   /api/v1/user
```

There are some cases where there is a need to compose a bunch of middlewares before grouped routes, e.g. to authenticate requests only for a selected group of endpoints. Instead of composing middlewares using [use operator](../api-reference/core/operator-use.md) for each route separately, you can compose them via the extended second parameter in`combineRoutes()` function.

```typescript
const user$ = combineRoutes('/user', {
  middlewares: [authorize$],
  effects: [getUsers$, postUser$],
});
```

## Body parameters

Marble.js doesn't come with a built-in mechanism for parsing _POST_, _PUT_ and _PATCH_ request bodies. In order to get the parsed request body you can use dedicated _@marblejs/middleware-body_ package. A new `req.body` object containing the parsed data will be populated on the request object after the middleware, or undefined if there was no body to parse, the `Content-Type` was not matched, or an error occurred. To learn more about body parsing middleware visit the [@marblejs/middleware-body](../api-reference/middleware-body.md) API specification.

{% hint style="danger" %}
All properties and values in `req.body`object are untrusted and should be validated before usage.
{% endhint %}

{% hint style="danger" %}
By design, the `req.body, req.params, req.query`are of type `unknown`. In order to work with decoded values you should validate them before \(e.g. using dedicated validator middleware\) or explictly assert attributes to the given type. We highly recommend to use the [@marblejs/middlware-io](../api-reference/middleware-io.md) package which allows you to properly infer the type of validated properties.
{% endhint %}

## URL parameters

The `combineRoutes` function and the `matchPath` allows you to define parameters in the path argument. All parameters are defined by the syntax with a colon prefix.

```typescript
const foo$ = r.pipe(
  r.matchPath('/:foo/:bar'),
  // ...
);
```

Decoded path parameters are placed in the `req.params` property. If there are no decoded URL parameters then the property contains an empty object. For the above example and route `/bob/12` the `req.params` object will contain the following properties:

```typescript
{
  foo: 'bob',
  bar: '12',
}
```

For parsing and decoding URL parameters, Marble.js makes use of [`path-to-regexp`](https://github.com/pillarjs/path-to-regexp) library.

{% hint style="danger" %}
All properties and values in `req.params` object are untrusted and should be validated before usage.
{% endhint %}

{% hint style="info" %}
You should validate incoming URL params using dedicated [requestValidator$](../api-reference/middleware-io.md) middleware.
{% endhint %}

Path parameters can be suffixed with an asterisk \(`*`\) to denote a zero or more parameter matches. The code snippet below shows an example use case of a "zero-or-more" parameter. For example, it can be useful for defining routing for static assets.

{% tabs %}
{% tab title="getFile.effect.ts" %}
```typescript
const getFile$ = r.pipe(
  r.matchPath('/:dir*'),
  r.matchType('GET'),
  r.useEffect(req$ => req$.pipe(
    // ...
    map(req => req.params.dir),
    mergeMap(readFile(STATIC_PATH)),
    map(body => ({ body }))
  )));
```
{% endtab %}
{% endtabs %}

## Query parameters

Except intercepting URL params, the routing is able to parse query parameters provided in path string. All decoded query parameters are located inside `req.query` property. If there are no decoded query parameters then the property contains an empty object. For parisng and decoding query parameters, Marble.js makes use of [`qs`](https://github.com/ljharb/qs) libray.

**Example 1:**

```text
GET /user?name=Patrick
```

```typescript
req.query = {
  name: 'Patrick',
};
```

**Example 2:**

```text
GET /user?name=Patrick&location[country]=Poland&location[city]=Katowice
```

```typescript
req.query = {
  name: 'Patrick',
  location: {
    country: 'Poland',
    city: 'Katowice',
  },
};
```

{% hint style="danger" %}
All properties and values in `req.query` object are untrusted and should be validated before usage.
{% endhint %}

{% hint style="info" %}
You should validate incoming `req.query` parameters using dedicated[ requestValidator$](../api-reference/middleware-io.md) middleware.
{% endhint %}

