---
layout: post
title: "React + Flux backed by Rails API - Part 3"
date: 2015-01-30 15:40:43 +0100
comments: true
categories: rails api react flux javascript 
---

This is the last part of "_React + Flux backed by Rails API_", make sure to check out [Part 1](/blog/2015/01/28/react-plus-flux-backed-by-rails-api/) and [Part 2](/blog/2015/01/29/react-plus-flux-backed-by-rails-api-part-2/) if you have missed them.  

In part 1 we built a Rails API for a tiny clone of Medium, called appropriately _Small_. In part 2 we went through the setup of a React app with a Flux architecture, and built our authentication workflow. We close this series by providing the list of the stories and a creation page, protected by authentication. A note of warning: this is a quick and sometimes naive implementation, its main purpose is to show you how to get started with React and Flux, that's why we didn't put too much effort in handling possible errors or, for that matter, even showing progress indicators.

<!-- More -->
#Listing stories
Looking back at our routes, I defined a default route, mounted under `/stories`, handled by a component called `StoriesPage`. This component will abide the flux architecture, therefore its only goal in life will be displaying data fetched by a store, and re-render when something has changed. Before diving in this component, let's create a store then:

~~~js
// ./scripts/stores/StoryStore.rect.jsx
var SmallAppDispatcher = require('../dispatcher/SmallAppDispatcher.js');
var SmallConstants = require('../constants/SmallConstants.js');
var EventEmitter = require('events').EventEmitter;
var assign = require('object-assign');
var WebAPIUtils = require('../utils/WebAPIUtils.js');

var ActionTypes = SmallConstants.ActionTypes;
var CHANGE_EVENT = 'change';

var _stories = [];
var _errors = [];
var _story = { title: "", body: "", user: { username: "" } };

var StoryStore = assign({}, EventEmitter.prototype, {

  emitChange: function() {
    this.emit(CHANGE_EVENT);
  },

  addChangeListener: function(callback) {
    this.on(CHANGE_EVENT, callback);
  },

  removeChangeListener: function(callback) {
    this.removeListener(CHANGE_EVENT, callback);
  },

  getAllStories: function() {
    return _stories;
  },

  getStory: function() {
    return _story;
  },

  getErrors: function() {
    return _errors;
  }

});

StoryStore.dispatchToken = SmallAppDispatcher.register(function(payload) {
  var action = payload.action;

  switch(action.type) {
    
    case ActionTypes.RECEIVE_STORIES:
      _stories = action.json.stories;
      StoryStore.emitChange();
      break;

    case ActionTypes.RECEIVE_CREATED_STORY:
      if (action.json) {
        _stories.unshift(action.json.story);
        _errors = [];
      }
      if (action.errors) {
        _errors = action.errors;
      }
      StoryStore.emitChange();
      break;
    
    case ActionTypes.RECEIVE_STORY:
      if (action.json) {
        _story = action.json.story;
        _errors = [];
      }
      if (action.errors) {
        _errors = action.errors;
      }
      StoryStore.emitChange();
      break;
  }

  return true;
});

module.exports = StoryStore;
~~~

At this point this should look familiar, we define our private state, the properties to access it from outside, and we register our callbacks. Once a new action tagged `RECEIVE_STORIES` is dispatched, we take the content of the payload, store it in the store's state and emit a change. The `RECEIVE_STORIES` is generated when `WebAPIUtils` receives the XHR response from the server, just as the login process. What component does fire this AJAX request though? That's a view action, initiated by the user when he requests the `/stories` route. Here's the component of this route handler:

~~~js
// ./scripts/components/StoriesPage.react.jsx
var React = require('react');
var WebAPIUtils = require('../../utils/WebAPIUtils.js');
var StoryStore = require('../../stores/StoryStore.react.jsx');
var ErrorNotice = require('../../components/common/ErrorNotice.react.jsx');
var StoryActionCreators = require('../../actions/StoryActionCreators.react.jsx');
var Router = require('react-router');
var Link = Router.Link;
var timeago = require('timeago');

