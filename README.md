# Flux

Following are **proposed** guidelines for using React with Flux.


## A Loose Pattern

"Flux" is a loose pattern. Every company represented at React.js
Conf 2015 builds their Flux apps in a slightly different way, and
even different groups within Facebook didn't agree on a single way
to implement Flux apps.

Because of that, you can take the following guidelines fairly
loosely and adapt to your own needs. This document is likely to
change over time as our understanding of the Flux pattern improves.

That being said, the following guidelines represent our best
understanding right now. If you don't have a good reason to
deviate, then don't.


## When do I **not** need Flux?

If the feature you're building can be made of a few small components,
with little nesting, then a simple tree of React components, with
the root component managing state and passing props may be sufficient
for your needs.

You can use `react_component` and a simple props object at the
root of your tree. If you're using this style, it's usually best
to use regular "Rails" ways to persist the data to the server
rather than $.ajax you manage yourself. You can use a form post, or
a "remote" form post via standard Rails mechanisms.

However, you may need a tighter feedback loop between the server
and the user. In that case -- if you're needing to make ajax
calls to the server to update component state, or if you need to
access callbacks on parent components, please consider using Flux
to build your feature.

In our experience, switching to Flux can take as few as
**10 minutes** if done early in the process. If, however, you wait
until the pain of managing multiple levels of state and callbacks
sets in, switching to the Flux pattern can easily take
**over an hour.**


## When do I need Flux?

The following questions can help you decide if you need the
Flux pattern instead:

* Do you have more than 5 components?
* Are components nested more than 2 levels deep, i.e.
  do you have grandchildren components?
* Do child components need to access callbacks on the parent
  component?
* Do components make ajax calls to the server?

If you answered *yes* to two or more of these questions, chances are
good you need Flux.


## Anti-Pattern: Stores + Components Only

This is discouraged. It may seem like moving your props and state
into a store (and stopping there) is "good enough."

This style doesn't address callbacks very well and leads to stores
that know too much, components that are tightly coupled,
and is usually "just enough" structure to make developers who come
along later resistant to refactoring it all -- instead they'll layer
more complexity on top of this.


## Trust the Flow

Study this diagram. Create each of the large boxes as files within
your application. (The smaller boxes represent communication paths --
not files.)

