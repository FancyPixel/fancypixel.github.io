---
layout: post
title: "React + Flux backed by Rails API - Part 2"
date: 2015-01-29 10:31
comments: true
categories: rails api react flux javascript 
---
This is the second part of "React + Flux backed by Rails API", make sure to checkout [Part 1](/blog/2015/01/28/react-plus-flux-backed-by-rails-api/).  

In part 1 we created our fancy Rails API, setup the authentication and defined a resource for our tiny clone of Medium. 
Time to reach the core of this post: the frontend built in React with the flux architecture. 

<!-- More -->

#Setting up the frontend
The whole idea behind splitting the backend from the fronted is to treat the web UI as a first class citizen, sitting in its own folder, in its own repo, with no bindings to the backend. The backend can be easily interchanged, as long as the API specs remain consistent. So we'll create a new app from scratch. We have the option to use automated tools like Yeoman, but I wasn't able to find the solution that fit all my needs. 

##Tools
I'll be using node's NPM to fetch the main tools, Gulp for the build and watch tasks, and Bower for the resources. 
Before diving in the details, I have to warn you, I'm awful with gulp, so I pretty much gathered tasks around the web. So take my gulp file lightly, I'm planning on fixing all the horrors as soon as I can. You might also notice that I didn't use ES6, although I wanted to. I encountered in a couple of issues while working with ES6, so being time in short supply, I switched to vanilla Javascript.  
You'll find the package.json and gulpfile.js in the [sample repo](https://github.com/FancyPixel/small-frontend), to make a long story short we'll be using `react` (obviously), `react-router`, `superagent` for the ajax calls and `flux`. 
As I said before, Flux is just an architecture, so what am I importing really in my package.json? Turns out that Facebook released a small library called `flux` that contains basically the code for a Flux Dispatcher (more on that later), that will cut down the amount of boilerplate code that we'll need to get started.

##Flux Architecture
If you already took a stab at Flux you might know this diagram:

{% img center /images/posts/2015-01-29/flux.png  640 320 'Flux architecture' %} 

It might not be easy to understand at first, but it makes more and more sense while you are implementing all those coloured blocks. Let me shed some light over it if I can.  

The leftmost block is our Web API, we built that in the previous part of this blog post, so we are set. Our API will be called by the "Web API Utils", that's just a plain JS file making ajax requests. Eventually this JS component will receive an AJAX callback, and it'll need to update our frontend app. It does that using Actions. An action is just a data structure that tells the system what happened and what payload is associated with that action.   
There are two types of actions: the one initiated by a server (e.g.: an AJAX callback) and those initiated by the views (e.g.: the user clicks a button). The difference between the two is basically semantics.  
The actions are created through Action Creators, that are really just utility functions that build the action and toss it to the system, or to be more precise, the dispatcher.  
The dispatcher is a single object (one per app) that, as the name suggests, dispatches actions to those who registered interest in them. Look at it as a pub-sub mechanism, plain and simple. 
The objects that register interest in this actions are called Stores. Stores contain the application logic and state. They are similar to a model, but they manage the state of all the objects, not a single record. 
Stores are the one offering the state that will be presented by the React views. React views should hold as few state as possible, they should grab the state of the data from a store, and pass the state to their children as props.  

That's it really, it seems rather convoluted at first, but an example can clear the fog, let's consider the login process:

- The user enters his username and password, and clicks Login
- The React view handles the click event, grabs the content of the fields and creates an action through an action creator, with the tag `LOGIN_REQUEST` and a payload with the user's credentials
- The Action creator creates the `LOGIN_REQUEST` action with its payload attached, and alerts the Dispatcher
- The Action creator also calls the Web API utils, passing the payload
- The Web API Utils perform the AJAX call
- The Rails API responds authenticating the user, providing the JSON response
- The Web API Utils receives the JSON and creates a new action, called `LOGIN_RESPONSE`, with the new JSON as payload.
- The dispatcher is notified, and forwards the action to the store(s) that is(are) interested in a `LOGIN_RESPONSE` action
- The store (e.g.: a SessionStore) gets notified and extracts the payload from the action
- The store updates its state (username, auth token and login state set to true)
- The store emits its changes
- The React views are notified of the changes and can be refreshed
- The React views can grab the state from the store, and if needed pass the state to their children

And that's it. Looks like a lot of work for a simple login, but the secret sauce that makes Flux work is that this pattern can be applied to every action performed by the user or the server. It keeps the main components decoupled, it's easier to maintain, and best of all, everything is tidy, for once.  

