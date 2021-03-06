# Non blocking calls with fork/join

the `yield` statement causes the generator to pause until the yielded effect resolves or rejects.
If you look closely at this example :

```javascript
function* watchFetch() {
  while ( yield take(FETCH_POSTS) ) {
    yield put( actions.requestPosts() )
    const posts = yield call(fetchApi, '/posts') // blocking call
    yield put( actions.receivePosts(posts) )
  }
}
```

the `watchFetch` generator will wait until `yield call(fetchApi, '/posts')` terminates. Imagine that the `FETCH_POSTS` action is fired from a `Refresh` button. If our application disables the button between each fetch (no concurrent fetches) then there is no issue, because we know that no `FETCH_POSTS` action will occur until we get the response from the `fetchApi` call.

But what happens if the application allows the user to click on `Refresh` without waiting for the current request to terminate ?

The following example illustrates a possible sequence of the events :

```
UI                              watchFetch
--------------------------------------------------------
FETCH_POSTS.....................call fetchApi........... waiting to resolve
........................................................
........................................................                     
FETCH_POSTS............................................. missed
........................................................
FETCH_POSTS............................................. missed
................................fetchApi returned.......
........................................................
```

When `watchFetch` is blocked on the `fetchApi` call, all `FETCH_POSTS` occurring in between the call and the response are missed.

To express non-blocking calls, we can use the `fork` function. A possible rewrite of the previous example with `fork` can be:

```javascript
import { fork, call, take, put } from 'redux-saga'

function* fetchPosts() {
  yield put( actions.requestPosts() )
  const posts = yield call(fetchApi, '/posts')
  yield put( actions.receivePosts(posts) )
}

function* watchFetch() {
  while ( yield take(FETCH_POSTS) ) {
    yield fork(fetchPosts) // non blocking call
  }
}
```

`fork`, just like `call`, accepts function/generator calls.

```javascript
yield fork(func, ...args)       // simple async functions (...) -> Promise
yield fork(generator, ...args)  // Generator functions
```

The result of `yield fork(api)` is a [Task](http://yelouafi.github.io/redux-saga/docs/api/index.html#task) object. To get the result of a forked Task
in a later time, we use the `join` function

```javascript
import { fork, join } from 'redux-saga'

function* child() { ... }

function *parent() {
  // non blocking call
  const task = yield fork(subtask, ...args)

  // ... later
  // now a blocking call, will resume with the outcome of task
  const result = yield join(task)

}
```