![Flux Pattern](https://raw.githubusercontent.com/facebook/flux/master/docs/img/flux-diagram-white-background.png)

Every time you call a method from one box to another, make sure you
are honoring the flow direction. If you are going the wrong way,
then stop and think about which box in this diagram you're not
building properly.


## Dispatcher

A single dispatcher can be used for multiple, distinct features.
We don't yet know of any reason to create multiple dispatchers
within the same application, but use your own judgement.

The dispatcher acts as a simple event delegator. Don't be tempted
to make it smarter than that. **No business logic goes here.**

### TL;DR

* single dispatcher per app
* very dumb - just copy-and-paste this from another app or use McFly
* no business logic


## Stores

A store is a place to dump blobs of data. This data is almost always
provided by your server-side application, either via bootstrapping
in the view, or via an ajax call upon page load.

Data in a store should generally be related to a single concern,
but is not necessarily the same thing as a Rails "model" (though
that's sometimes what we default to).

The data in a store need not always match the state of your
server-side database. It is often helpful to mutate data in your
store in order to represent the user's intention, then push that
data to "persist" it to the server.

The following methods are common on a store:

* `getThing()`         - retrieves a thing from the store
* `updateThing(thing)` - updates data in the store (**no ajax!**)

The following methods should **not** be present in a store:

* `saveThing()` or `persistThing()` - saves to the server
* `refreshThing()` or `revertThing()` - reloads from the server

(These methods should be in Action Creators.)

### Singleton

A store is a singleton object.

It's ok for a store to hold multiple "things" -- in that case your
methods should take an `id` so you can retreive/update the right
chunk of data.

### Separation of Concerns

Each store should meet the need of a single concern if possible.
The following are examples of good stores:

* `PersonStore` (People)  - holds data for the person on the page
* `CountdownStore` (Live) - mutates every second to tell the app
                            that a second has passed
* `FieldsStore` (People)  - holds all the fields being edited
                            on the customizer page

### No Business Logic

**Do not put business logic in any method on a store that mutates
the store data.**

If you need business logic in your setters, then that should be moved
into your Action Creators instead.

### Presentation Logic

It is ok to have additional "getter" methods on your store that
return data that is tailored to a particular view or component need.
This can be considered view-related business logic.

An example might be a `CountdownStore.isItemBehind(item)`.

Another option is to put that view-related business logic in your
Rect components, but we believe it is generally better to put it
in your getter methods on the store so it's reusable by other
components.

### TL;DR

* very dumb
* singleton object
* no business logic
* view-related presentation logic can go in getters


## Action Creators

Create action creators that your components talk to. The action
creators build events that feed through the Dispatcher and
subsequently into your stores.

It's *probably* best to have an action creator object for each of
your stores, but our experience here is still limited. Use your best
judgement, and build your action creators to meet the needs of
"concerns" when possible.

Action Creators (ACs) should be very simple.

### TL;DR

* ACs can read data from stores, but never manipulate it directly
* ACs cannot make ajax calls -- they must call Utilities for that
* ACs can contain business logic in terms of how they manipulate
  your store data (remember, via the dispatcher -- not directly)


## Utilities

These make ajax calls and talk to your server and the outside
world.

## React Components

When using Flux, you can follow most (if not all) of the guidelines
in [React Patterns](https://github.com/planningcenter/react-patterns/blob/master/README.md).
But also, you should be sure to separate out state and callbacks
as dictated by the pattern.

Here is an example:


```javascript
var PersonStore = require('stores/person_store');
var PersonActions = require('actions/person_actions');

function getState() {
  var person = PersonStore.getPerson();
  return {
    first_name: person.first_name,
    last_name: person.last_name,
    editing: person.editingName
  };
}

var NameComponent = React.createClass({
  getInitialState: function() {
    return getState();
  },

  componentDidMount: function() {
    PersonStore.addEventListener(this.handleChange);
  },

  componentWillUnmount: function() {
    PersonStore.removeEventListener(this.handleChange);
  },

  render: function() {
    return (
      <div className="person-name">
        {
          if(this.state.editing) {
            this.renderNameFields();
          } else {
            this.renderNameHeading();
          }
        }
      </div>
    );
  },

  renderNameHeading: function() {
    return (
      <div>
        <span>{first_name} {last_name}</span>
        <button onClick: {this.handleEditButtonClick}>
          edit
        </button>
      </div>
    );
  },

  renderNameFields: function() {
    return (
      <div>
        <input ref='first_name'
               defaultValue={this.state.first_name}
               onChange={this.handleUpdate}>
        <input ref='last_name'
               defaultValue={this.state.last_name}
               onChange={this.handleUpdate}>
        <button onClick: {this.handleSaveButtonClick}>
          save
        </button>
      </div>
    );
  },

  handleEditButtonClick: function() {
    PersonActions.editPerson();
  },

  handleUpdate: function() {
    PersonActions.updatePerson({
      first_name: this.refs.first_name.getDOMNode().value,
      last_name: this.refs.first_name.getDOMNode().value
    });
  },

  handleSaveButtonClick: function() {
    PersonActions.persistPerson();
  },

  handleChange: function() {
    this.setState(getState());
  }

});

module.exports = NameComponent;
```

Notice some conventions in this example:

* Set the state from the store once, and each time the store updates.
* Don't query the store inside your `render()` methods.
* Set individual scalar properties in state -- not the entire
  `person` object. It makes your components simpler and less reliant
  on the structure of objects.
* Store the "editing" state of a component in the store if possible.
  This makes it possible for other components to know when data
  is being edited and eliminates the need to pass state between
  components.
* Even the transient changes on the name fields are saved to the
  store. This:
  * removes transient state management from the component
  * allows other components to share data
* If you don't want other components to see the transient changes
  until the data is "saved", then you can store the the changes
  in a "updatingPerson" object on the store.
* To reiterate, no state is managed by the component itself,
  and the store is the single source of truth.
* Resist the urge to pass callbacks down from parent components --
  instead, use Action Creators to mutate state (data in the store).
