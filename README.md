# next-connect

[![npm](https://badgen.net/npm/v/next-connect)](https://www.npmjs.com/package/next-connect)
[![CircleCI](https://circleci.com/gh/hoangvvo/next-connect.svg?style=svg)](https://circleci.com/gh/hoangvvo/next-connect)
[![codecov](https://codecov.io/gh/hoangvvo/next-connect/branch/main/graph/badge.svg?token=LGIyl3Ti4H)](https://codecov.io/gh/hoangvvo/next-connect)
[![minified size](https://badgen.net/bundlephobia/min/next-connect)](https://bundlephobia.com/result?p=next-connect)
[![download/year](https://badgen.net/npm/dy/next-connect)](https://www.npmjs.com/package/next-connect)

The promise-based method routing and middleware layer for [Next.js](https://nextjs.org/) and many other frameworks.

## Features

- [Koa](https://koajs.com/)-like Async middleware
- Lightweight => Suitable for serverless environment
- 5x faster than Express.js with no overhead. Compatible with Express.js via [a wrapper](#expressjs-compatibility).
- Works with async handlers (with error catching)
- TypeScript support

## Installation

```sh
npm install next-connect@next
```

## Usage

Although `next-connect` is initially written for Next.js, it can be used in [http server](https://nodejs.org/api/http.html#httpcreateserveroptions-requestlistener), [Vercel](https://vercel.com/docs/concepts/functions/serverless-functions). See [Examples](./examples/) for more integrations.

See an example in [nextjs-mongodb-app](https://github.com/hoangvvo/nextjs-mongodb-app) (CRUD, Authentication with Passport, and more.

Below are some use cases.

### Next.js API Routes

```typescript
// pages/api/hello.js
import type { NextApiRequest, NextApiResponse } from "next";
import { createRouter } from "next-connect";

// Default Req and Res are IncomingMessage and ServerResponse
// You may want to pass in NextApiRequest and NextApiResponse
const router = createRouter<NextApiRequest, NextApiResponse>();

router
  .use(async (req, res, next) => {
    const start = Date.now();
    await next(); // call next in chain
    const end = Date.now();
    console.log(`Request took ${end - start}ms`);
  })
  .use(authMiddleware)
  .get((req, res) => {
    res.send("Hello world");
  })
  .post(async (req, res) => {
    // use async/await
    const user = await insertUser(req.body.user);
    res.json({ user });
  })
  .put(
    async (req, res, next) => {
      // You may want to pass in NextApiRequest & { isLoggedIn: true }
      // in createRouter generics to define this extra property
      if (!req.isLoggedIn) throw new Error("thrown stuff will be caught");
      // go to the next in chain
      return next();
    },
    async (req, res) => {
      const user = await updateUser(req.body.user);
      res.json({ user });
    }
  );

// create a handler from router with custom
// onError and onNoMatch
export default router.handler({
  onError: (err, req, res, next) => {
    console.error(err.stack);
    res.status(500).end("Something broke!");
  },
  onNoMatch: (req, res) => {
    res.status(404).end("Page is not found");
  },
});
```

### Next.js getServerSideProps

```jsx
// page/users/[id].js
import { createRouter } from "next-connect";

export default function Page({ user, updated }) {
  return (
    <div>
      {updated && <p>User has been updated</p>}
      <div>{JSON.stringify(user)}</div>
      <form method="POST">{/* User update form */}</form>
    </div>
  );
}

const router = createRouter()
  .use(async (req, res, next) => {
    logRequest(req);
    return next();
  })
  .get(async (req, res) => {
    const user = await getUser(req.params.id);
    if (!user) {
      // https://nextjs.org/docs/api-reference/data-fetching/get-server-side-props#notfound
      return { props: { notFound: true } };
    }
    return {
      props: { user, updated: true },
    };
  })
  .post(async (req, res) => {
    const user = await updateUser(req);
    return {
      props: { user, updated: true },
    };
  });

export async function getServerSideProps({ req, res }) {
  try {
    // we await here so that the error can be caught below
    // rather than propagate to the outer layer
    return await router.run(req, res);
  } catch (e) {
    // handle the error
    return {
      props: { error: e.message },
    };
  }
}
```

## API

### router = createRouter(options)

Create an instance Node.js router.

**options.attachParams**

Passing `true` will attach params object to req. By default, `next-connect` does not set to req.params. If `req.params` already exists, it [Object.assign](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) into `req.params`.

```js
const router = createRouter({ attachParams: true });

router.get("/users/:userId/posts/:postId", (req, res) => {
  // Visiting '/users/12/posts/23' will render '{"userId":"12","postId":"23"}'
  res.send(req.params);
});
```

### router.use(base, ...fn)

`base` (optional) - match all routes to the right of `base` or match all if omitted. (Note: If used in Next.js, this is often omitted)

`fn`(s) can either be:

- functions of `(req, res[, next])`
- **or** an instance of `next-connect`, where it will act as a sub application. `onError` and `onNoMatch` of that subapp are disregarded.

```javascript
// Mount a middleware function
router.use(async (req, res, next) => {
  req.hello = "world";
  await next(); // call to proceed to the next in chain
  console.log("request is done"); // call after all downstream handler has run
});

// Or include a base
router.use("/foo", fn); // Only run in /foo/**
```

### router.METHOD(pattern, ...fns)

`METHOD` is an HTTP method (`GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, `TRACE`) in lowercase.

`pattern` (optional) - match routes based on [supported pattern](https://github.com/lukeed/regexparam#regexparam-) or match any if omitted.

`fn`(s) are functions of `(req, res[, next])`.

```javascript
router.get("/api/user", (req, res, next) => {
  res.json(req.user);
});
router.post("/api/users", (req, res, next) => {
  res.end("User created");
});
router.put("/api/user/:id", (req, res, next) => {
  // https://nextjs.org/docs/routing/dynamic-routes
  res.end(`User ${req.query.id} updated`);
});

// Next.js already handles routing (including dynamic routes), we often
// omit `pattern` in `.METHOD`
router.get((req, res, next) => {
  res.end("This matches whatever route");
});
```

**Note:** You should understand Next.js [file-system based routing](https://nextjs.org/docs/routing/introduction). For example, having a `router.put("/api/foo", handler)` inside `page/api/index.js` _does not_ serve that handler at `/api/foo`.

### router.all(pattern, ...fns)

Same as [.METHOD](#methodpattern-fns) but accepts _any_ methods.

### router.handler(options)

Create a handler to handle incoming requests.

**options.onError**

Accepts a function as a catch-all error handler; executed whenever a handler throws an error.
By default, it responds with status code `500` and an error stack if any.

```javascript
function onError(err, req, res) {
  logger.log(err);
  // OR: console.error(err);

  res.status(500).end("Internal server error");
}

export default router.handler({ onError });
```

**Note:** exposing the error stack by default is a security risk, consider defining a custom one like the above to mitigate the risk.

**options.onNoMatch**

Accepts a function of `(req, res)` as a handler when no route is matched.
By default, it responds with a `404` status and a `Route [Method] [Url] not found` body.

```javascript
function onNoMatch(req, res) {
  res.status(404).end("page is not found... or is it!?");
}

export default router.handler({ onNoMatch });
```

### router.run(req, res)

Runs `req` and `res` through the middleware chain and returns a **promise**. It will **not** call `onError` or `onNoMatch`. It resolves with the value returned from handlers.

```js
router
  .use(async (req, res, next) => {
    return (await next()) + 1;
  })
  .use(async () => {
    return (await next()) + 2;
  })
  .use(async () => {
    return 3;
  });

console.log(await router.run(req, res));
// The above will print "6"
```

This can be useful in [`getServerSideProps`](https://nextjs.org/docs/basic-features/data-fetching#getserversideprops-server-side-rendering).

## Common errors

There are some pitfalls in using `next-connect`. Below are things to keep in mind to use it correctly.

1. **Always** `await next()`

If `next()` is not awaited, errors will not be caught if they are thrown in async handlers, leading to `UnhandledPromiseRejection`.

```javascript
// OK: we don't use async so no need to await
router
  .use((req, res, next) => {
    next();
  })
  .use((req, res, next) => {
    next();
  })
  .use(() => {
    throw new Error("💥");
  });

// BAD: This will lead to UnhandledPromiseRejection
router
  .use(async (req, res, next) => {
    next();
  })
  .use(async (req, res, next) => {
    next();
  })
  .use(async () => {
    throw new Error("💥");
  });

// GOOD: next() is await, so errors are caught properly
router
  .use(async (req, res, next) => {
    await next();
  })
  .use((req, res, next) => {
    return next(); // this works as well since it forwards the rejected promise
  })
  .use(async () => {
    throw new Error("💥");
  });
```

Another issue is that the handler would resolve before all the code in each layer runs.

```javascript
const handler = router
  .use(async (req, res, next) => {
    next(); // this is not returned or await
  })
  .get(async () => {
    // simulate a long task
    await new Promise((resolve) => setTimeout(resolve, 1000));
    res.send("ok");
    console.log("request is completed");
  })
  .handler();

await handler(req, res);
console.log("finally"); // this will run before the get layer gets to finish

// This will result in:
// 1) "finally"
// 2) "request is completed"
```

2. **DO NOT** reuse the same instance of `router` like the below pattern:

```javascript
// api-libs/base
export default createRouter().use(a).use(b);

// api/foo
import router from "api-libs/base";
export default router.get(x);

// api/bar
import router from "api-libs/base";
export default router.get(y);
```

This is because, in each API Route, the same router instance is mutated, leading to undefined behaviors.
If you want to achieve something like that, try rewriting the base instance as a factory function to avoid reusing the same instance:

```javascript
// api-libs/base
export default function createBaseRouter() {
  return createRouter().use(a).use(b);
}

// api/foo
import createBaseRouter from "api-libs/base";
export default createBaseRouter().get(x);

// api/bar
import createBaseRouter from "api-libs/base";
export default createBaseRouter().get(y);
```

3. **DO NOT** use response function like `res.(s)end` or `res.redirect` inside `getServerSideProps`.

```javascript
// page/index.js
const handler = createRouter()
  .use((req, res) => {
    // BAD: res.redirect is not a function (not defined in `getServerSideProps`)
    // See https://github.com/hoangvvo/next-connect/issues/194#issuecomment-1172961741 for a solution
    res.redirect("foo");
  })
  .use((req, res) => {
    // BAD: `getServerSideProps` gives undefined behavior if we try to send a response
    res.end("bar");
  });

export async function getServerSideProps({ req, res }) {
  await router.run(req, res);
  return {
    props: {},
  };
}
```

3. **DO NOT** use `handler()` directly in `getServerSideProps`.

```javascript
// page/index.js
const router = createRouter().use(foo).use(bar);
const handler = router.handler();

export async function getServerSideProps({ req, res }) {
  await handler(req, res); // BAD: You should call router.run(req, res);
  return {
    props: {},
  };
}
```

## Recipes

### Next.js

<details>
<summary>Match multiple routes</summary>

If you created the file `/api/<specific route>.js` folder, the handler will only run on that specific route.

If you need to create all handlers for all routes in one file (similar to `Express.js`). You can use [Optional catch-all API routes](https://nextjs.org/docs/api-routes/dynamic-api-routes#optional-catch-all-api-routes).

```javascript
// pages/api/[[...slug]].js
import { createRouter } from "next-connect";

const router = createRouter()
  .use("/api/hello", someMiddleware())
  .get("/api/user/:userId", (req, res) => {
    res.send(`Hello ${req.params.userId}`);
  });

export default router.handler();
```

While this allows quick migration from Express.js, consider seperating routes into different files (`/api/user/[userId].js`, `/api/hello.js`) in the future.

</details>

### Express.js Compatibility

<details>
<summary>Middleware wrapper</summary>

Express middleware is not built around promises but callbacks. This prevents it from playing well in the `next-connect` model. Understanding the way express middleware works, which is by calling the `next(err?)`, we can build a wrapper like the below:

```js
import someExpressMiddleware from "some-express-middleware";

const withExpressMiddleware = (middleware) => {
  return async (req, res, next) => {
    await new Promise((resolve, reject) => {
      middleware(req, res, (err) => (err ? reject(err) : resolve()));
    });
    return next();
  };
};

router.use(withExpressMiddleware(someExpressMiddleware));
```

</details>

## Contributing

Please see my [contributing.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
