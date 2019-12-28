> WIP: this package is under construction! Use `puppeteer-request-mocker` for now.

# teremock

## Do I need that thing?

If you are writing puppeteer tests, and you want to mock your network responses easily – probably yes.

## How to use

```js
import mocker from 'teremock'

await mocker.start()

// async stuff which is making requests

await mocker.stop()
```

## How it works

First, `teremock` intercepts puppeteers page requests and tries to find corresponding responses in the working directory. Generated filename depends on request `url`, `method` and `body` – so, you always know, do you have a mock for that particular request or not. If you have it – you will get it as a response. If not – request will go to the real backend (see also: options.capturing.urls).

Second, `teremock` intercepts all responds, and writes them to the filesystem, if they are not on it already. In case of `CI` (if mock was not found), it uses mockMiss middleware, so you could be sure – all your requests are mocked (or build will fail otherwise).

> Important to note! By default, teremock intercepts all requests, including html, assets, and even fonts requests! So, technically, you could mock not only requests from your testpage, but testpage itself! To prevent that, start mocker _after_ navigtion to your testpage happened, or use `options.capture` to filter requests to be mocked. See `examples/mock-testpage-itself/` for details.

## Pipeline

<img src="assets/pipeline.svg" />

## options

```js
mocker.start(options)
```
All options are optional (that's why they called so).

```js
const options = {
  // Absolute path to working directory, where you want to store mocks
  // path.resolve(process.cwd(), '__teremocks__') by default
  wd: path.resolve(__dirname, '__teremocks__'),

  // puppeteer page
  // global.page by default
  page: page,

  // Determines which request is for mocking.
  // All responses to be mocked by default
  capture: {
    // default: ['*'] (all requests)
    urls: ['my-backend.org/used/by/test']
    // default: ['*'] (all methods)
    methods: ['get']
  },

  // If request is not to be mocked, it could be aborted or passed.
  pass: {
    urls: ['same-origin'],
    methods: ['get'],
  },

  // Some parameters, which are used when calculating mock filenames (see below `Mock files naming` section)
  naming: {
    query: {
      blacklist: [],
      whitelist: []
    },
    body: {
      blacklist: [],
      whitelist: []
    },
  }

  // Additional delay for responses. Could be usefull for stabilizing flacky tests.
  // Note: you could add `mock.meta.delay` key for any mock
  delay: 0,

  // Run as CI if true. In CI mode storage will not try to save any mocks.
  // Default is `is-ci` package value (same as in Jest)
  ci: false,

  // A middleware to call when mock is not found on the file system
  // Works only in CI mode
  // Possible values are:
  // 1) CODE (number) – respond with CODE http code for any unmocked request (e.g. 200)
  // 2) (next) => next(anyResponse) - respond with anyResponse object
  // default value is: 500
  // Note: request is not available in the middleware function
  // Note: body must be a string (use JSON.stringify for objects)
  mockMiss: (next) => next({ code: 200, body: JSON.stringify({ foo: 'bar' }) }),

  // Set true, to await all non-closed connections when trying to stop mocker
  // Warning: some tests could became flaky
  awaitConnectionsOnStop: false,

  // Custom delay between request and response for mocked responses
  // Default value is mockde ttfb value
  ttfb: () => Math.random() * 1000
}
```

### Mock files naming

The name of mock file is consist of five parts: 1) `options.wd` value 2) url directory 3) lowercased http method 4) three words 5) `.json` extension.

The most important part is `4) three words`. These words are pseudorandom, and depends on a) request url (without query) b) query params (sorted) c) body params (sorted).

@todo examples

In many cases it is important to be independent from some query and body params, which, for example, have random value for each request (e.g. timestamp). There are four different list for skipping some parameters, when calculating mock filename: whitelist and blacklist for query and body parameters. Here its typing:

```ts
type ListItem = string | string[]
type List = ListItem[]
```
For example, you want to skip `timestamp` GET-parameter, `randomToken` and nested `data.wuid` POST-parameters. Then, you need to construct two lists, and set them to mocker options:

```ts
const dynamicQueryParams = [
  'timestamp'
]
const dynamicBodyParams = [
  'randomToken',
  ['data', 'wuid']
]

mocker.start({
  naming: {
    query: {
      blacklist: dynamicQueryParams
    },
    body: {
      blacklist: dynamicBodyParams
    }
  }
})
```
Now, when you have a POST request with url and body:
```
http://example.com/?foo=bar&timestamp=123

{
  randomToken: 'qweasd',
  data: {
    alice: 'bob',
    wuid: 32
  }
}
```
the mock filename will be exact as if it were just
```
http://example.com/?foo=bar

{
  data: {
    alice: 'bob'
  }
}
```

It is not recommended to use `whitelist`, because you may encounting mocks filenames collision. But in some rare cases (for example, when some keys are random) `whitelist` could be usefull.

It is not possible to use different lists for different urls simultaneously, but if you really need that, just create an issue!

## API methods

## mocker.start()

Starts the mocker. Mocks for all requests matched `options.capture` will be used, but no mocks used before `mocker.start()` and after `mocker.stop()`

Both `mocker.start()` and `mocker.stop()` return a `Promise`.

## mocker.mock(...)

Sometimes it is more convenient to set mocks right in tests, without storing them to the file system. For that cases mocker.mock method exist.

Example:
```js
mocker.mock(filter, response)
```
After that line, all request, matched `filter`, will be mocked with `response`.

> Note: latter `mocker.mock` filters have higher priority.