Ok, that was a mouthful, let's see some code.  

#Project structure
We'll start with the project structure. 

~~~
scripts
|-actions
|-components
  |-common
  |-session
  |-stories
|-constants
|-dispatcher
|-stores
|-utils
app.jsx
routes.jsx
~~~

`app.jsx` will be our mounting point, it will render the app in our html template, nothing fancy:

~~~js
// app.jsx
var React = require('react');
var router = require('./stores/RouteStore.react.jsx').getRouter();
window.React = React;

router.run(function (Handler, state) {
  React.render(<Handler/>, document.getElementById('content'));
});
~~~

That's our first taste of React and JSX. JSX is a JS extension that lets us write nodes with a syntax similar to XML. It's optional, but it cleans up the syntax and can be handled with ease by designers. 

##Routes
`router.jsx` holds all of our routes that will be used to instantiate react-router:

~~~js
// routes.jsx
var React = require('react');
var Router = require('react-router');
var Route = Router.Route;
var DefaultRoute = Router.DefaultRoute;

var SmallApp = require('./components/SmallApp.react.jsx');
var LoginPage = require('./components/session/LoginPage.react.jsx');
var StoriesPage = require('./components/stories/StoriesPage.react.jsx');
var StoryPage = require('./components/stories/StoryPage.react.jsx');
var StoryNew = require('./components/stories/StoryNew.react.jsx');
var SignupPage = require('./components/session/SignupPage.react.jsx');

module.exports = (
  <Route name="app" path="/" handler={SmallApp}>
    <DefaultRoute handler={StoriesPage} />
    <Route name="login" path="/login" handler={LoginPage}/>
    <Route name="signup" path="/signup" handler={SignupPage}/>
    <Route name="stories" path="/stories" handler={StoriesPage}/>
    <Route name="story" path="/stories/:storyId" handler={StoryPage} />
    <Route name="new-story" path="/story/new" handler={StoryNew}/>
  </Route>
);
~~~

Routes are expressed in JSX syntax, we can specify a name (that will be used to perform transitions and to create links), an handler (the React component that will be mounted when the route is visited) and an optional path (that the user will see in his address bar). As you can see we can also mount routes inside another route in a RESTful way.  

##Dispatcher
The dispatcher is the core of the app, it's the central hub for our messages (actions). It's also a fairly easy component to implement, it's mostly just boilerplate code:

~~~js
// ./dispatcher/SmallAppDispatcher.js
var SmallConstants = require('../constants/SmallConstants.js');
var Dispatcher = require('flux').Dispatcher;
var assign = require('object-assign');

var PayloadSources = SmallConstants.PayloadSources;

var SmallAppDispatcher = assign(new Dispatcher(), {

  handleServerAction: function(action) {
    var payload = {
      source: PayloadSources.SERVER_ACTION,
      action: action
    };
    this.dispatch(payload);
  },

  handleViewAction: function(action) {
    var payload = {
      source: PayloadSources.VIEW_ACTION,
      action: action
    };
    this.dispatch(payload);
  }
});

module.exports = SmallAppDispatcher;
~~~

We are basically defining two main methods that will be used to dispatch a message. We use two instead of one just for semantics: one will handle the dispatch of server-initiated action, the other one the view-initiated actions.  
Before proceeding to the meat of the implementation we'll take a look at the Constants file:

~~~js
// constants/SmallConstants.js
var keyMirror = require('keymirror');

var APIRoot = "http://localhost:3002";

module.exports = {

  APIEndpoints: {
    LOGIN:          APIRoot + "/v1/login",
    REGISTRATION:   APIRoot + "/v1/users",
    STORIES:        APIRoot + "/v1/stories"
  },

  PayloadSources: keyMirror({
    SERVER_ACTION: null,
    VIEW_ACTION: null
  }),

  ActionTypes: keyMirror({
    // Session
    LOGIN_REQUEST: null,
    LOGIN_RESPONSE: null,

    // Routes
    REDIRECT: null,

    LOAD_STORIES: null,
    RECEIVE_STORIES: null,
    LOAD_STORY: null,
    RECEIVE_STORY: null,
    CREATE_STORY: null,
    RECEIVE_CREATED_STORY: null
  })

};
~~~

This is an utility file that holds the constants that we'll use throughout the project, mainly the API endpoint and the types of action that we can perform in our app. 
Now, let's talk about the authentication process.

#Authentication
As explained in the Flux example above, the data flow will be initiated by the user, that will visit the login page, fill a form with his credentials and click on submit. We'll handle the submit as a `VIEW_ACTION`, this means that our view will call a method of our action creator for the session. Let's take a look at it:

~~~js
// ./scripts/actions/SessionActionCreators.react.jsx
var SmallAppDispatcher = require('../dispatcher/SmallAppDispatcher.js');
var SmallConstants = require('../constants/SmallConstants.js');
var WebAPIUtils = require('../utils/WebAPIUtils.js');

var ActionTypes = SmallConstants.ActionTypes;

module.exports = {

  signup: function(email, password, passwordConfirmation) {
    SmallAppDispatcher.handleViewAction({
      type: ActionTypes.SIGNUP_REQUEST,
      email: email,
      password: password,
      passwordConfirmation: passwordConfirmation
    });
    WebAPIUtils.signup(email, password, passwordConfirmation);
  },

  login: function(email, password) {
    SmallAppDispatcher.handleViewAction({
      type: ActionTypes.LOGIN_REQUEST,
      email: email,
      password: password
    });
    WebAPIUtils.login(email, password);
  },

  logout: function() {
    SmallAppDispatcher.handleViewAction({
      type: ActionTypes.LOGOUT
    });
  }
  
};
~~~

This cover all the user-initiated actions in the context of the session. The login action creator as you can see creates a new ViewAction, attaching a payload with the user's email and password, and then calls the `WebAPIUtils.login` method. If other components registered their interest in receiving the `LOGIN_REQUEST` action, the dispatcher would deliver this action right now.  
The login method of our WebAPIUtils class is this:

~~~js
// ./scripts/utils/WebAPIUtils.js
var ServerActionCreators = require('../actions/ServerActionCreators.react.jsx');
var request = require('superagent');

module.exports = {
  
  login: function(email, password) {
    request.post('http://localhost:3002/v1/login')
      .send({ username: email, password: password, grant_type: 'password' })
      .set('Accept', 'application/json')
      .end(function(error, res){
        if (res) {
          if (res.error) {
            var errorMsgs = _getErrors(res);
            ServerActionCreators.receiveLogin(null, errorMsgs);
          } else {
            json = JSON.parse(res.text);
            ServerActionCreators.receiveLogin(json, null);
          }
        }
      });
  },
  // ...
};
~~~

A common pattern should start to be apparent right now: no class is directly modifying the state of another one, but they are just creating new actions. That's the Flux way of handling data in a nutshell.  
To keep things tidy the actions for results of the login process are created in a separate action creator:

~~~js
// ./scripts/actions/ServerActionCreators.react.jsx
var SmallAppDispatcher = require('../dispatcher/SmallAppDispatcher.js');
var SmallConstants = require('../constants/SmallConstants.js');

var ActionTypes = SmallConstants.ActionTypes;

module.exports = {

  receiveLogin: function(json, errors) {
    SmallAppDispatcher.handleServerAction({
      type: ActionTypes.LOGIN_RESPONSE,
      json: json,
      errors: errors
    });
  },

 // ... 
};
~~~

And this covers the server and view actions for the login process. Who handles the result though? Let's talk about stores.

##SessionStore
Stores are like a mix between a model and a controller, they handle the data, the main state of the application, feeding the records to the views, while retrieving the data from a server. We are about to see the `SessionStore`, which keeps track of the current user (and holds his access token, used in the API calls) and listens for the `LOGIN_RESPONSE` action.

~~~js
// ./scripts/stores/SessionStore.react.jsx
var SmallAppDispatcher = require('../dispatcher/SmallAppDispatcher.js');
var SmallConstants = require('../constants/SmallConstants.js');
var EventEmitter = require('events').EventEmitter;
var assign = require('object-assign');

var ActionTypes = SmallConstants.ActionTypes;
var CHANGE_EVENT = 'change';

// Load an access token from the session storage, you might want to implement
// a 'remember me' using localSgorage
var _accessToken = sessionStorage.getItem('accessToken')
var _email = sessionStorage.getItem('email')
var _errors = [];

var SessionStore = assign({}, EventEmitter.prototype, {
  
  emitChange: function() {
    this.emit(CHANGE_EVENT);
  },

  addChangeListener: function(callback) {
    this.on(CHANGE_EVENT, callback);
  },

  removeChangeListener: function(callback) {
    this.removeListener(CHANGE_EVENT, callback);
  },

  isLoggedIn: function() {
    return _accessToken ? true : false;    
  },

  getAccessToken: function() {
    return _accessToken;
  },

  getEmail: function() {
    return _email;
  },

  getErrors: function() {
    return _errors;
  }

});