var StoriesPage = React.createClass({

  getInitialState: function() {
    return { 
      stories: StoryStore.getAllStories(), 
      errors: []
    };
  },
 
  componentDidMount: function() {
    StoryStore.addChangeListener(this._onChange);
    StoryActionCreators.loadStories();
  },

  componentWillUnmount: function() {
    StoryStore.removeChangeListener(this._onChange);
  },

  _onChange: function() {
    this.setState({ 
      stories: StoryStore.getAllStories(),
      errors: StoryStore.getErrors()
    }); 
  },

  render: function() {
    var errors = (this.state.errors.length > 0) ? <ErrorNotice errors={this.state.errors}/> : <div></div>;
    return (
      <div>
        {errors}
        <div className="row">
          <StoriesList stories={this.state.stories} />
        </div>
      </div>
    );
  }
});

var StoryItem = React.createClass({
  render: function() {
    return (
      <li className="story">
        <div className="story__title">
          <Link to="story" params={ {storyId: this.props.story.id} }>
            {this.props.story.title}
          </Link>
        </div>
        <div className="story__body">{this.props.story['abstract']}...</div>
        <span className="story__user">{this.props.story.user.username}</span>
        <span className="story__date"> - {timeago(this.props.story.created_at)}</span>
      </li>
      );
  }
});

var StoriesList = React.createClass({
  render: function() {
    return (
      <ul className="large-8 medium-10 small-12 small-centered columns">
        {this.props.stories.map(function(story, index){
          return <StoryItem story={story} key={"story-" + index}/>
        })}
      </ul>
    );
  }
});

module.exports = StoriesPage;
~~~

In this component I took advantage of the `componentDidMount` function to start the asynchronous request to our API. The stories are then retrieved from the aforementioned StoryStore and held in the component's state. In the `render` function I took the chance to show you a more React-y way to write a view, with modular and reusable components. These component might also live in separate files (or even projects), but for readability I put them in the same file. As you can see the page renders just one component, `StoriesList`, passing the stories list as props. The `StoriesList` uses the `.map` function to render one `StoryItem` component for each item in the stories array. Note how I needed to specify a unique key for the item (you could use an index as I did or even the story's `id`), this helps React keeping track of what element changed.  
The last element of the chain is the `StoryItem` component, that renders the props provided by its parents. You might notice the weird css class names, bear with me, I'm trying to get accustomed with the [BEM methodology](https://bem.info/method/).

#Loading a story
If everything has gone smoothly we should see a list of stories, something like this:

{% img center /images/posts/2015-01-30/list.png  648 703 'Stories list' %} 

What happens when I click a story? Looking back at our StoriesPage we have this bit of code:

~~~html
  <div className="story__title">
    <Link to="story" params={ {storyId: this.props.story.id} }>
      {this.props.story.title}
    </Link>
  </div>
~~~

and in our router we configured the route like this:

~~~js
  <Route name="story" path="/stories/:storyId" handler={StoryPage} />
~~~

This means that once we select an item from the list, the router will switch the current component (or handler) with the `StoryPage` component, passing along a query parameter named `storyId`. Our `StoryPage` will then be able to get that id, and create a new action, requesting the download of the selected item. As usual the request will be handled by the WebAPIUtils, the response will be encapsulated in a new action, handled by the store, that emits the change, ending with our `StoryPage` refreshing its content. If you followed along you should have this pattern memorized by now. If so, you have the Flux architecture figured out.  
Just for kicks, here's the StoryPage:

~~~js
var React = require('react');
var WebAPIUtils = require('../../utils/WebAPIUtils.js');
var StoryStore = require('../../stores/StoryStore.react.jsx');
var StoryActionCreators = require('../../actions/StoryActionCreators.react.jsx');
var State = require('react-router').State;

var StoryPage = React.createClass({
  
  mixins: [ State ],

  getInitialState: function() {
    return { 
      story: StoryStore.getStory(), 
      errors: []
    };
  },
 
  componentDidMount: function() {
    StoryStore.addChangeListener(this._onChange);
    StoryActionCreators.loadStory(this.getParams().storyId);
  },

  componentWillUnmount: function() {
    StoryStore.removeChangeListener(this._onChange);
  },

  _onChange: function() {
    this.setState({ 
      story: StoryStore.getStory(),
      errors: StoryStore.getErrors()
    }); 
  },
  
  render: function() {
    return (
      <div className="row">
        <div className="story__title">{this.state.story.title}</div>
        <div className="story__body">{this.state.story.body}</div>
        <div className="story__user">{this.state.story.user.username}</div>
      </div>
     );
  }

});

module.exports = StoryPage;
~~~

