Title: Redux + Roku = Redoku
Date: 2017.07.07
Summary: Making Roku development less painful by making state changes more predictable

<div class="hero-unit">
<h1>Redux + Roku = Redoku</h1>
<p>Making Roku development less painful by making state changes more predictable</p>
</div>

If you have done any JS development in the last several years, then you have no doubt encountered [Redux][]. From their own description it "helps you write applications that behave consistently". I have been doing a lot of Roku development as well, and one thing that is sorely needed is "more consistency". I liked Redux and the way it solved state management so much that I thought I should bring it over to the BrightScript/Roku world.

## Usage

In your main() function, construct an associative array that represents your app's
state structure and initial values. Pass this to `RedokuSetInitialState()` before
showing your root Scene.

In your root scene, call `RedokuRegisterReducer()` for each reducer in your app.
Note that each reducer should target a 'section' (property) of the overall state,
and that is the only portion of the state that it will be passed when called.
Each 'section' that will be handled by a reducer must be an associative array.

Ex: For a 'todos' reducer (`RedokuRegisterReducer("todos", todosReducer)`, set up
your state like this:

	{
		todos: {
			items: [...]
		},
		config: {
		}
	}

Not this:

	{
		todos: [...],  'this should not be an array
		config: {
		}
	}

After you have registered all of your reducers, call `RedokuInitialize` from your
root scene to set up the dispatch mechanisms and to trigger the initial reduction
pass.

Individual components can call any action creator functions to trigger state updates.
Action creator functions should call `RedokuDispatch()` with a single action parameter.
The action parameter should be an associative array with (at minimum) a `type`
property and also contain any other properties as appropriate. You can also call
`RedokuDispatch()` from a Task node at any point during its execution to report
asynchronous state changes.

Internally, calls to `RedokuDispatch()` will result in a call to `RedokuRunReducers`
which will loop through all registered reducers and provide them with an opportunity
to modify the state. If a given reducer does not respond to the action specified,
simply return the passed-in state parameter. If the reducer needs to modify the state,
make sure to create a copy of the state before mutating it and returning it. You
can use `RedokuClone()` to easily create a copy of the state before modifying it.

If the state changes, `m.global.state` will be updated. Any components can set
up an observer on that field and update their UI as appropriate.

## For Redux Javascript Developers <a id="redokuforjs"></a>

If you are familiar with Redux in Javascript, most of the concepts have an analog
version in Redoku.

#### store & createStore()
You initialize your 'store' by calling `RedokuSetInitialState()` and
`RedokuInitialize()`. The equivalent of `store.dispatch()` is `RedokuDispatch()`.
State is stored in the global context, so the equivalent of `store.getState()` is
to access `m.global.state`. You can set up an observer on `m.global.state` to be
notified when it changes.

#### reducers
Redoku reducers are pure functions of the type `state => (state, action)` just
like in Redux. There is no `combineReducers()` function, but instead you register
each reducer independently using `RedokuRegisterReducer()`.

#### actions & action creators
Actions and action creators are exactly the same as in Redux. Actions must have
a `type` property along with any other additional data.

#### async actions & thunk
There is no need for `thunk` or middleware in Redoku to handle async actions.
Asynchronous logic is handled by Scene Graph Tasks. There are two ways to trigger
async actions using Tasks:

  1. In the actual Task itself, dispatch your actions directly (when the task is
	  complete or at any point during its execution).
  2. In your components, spin up the Task and watch for its completion using
      `observeField` and when it is complete, dispatch your actions from your
	  component.

In both cases, the actions are just normal synchronous actions fired during/after
any asynchronous activity.

The full source code is available here: [Redoku][]

*(NOTE: If you like Redux, you probably like [React][] as well. Creating a React analog for Roku is quite a bit more complex, but stay tuned for a future blog post.)*

[Redoku]: https://github.com/briandunnington/Redoku
[Redux]: https://redux.js.org/
[React]: https://reactjs.org/