SessionStore.dispatchToken = SmallAppDispatcher.register(function(payload) {
  var action = payload.action;

  switch(action.type) {

    case ActionTypes.LOGIN_RESPONSE:
      if (action.json && action.json.access_token) {
        _accessToken = action.json.access_token;
        _email = action.json.email;
        // Token will always live in the session, so that the API can grab it with no hassle
        sessionStorage.setItem('accessToken', _accessToken);
        sessionStorage.setItem('email', _email);
      }
      if (action.errors) {
        _errors = action.errors;
      }
      SessionStore.emitChange();
      break;

    case ActionTypes.LOGOUT:
      _accessToken = null;
      _email = null;
      sessionStorage.removeItem('accessToken');
      sessionStorage.removeItem('email');
      SessionStore.emitChange();
      break;

    default:
  }
  
  return true;
});

module.exports = SessionStore;
~~~

That looks like a bunch of code, but most of it is boilerplate, the interesting part is in the `.register` function. When the store receives the `LOGIN_RESPONSE` action unpacks the payload and checks wether the login was successful or not. It then updates its state (that will be accessed by the public properties declared on top of the file) and notifies a change to whomever might be listening (that's why we import node's EventEmitter and merge the class with it).  
Ok, we have the ability to send a view action, we receive the result and store it, cool, now we need to use this store somewhere and show some UI already.

##Application
Having a store and a state brings up a tricky question: who should listen to its changes and who should use its state? Following the React philosophy we should find the component at the topmost of our view's tree, without bloating the component itself though. As far as session goes I think the best place is the root of our app. The root is the first component that is mounted by the routes, and if you take a look at our routes, that would be the component called `SmallApp`:

~~~js
  // ./scripts/components/SmallApp.react.jsx
  var React = require('react');
  var RouteHandler = require('react-router').RouteHandler;
  var Header = require('../components/Header.react.jsx');
  var SessionStore = require('../stores/SessionStore.react.jsx');
  var RouteStore = require('../stores/RouteStore.react.jsx');

  function getStateFromStores() {
    return {
      isLoggedIn: SessionStore.isLoggedIn(),
      email: SessionStore.getEmail()
    };
  }

  var SmallApp = React.createClass({

    getInitialState: function() {
      return getStateFromStores();
    },
    
    componentDidMount: function() {
      SessionStore.addChangeListener(this._onChange);
    },

    componentWillUnmount: function() {
      SessionStore.removeChangeListener(this._onChange);
    },

    _onChange: function() {
      this.setState(getStateFromStores());
    },

    render: function() {
      return (
        <div className="app">
          <Header 
            isLoggedIn={this.state.isLoggedIn}
            email={this.state.email} />
          <RouteHandler/>
        </div>
      );
    }

  });

  module.exports = SmallApp;
~~~

This is a really simple component that serves as the root layout. If you take a closer look at the render function you can see that it only renders a React component named `Header` and then mounts the content provided by the Router. The header has a couple of properties though (React `props` to be exact) and we fill them with the `SmallApp` state. Those props will be accessible within the `Header` component. The `SmallApp` state is obtained by querying the `SessionStore` in two ways:  

- by the function `getInitialState`, fired when the component is initialized
- by the `_onChange` function, called when the `Sessionstore` emits a new change.

The latter behaviour is possible since the `SmallApp` component registered its callback in the function `componentDidMount`, that is fired, you guessed it, when the component is mounted in the page.  

Small recap: the user initiates a view action, the WebAPIUtils calls the server, the server replies, a new action is raised, the dispatcher forwards it to the SessionStore, which updates its status and emits a change event, catched by the `SmallApp` component. The `SmallApp` component forwards its state to its child: the `Header` component. Wew! Let's close the circle by writing our Header:

~~~js
// ./scripts/components/Header.react.jsx
var React = require('react');
var Router = require('react-router');
var Link = Router.Link;
var ReactPropTypes = React.PropTypes;
var SessionActionCreators = require('../actions/SessionActionCreators.react.jsx');

