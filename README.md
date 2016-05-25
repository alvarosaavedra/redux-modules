---
todos:
  - Why handle this in one project
---

## Redux modules

The `redux-modules` package offers a different method of handling [redux](http://redux.js.org/) module packaging.

The overall idea behind `redux-modules` is to build all of the corresponding redux functionality, constants, reducers, actions in a common location that makes it easy to understand the entire workflow common to a component of an application.

Redux modules is essentially a collection of helpers that provide functionality for common tasks related to using Redux.


## Quick take

A redux module enables us to build our redux modules in a simple manner. A redux module can be as simple as:

```javascript
/**
 * Creates a `types` object
 * with the hash of constants
 **/
const types = createConstants({
  prefix: 'TODO'
})(
  'CREATE',
  'MARK_DONE',
  'FETCH_ALL': {api: true}
);

/**
 * The root reducer that handles only create
 **/
const reducer = createReducer({
  [types.CREATE]: (state, {payload}) => ({
    ...state,
    todos: state.todos.concat(payload)
  }),

  // decorator form
  @apiHandler(types.FETCH_ALL, (apiTypes) => {
    // optional overrides
    [apiTypes.FETCH_ALL_LOADING]: (state, action) => ({...state, loading: true})
  })
  handleFetchAll: (state, action) => ({...state, todos: action.payload})

  // or non-decorator form:
  handleFetchAll: createApiHandler(types.FETCH_ALL)((state, action) => {
    return {...state, todos: action.payload};
  })
})

/**
 * Actions
 **/
const actions = createActions({
  createTodo: (text) => (dispatch) => dispatch({
    type: types.CREATE,
    payload: {text: text, done: false}
  }),

  // decorator form
  @api(types.FETCH_ALL)
  fetchAll: (client, opts) => client.get({path: '/todos'})

  // or non-decorator form
  fetchAll: createApiAction(types.FETCH_ALL)((client, opts) => {
    return () => (dispatch, getState) => client.get('/todos')
  })
})
```

In our app, our entire todo handler, reducer, and actions are all in one place in a single file. Incorporating the handler, reducer, and actions in our redux app is up to you. See [Usage in Redux](#usage-with-react-redux) for information.

## Example

For example, let's take the idea of writing a TODO application. We'll need a few actions:

* Create
* Mark done
* Fetch All

Using `redux-modules`, we can create an object that carries a unique _type_ string for each of the actions in namespace path on a type object using the `createConstants()` exported method. For instance:

```javascript
const types = createConstants('TODO')({
  'CREATE': true, // value is ignored
  'FETCH_ALL': {api: true}
})
```

<div id="constantExample"></div>

If you prefer not to use any of the api helpers included with `redux-modules`, the `createConstants()` function accepts a simple list of types instead:

```javascript
const types = createConstants('TODO')(
  'CREATE', 'MARK_DONE', 'DELETE'
)
```

## createReducer

The `createReducer()` exported function is a simple reduce handler. It accepts a single object and is responsible for passing action handling down through defined functions on a per-action basis.

The first argument object is the list of handlers, by their type with a function to be called on the dispatch of the action. For instance, from our TODO example, this object might look like:

```javascript
const reducer = createReducer({
  [types.CREATE]: (state, {payload}) => ({
    ...state,
    todos: state.todos.concat(payload)
  })
});
```

The previous object defines a handler for the `types.CREATE` action type, but does not define one for the `types.FETCH_ALL`. When the `types.CREATE` type is dispatched, the function above runs and is considered the reducer for the action. In this example, when the `types.FETCH_ALL` action is dispatched, the default handler: `(state, action) => state` is called (aka the original state).

To add handlers, we only need to define the key and value function.

## API handling

The power of `redux-modules` really comes into play when dealing with async code. The common pattern of handling async API calls, which generates multiple states.

Notice in our above example we can mark types as `api: true` within a type object and notice that it created 4 action key/values in the object which correspond to different states of an api call (loading, success, error, and the type itself). We can use this pattern to provide simple api handling to any action/reducer.

### apiMiddleware

In order to set this up, we need to add a middleware to our redux stack. This middleware helps us define defaults for handling API requests. For instance. In addition, `redux-modules` depends upon [redux-thunk](https://github.com/gaearon/redux-thunk) being available as a middleware _after_ the apiMiddleware.

For instance:

```javascript
// ...
let todoApp = combineReducers(reducers)
let apiMiddleware = createApiMiddleware({
                      baseUrl: `https://fullstackreact.com`,
                      headers: {}
                    });
let store = createStore(reducers,
            applyMiddleware(apiMiddleware, thunk));
// ...
```

The object the `createApiMiddleware()` accepts is the default configuration for all API requests. For instance, this is a good spot to add custom headers, a `baseUrl` (required), etc. Whatever we pass in here is accessible across every api client.

For more _dynamic_ requests, we can pass a function into any one of these options and it will be called with the state so we can dynamically respond. An instance where we might want to pass a function would be with our headers, which might respond with a token for every request. Another might be a case for A/B testing where we can dynamically assign the `baseUrl` on a per-user basis.

```javascript
// dynamically adding headers for _every_ request:
let apiMiddleware = createApiMiddleware({
                      baseUrl: `https://fullstackreact.com`,
                      headers: (state) => ({
                        'X-Auth-Token': state.user.token
                      })
                    });
```

The `apiMiddleware` above currently decorates an actions `meta` object. For instance:

```javascript
{
  type: 'API_FETCH_ALL',
  payload: {},
  meta: {isApi: true}
}
// passes to the `thunk` middleware the resulting action:
{
  type: 'API_FETCH_ALL',
  payload: {},
  meta: {
    isApi: true,
    baseUrl: BASE_URL,
    headers: {}
  }
}
```

### apiActions

The easiest way to take advantage of this api infrastructure is to decorate the actions with the `@api` decorator (or the non-decorator form `createApiAction()`). When calling an api action created with the `@api/createApiAction`, `redux-modules` will dispatch two actions, the loading action and the handler.

The `loading` action is fired with the type `[NAMESPACE]_loading`. The second action it dispatches is the handler function. The method that it is decorated with is expected to return a promise (although `redux-modules` will convert the response to a promise if it's not already one) which is expected to resolve in the case of success and error otherwise.

More conveniently, `redux-modules` provides a client instance of [ApiClient](https://github.com/fullstackreact/redux-modules/blob/master/src/lib/apiClient.js) for every request (which is a thin wrapper around `fetch`).

```javascript
// actions
@api(types.FETCH_ALL)
fetchAll: (client, opts) => {
  return client.get({
    path: '/news'
  }).then((json) => {
    return json;
  });
}
```

The `apiClient` includes handling for request transformations, status checking, and response transformations. We'll look at those in a minute. The `apiClient` instance includes the HTTP methods:

* GET
* POST
* PUT
* PATCH
* DELETE
* HEAD

These methods can be called on the client itself:

```javascript
client.get({})
client.post({})
client.put({})
```

When they are called, they will be passed a list of options, which includes the base options (passed by the middleware) combined with custom options passed to the client (as the first argument).

#### api client options

The `apiClient` methods accept an argument of options for a per-api request customization. The following options are available and each can be either an atomic value or a function which gets called with the api request options within the client itself _or_ in the `baseOpts` of the middleware.

When defining these options in the middleware, keep in mind that they will be available for every request passed by the `client` instance.

1. path

If a `path` option is found, `apiClient` will append the path to the `baseUrl`. If the `client` is called with a single _string_ argument, then it is considered the `path`. For instance:

```javascript
// the following are equivalent
client.get('/foo')
client.get({path: '/foo'})
// each results in a GET request to [baseUrl]/foo
```

2. url

To completely ignore the `baseUrl` for a request, we can pass the `url` option which is used for the request.

```javascript
client.get({url: 'http://google.com/?q=fullstackreact'})
```

3. appendPath

For dynamic calls, sometimes it's convenient to add a component to the path. For instance, we might want to append the url with a custom session key.

```javascript
client.get({appendPath: 'abc123'})
```

4. appendExt

The `appendExt` is primarily useful for padding extensions on a url. For instance, to make all requests to the url with the `.json` extension, we can pass the `appendExt` to json:

```javascript
client.get({appendExt: 'json'})
```

5. params

The `params` option is a query-string list of parameters that will be passed as query-string options.

```javascript
client.get({params: {q: 'fullstackreact'}})
```

6. headers

Every request can define their own headers (or globally with the middleware) by using the `headers` option:

```javascript
client.get({headers: {'X-Token': 'bearer someTokenThing'}})
```

#### transforms

The request and response transforms provide a way to manipulate requests as they go out and and they return. These are functions that are called with the `state` as well as the current request options.

#### requestTransforms

Request transforms are functions that can be defined to create a dynamic way to manipulate headers, body, etc. For instance, if we want to create a protected route, we can use a requestTransform to append a custom header.

```javascript
client.get({
  requestTransforms: [(state, opts) => req => {
    req.headers['X-Name'] = 'Ari';
    return req;
  }]
})
```

#### responseTransforms

A response transform handles the resulting request response and gives us an opportunity to transform the data to another format on the way in. The default response transform is to respond with the response body into json. To handle the actual response, we can assign a responseTransform to overwrite the default json parsing and get a handle on the actual fetch response.

```javascript
let timeTransform = (state, opts) => res => {
  res.headers.set('X-Response-Time', time);
  return res;
}
let jsonTransform = (state, opts) => (res) => {
  let time = res.headers.get('X-Response-Time');
  return res.json().then(json => ({...json, time}))
}
client.get({
  responseTransforms: [timeTransform, jsonTransform]
})
```

For apis that do not respond with json, the `responseTransforms` are a good spot to handle conversion to another format, such as xml.

### apiHandlers

To handle api responses in a reducer, `redux-modules` provides the `apiHandler` decorator (and it's non-decorator form: `createApiHandler()`). This decorator provides a common interface for handling the different states of an api request (i.e. `loading`, `success`, and `error` states).

```javascript
@apiHandler(types.FETCH_ALL)
handleFetchAll: (state, {payload}) => {...state, ...payload}
```

The decorated function is considered the _success_ function handler and will be called upon a success status code returned from the request.

To handle custom loading states, we can "hook" into them with a second argument. The second argument is a function that's called with the dynamic states provided by the argument it's called with. For instance, to handle custom handling of an error state:

```javascript
@apiHandler(types.FETCH_ALL, (apiTypes) => ({
  [apiTypes.ERROR]: (state, {payload}) => ({
    ...state,
    error: payload.body.message,
    loading: false
  })
}))
handleFetchAll: (state, {payload}) => {...state, ...payload}
```

## Usage with react-redux

There are multiple methods for combining `redux-modules` with react and this is our opinion about how to use the two together.

First, our directory structure generally sets all of our modules in their own directory:

```bash
index.js
  /redux
    /modules/
      todo.js
      users.js
    configureStore.js
    rootReducer.js
    index.js
```

Configuring the store for our app is straight-forward. First, we'll apply the `createApiMiddleware()` before we create the final store. In a `configureStore.js` file, we like to handle creating a store in a single spot. We'll export a function to configure the store:

```javascript
import {rootReducer, actions} from './rootReducer'

export const configureStore = ({initialState = {}}) => {
  let middleware = [
    createApiMiddleware({
      baseUrl: BASE_URL
    }),
    thunkMiddleware
  ]
  // ...
  const finalCreateStore =
        compose(applyMiddleware(...middleware))(createStore);

  const store = finalCreateStore(rootReducer, initialState);
  // ...
}
```

This creates the middleware for us. Next, we like to combine our actions into a single actions object that we'll pass along down through our components. Although this isn't super elegant, we use the following snippet to bind our actions to the store. Just after we create the store, we'll:

```javascript
// create actions here
Object.keys(actions).forEach(k => {
  let theseActions = actions[k];

  let actionCreators = Object.keys(theseActions)
    .reduce((sum, actionName) => {
      // theseActions are each of the module actions which
      // we export from the `rootReducer.js` file (we'll create shortly)
      let fn = theseActions[actionName];
      // We'll bind them to the store
      sum[actionName] = fn.bind(store);
      return sum;
    }, {});

  // Using react-redux, we'll bind all these actions to the
  // store.dispatch
  actions[k] = bindActionCreators(actionCreators, store.dispatch);
});
```
From here, we just return the store and actions from the function:

```javascript
export const configureStore = ({initialState = {}}) => {
  // ...
  const store = finalCreateStore(rootReducer, initialState);
  // ...
  actions[k] = bindActionCreators(actionCreators, store.dispatch);

  return {store, actions};
}
```

Now that the heavy-lifting is done, the `rootReducer.js` file is pretty simple. We export all the actions and reducers pretty simply:

```javascript
const containers = ['users', 'todos'];

export const reducers = {}
export const actions = {};

containers.forEach(k => {
  let val = require(`./modules/${v}`);
  reducers[k] = val.reducer;
  actions[k] = val.actions || {};
});

export const rootReducer = combineReducers(reducers);
```

From here, our main container can pass the store and actions as props to our components:

```javascript
const {store, actions} = configureStore({initialState});
// ...
ReactDOM.render(
  <Container store={store}, actions={actions} />,
  node);
```

## Combining usage with `ducks-modular-redux`


## All exports

The `redux-modules` is comprised by the following exports:

### createConstants

`createConstants()` creates an object to handle creating an object of type constants. It allows for multiple

### createReducer

### apiClient

### createApiMiddleware

### createApiAction/@api

### createApiHandler/@apiHandler