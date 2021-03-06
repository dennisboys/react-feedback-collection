## React Feedback Collection System - Emaily

Powered by Create React App, React-router, Redux, Node.js, Express, PassportJS, MongoDB, Stripe, SendGrid, Localtunnel and Materialize.

### Steps
- cd into server folder and run `npm install` to install the dependencies
- cd into client folder and run `npm install` to install the dependencies
- cd into server folder and run `npm run dev` to run the app server
- default port for React server: `localhost:3000`
- default port for Express server: `localhost:5000`

### App Flow

1. User signs up via OAuth (Express + MongoDB + PassportJS)
2. User pays for email credits via Stripe (Stripe + MongoDB)
3. User creates a new email campaign (React + Redux)
4. User enters list of emails to send survey (React + Redux + Redux Form)
5. Emaily sends emails to list of surveyees (Email Provider)
6. Surveyees click on a link in email to provide feedback (Email Provider + Express + MongoDB)
7. Emaily tabulates and displays feedback (MongoDB)
8. User sees report of all survey responses (MongoDB + React + Redux)

### Application Architecture

Though we can handle all HTTP requests using Node.js itself, we will use Express framework to simplify HTTP traffic handling. Node.js is a Javascript runtime used to execute code outside of browser, while Express is a framework runs in Node.js runtime which makes dealing with HTTP traffic easier.

When users visit our domain, we are going to send them back a React app. Since all the survey data is stored in MongoDB, the React app will communicate with an Express API, which will then retrieve data from MongoDB and send them back to the React app in JSON format.

```
// a helicopter view of the app architecture

React APP
    |
    | HTTP request (JSON)
    |
 Express
    |
 MongoDB
```

### Development Environment

The app will be hosted on Heroku. Both development and production environment have a separate config file. The production environment config file can be found in `server/config/prod.js`

Since we will be utilizing Create React App to build our app, the React server comes with the tool will be responsible for delivering the front end code to browser. The Express server, however, will only handle data requests and communicate with the browser in JSON.

```
====================
       Browser
====================
    |          |
    |          |
 bundle.js   JSON
    |          |
  React     Express
 Server     Server
               |
               |
            MongoDB
```

To launch the two servers at the same time in a painless way, we would use `concurrently` package for help.

```
"scripts": {
  "start": "node index.js",
  "server": "nodemon index.js",
  "client": "npm run start --prefix client",
  "dev": "concurrently \"npm run server\" \"npm run client\""
}
```

In order to connect the React server (localhost:3000) and Express server (localhost:5000), we need to add a proxy to the create-react-app server.

```
// add this proxy proerty in the package.json file

"proxy": {
  "/auth/google": {
    "target": "http://localhost:5000"
  }
}
```

Essentially, we have set up a structure like the following. Any request trying to access a route defined in the proxy will then be automatically directed to the target domain. So any data request from the browser will be directed to Express server, however, as far as the browser concern, it is only dealing with the React server all the time. This structure cleverly bypasses the issue of cookie security concern. We would stumble upon cookie seurity and CORS issue if we serve our front end code and data in two separate servres.

```
====================
      Browser
====================
    |          |
    |          |
 bundle.js   JSON
    |          |
====================    
  React      Proxy
  Server     Setting
====================
               |
               |
====================
   Express Server
====================
```

__OAuth Flow in Development__

After setting up the proxy, our OAuth flow now looks like the following diagram.

![OAuth flow in development](./diagrams/oauth_flow_development.png)

__OAuth Flow in Production__

In production, OAuth flow is less complicated as there is no proxy involved.

![OAuth flow in production](./diagrams/oauth_flow_production.png)

### Node.js and Express

The following diagram shows the relationship between Node.js and Express

![Relationship between Node.js and Express](./diagrams/express_and_node.png)

We will set Node.js to listen to port 5000 on our local machine. When an HTTP request comes to port 5000, Node.js will hand the incoming request to Express. Upon receiving the request, Express will look through all the route handlers to determine which one should be responsible for handling the request. The route handler responsible for the request will then process it and generate a response, which will then be sent back to Node.js. Finally, Node.js will send the response back to request maker.

The following diagram takes a closer look at how Express works.

![Express Workflow](./diagrams/express_workflow.png)

When a request from browser comes to Express, it first goes through a series of middlewares for some pre-processing, and then is sent to different route handlers to generate a proper response, which will then be sent to whoever make the initial request.

### PassportJS

There are three main parts in using PassportJS.

- Requiring the module and using its passport.initialize() and passport.session() middleware with express.

- Configuring passport with at least one Strategy and setting up passport's serializeUser and deserializeUser methods.

- Specifying a route which uses the passport.authenticate middleware to actually authenticate a user.

Here is an OAuth workflow involving PassportJS.

![OAuth Workflow](./diagrams/OAuth_flow.png)