var Header = React.createClass({

  propTypes: {
    isLoggedIn: ReactPropTypes.bool,
    email: ReactPropTypes.string
  },
  logout: function(e) {
    e.preventDefault();
    SessionActionCreators.logout();
  },
  render: function() {
    var rightNav = this.props.isLoggedIn ? (
      <ul className="right">
        <li className="has-dropdown">
          <a href="#">{this.props.email}</a>
          <ul className="dropdown">
            <li><a href='#' onClick={this.logout}>Logout</a></li>
          </ul>
        </li> 
      </ul>
    ) : (
      <ul className="right">
        <li><Link to="login">Login</Link></li>
        <li><Link to="signup">Sign up</Link></li>
      </ul>
    );

    var leftNav = this.props.isLoggedIn ? (
      <ul className="left">
        <li><Link to="new-story">New story</Link></li>
      </ul>
    ) : (
      <div></div>
    );

    return (
      <nav className="top-bar" data-topbar role="navigation">
        <ul className="title-area">
          <li className="name">
            <h1><a href="#"><strong>S</strong></a></h1>
          </li>
          <li className="toggle-topbar menu-icon"><a href="#"><span>Menu</span></a></li>
        </ul>

        <section className="top-bar-section">
          {rightNav}
          {leftNav}
        </section>
      </nav>
    );
  }
});

module.exports = Header;
~~~

As you can see we are finally defining our markup. Within this markup you might notice a couple of calls within curly braces, that references `this.props`. This object is filled with the properties declared in the previous component, so that's how a parent can forward informations down the chain of its child components. No more two way bindings, the data just flows from the root to the leaves. Also, React offers the ability to validate the props, by specifying the `propTypes`. Whenever we pass a prop to a component, React checks the data type, and raises a JS warning in the inspector's console. That's a handy debugging tool that improves the reusability of a single component.  
It's apparent how we are defining our views in a declarative way. Once defined we are not handling the state with a barrage of spaghetti jQuery calls, we keep in mind that the view will get refreshed down the line.  

It does look convolute at first, right? The Flux architecture becomes awesome once you realise that _every_ interaction follows the same principle, then everything clicks in your brain. Noticed the `logout` function in the Header? It doesn't make a reference to the SessionStore, the Header doesn't even know it exists, it just follows a pattern: "_when something happens, create an action_".  
The code is decoupled, the responsabilities are separated, we can achieve modularity and reusability. For real this time. Brilliant. 

##LoginPage
We covered the server action, but we still need to let the user perform the login action. Let's fix that. You might have guessed it: we'll create a React component that will fire and action when the user submits the form.

~~~js
// ./scripts/component/session/LoginPage.react.jsx
var React = require('react');
var SessionActionCreators = require('../../actions/SessionActionCreators.react.jsx');
var SessionStore = require('../../stores/SessionStore.react.jsx');
var ErrorNotice = require('../../components/common/ErrorNotice.react.jsx');

var LoginPage = React.createClass({
  getInitialState: function() {
    return { errors: [] };
  },
 
  componentDidMount: function() {
    SessionStore.addChangeListener(this._onChange);
  },

  componentWillUnmount: function() {
    SessionStore.removeChangeListener(this._onChange);
  },

  _onChange: function() {
    this.setState({ errors: SessionStore.getErrors() });
  },

  _onSubmit: function(e) {
    e.preventDefault();
    this.setState({ errors: [] });
    var email = this.refs.email.getDOMNode().value;
    var password = this.refs.password.getDOMNode().value;
    SessionActionCreators.login(email, password);
  },

  render: function() {
    var errors = (this.state.errors.length > 0) ? <ErrorNotice errors={this.state.errors}/> : <div></div>;
    return (
      <div>
        {errors}
        <div className="row">
          <div className="card card--login small-10 medium-6 large-4 columns small-centered">
            <form onSubmit={this._onSubmit}>
              <div className="card--login__field">
                <label name="email">Email</label>
                <input type="text" name="email" ref="email" /> 
              </div>
              <div className="card--login__field">
                <label name="password">Password</label>
                <input type="password" name="password" ref="password" />
              </div>
              <button type="submit" className="card--login__submit">Login</button>
            </form>
          </div>
        </div>
      </div>
    );
  }
});

module.exports = LoginPage;
~~~

As you can see we have our boilerplate declaration of the change callback at the start of the file, that's pretty common. You might also notice that this component has its own state, unlike the Header. This is because I felt that the login error (retrieved by the SessionStore) belongs in the page itself, since it'll be rendered there, and there's no need to let it have a broader scope by integrating it as a state in the main `SmallApp` component.  
Getting back to the login process, the render function defines the markup of our login form, and on submit the component will retrieve email and password, and create a new action with them. That's it.

#Next up
It might be a good time to take another break, in the next part we'll go through listing and posting a new story in our app, and then it's conclusions time. I'll see you soon.  

[Part 3](/blog/2015/01/30/react-plus-flux-backed-by-rails-api-part-3/)