#Creating a new story
We have one last thing to cover, and that's user input. We already did that in the authentication process, but without going into detail. Let's take a look at the `StoryNew` component:

~~~js
// ./scripts/components/stories/StoryNew.react.jsx
var React = require('react');
var SmallAppDispatcher = require('../../dispatcher/SmallAppDispatcher.js');
var SmallConstants = require('../../constants/SmallConstants.js');
var WebAPIUtils = require('../../utils/WebAPIUtils.js');
var SessionStore = require('../../stores/SessionStore.react.jsx');
var StoryActionCreators = require('../../actions/StoryActionCreators.react.jsx');
var RouteActionCreators = require('../../actions/RouteActionCreators.react.jsx');

var StoryNew = React.createClass({

  componentDidMount: function() {
    if (!SessionStore.isLoggedIn()) { 
      RouteActionCreators.redirect('app');
    }
  },

  _onSubmit: function(e) {
    e.preventDefault();
    var title = this.refs.title.getDOMNode().value;
    var body = this.refs.body.getDOMNode().value;
    StoryActionCreators.createStory(title, body);
  },
  
  render: function() {
    return (
      <div className="row">
        <form onSubmit={this._onSubmit} className="new-story">
          <div className="new-story__title">
            <input type="text" placeholder="Title" name="title" ref="title" /> 
          </div>
          <div className="new-story__body">
            <textarea rows="10" placeholder="Your story..." name="body" ref="body" /> 
          </div>
          <div className="new-story__submit">
            <button type="submit">Create</button>
          </div>
         </form>
       </div>
     );
  }

});

module.exports = StoryNew;
~~~

In the `render` function we defined an html form with a text input and a textarea. Notice the `ref` property of those tags, its value is the binding we'll use to retrieve their content when the users submits the form, using the `getDOMNode()` function. The submit once again kickstarts the Flux chain, by creating a view action. There's one thing to note here: posting a new story is an action performed by an authenticated user, so our WebAPIUtils method will look like this:

~~~js
  createStory: function(title, body) {
    request.post('http://localhost:3002/v1/stories')
      .set('Accept', 'application/json')
      .set('Authorization', sessionStorage.getItem('accessToken'))
      .send({ story: { title: title, body: body } })
      .end(function(error, res){
        if (res) {
          if (res.error) {
            var errorMsgs = _getErrors(res);
            ServerActionCreators.receiveCreatedStory(null, errorMsgs);
          } else {
            json = JSON.parse(res.text);
            ServerActionCreators.receiveCreatedStory(json, null);
          }
        }
      });
  }
~~~
Notice how we are passing the user's access token in the request header. That's it, our Rails API will authenticate the user thanks to that header, and the action will be performed. 

#Wrap up
This concludes this tiny application. As stated above, this is a naive implementation, not meant for production, so I didn't cover too much the error handling, the user feedback when the app is loading, or the testing.  
You can find all the sources here: 

[Rails API](https://github.com/FancyPixel/small-rails)  

[React + Flux frontend](https://github.com/FancyPixel/small-frontend) 

##Flux vs. Fluxxor vs. Reflux vs...
Flux is not the only implementation of this architecture, you can find several libraries that do so, some of them are tied to React, some of them aren't, some of them don't even use the whole architecture. It's just a matter of preferences, you can checkout [this discussion](https://github.com/kriasoft/react-starter-kit/issues/22) on GitHub for more info. Personally I chose _vanilla_ flux to have a better understanding of all the underlying components, once you figure that out, there's always time to trim out the fat.

#The future of React
While I was writing this the React.js conf took place, you can find a summary of what they announced [here](http://kevinold.com/2015/01/31/takeaways-from-reactjs-conf-2015.html). The most exciting thing was probably React native. Being an iOS developer at the core I always frowned upon web frameworks with native capabilities like Phonegap or Rubymotion, but if you take a look at [this presentation](https://www.youtube.com/watch?v=7rDsRXj9-cU) it's hard not to be excited about React native. The thing that strikes me the most is that they nailed the core concept: forget "_Write once run anywhere_", embrace "_Learn once, write anywhere_".  
React is here to stay, and it's already influencing every other JS framework around. I'm excited to build future application with it and see how it's going to expand in the future.  

Until next time 

_Andrea - [@theandreamazz](https://twitter.com/theandreamazz)_