1. User clicks 'Login' button and will be directed to localhost:5000/auth/google, resulting in the execution of passport.authenticate().

2. As the passport.authenticate() is configured to handle Google strategy, PassportJS will direct user to Google with an Google client ID and client secret to ask for permission

```
/*
When we use PassportJS to handle OAuth, we have to install a general Passport library as well as a Passport strategy. 
A strategy is used to handle a specific provider, such as, Google, Facebook, etc.
*/

// inform Express to handle OAuth using PassportJS
passport.use(
  new GoogleStrategy(
    {
      clientID: keys.googleClientID,
      clientSecret: keys.googleClientSecret,
      callbackURL: '/auth/google/callback',
      proxy: true
    }, (accessToken, refreshToken, profile, done) => {
      // this callback function is called when authentication is complete
    }
  )
);

// set up the authentication route
app.get(
  '/auth/google',
  passport.authenticate('google', {
    // specify what permissions we ask for
    scope: ['profile', 'email']
  })
);
```

3. User grants permission to the application and be redirected back to the callbackURL specified in the strategy with a code

4. PassportJS Extracts the code from URL and sends request to Google with the code included

```
// After user grants permission and Google redirects the user back with a code, Passport extracts the Google code from URL and then asks Google for user details we specified with the Google code included
app.get(
  '/auth/google/callback',
  passport.authenticate('google')
);
```

5. Google sees code in URL and replies with details about the user

6. Authentication is complete and the callback function in passport.use() is executed

```
passport.use(
  new GoogleStrategy(
    {
      clientID: keys.googleClientID,
      clientSecret: keys.googleClientSecret,
      callbackURL: '/auth/google/callback',
      proxy: true
    }, (accessToken, refreshToken, profile, done) => {
      // this callback function is executed
      const existingUser = await User.findOne({ googleId: profile.id });

      if (existingUser) {
        done(null, existingUser);
      } else {
        const user = await new User({ googleId: profile.id }).save();
        done(null, user);
      }
    }
  )
);
```

7. In the callback function, the user record is retrieved and handed to passport.serializeUser() to generate a unique token. The result of serializeUser method is attached to the session as `req.session.passport.user`.

```
passport.serializeUser((user, done) => {
	done(null, user.id);
});
```
```
app.get('/api/current_user', (req, res) => {
  res.send(req.session.passport.user);
});
```

### MongoDB and Mongoose

MongoDB stores collections, which in turn stores records.

For example, here is a MongoDB database with three collections, namely, Users, Posts and Payments.

```
MongoDB Collection
----------------------------
| Users | Posts | Payments |
----------------------------
```

Each collection can have multiple records, with each record a plain Javascript object. Notice that each record can has different properties, which is in stark contrast with traditional database solutions.

```
MongoDB Record
------------------------------------
| {                  | {           |
|     "id": 1,       |    "id": 2  |
|   "name": "dennis" |   "age": 30 |
| }                  | }           |
------------------------------------
```

Mongoose matches a model class to MongoDB collection and a model instance to a MongoDB record, so we can easily manipulate MongoDB data in Javascript.

### Redux-thunk Middleware

The normal flow of redux goes like `React Component`->`Action Creator`->`Action`->`Reducers`->`Store`->`React Component`.

In redux, action creator immediately returns an action, which is then dispatched to all reducers. However, sometimes we don't want to return an action immediately, so we need a way to manipulate the behavior of action creator. Redux-thunk middleware allows us to control when to dispatch an action. When it detects a function, instead of an object, is returned from an action creator, it would kick in.

```
// the 'dispatch' parameter is the function responsible for dispatching the action to all reducers
export const fetchUser = () => {
  return function (dispatch) {
    axios.get('/api/current_user')
      .then((res) => dispatch({
        type: FETCH_USER,
        payload: res
      }))
  }
};
```

### Webhook

In development mode, we need a localtunnel to help receive data from SendGrid regarding the email click events.

```
// direct traffic received from sub-domain to port 5000
"webhook": "lt -p 5000 --subdomain subDomainName"
```

After setting up localtunnel on our machine, we should head to SendGrid to specify a URL for event notification POST request.

### Billing

__Rule of Thumb__

- Never accept raw credit card numbers
- Never store credit card numbers
- Always use an outside payment processor (Stripe in this project)

__Billing Flow__

1. User clicks 'Add Credit'
2. Inform Stripe to show a credit card form
3. User enters credit card details
4. Details sent directly from the form to Stripe
5. Stripe sends back a token representing the charge
6. We send the token to our API
7. Our API confirms the charge is successful with Stripe
8. Add credits to user's account

__Stripe Testing Account__

Email: Any Email
Number: 4242 4242 4242 4242
Date: Any Date in the Future
CVC: Any 3 Numbers
