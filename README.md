## React Feedback Collection System - Emaily

Powered by Create React App, Redux, Node.js, Express, PassportJS and MongoDB.

### Steps
- cd into server folder and run `npm install` to install the dependencies
- cd into client folder and run `npm install` to install the dependencies
- run `npm run dev` to run the app server
- default port: `localhost:3000`

========================================

### App Flow

1. User signs up via OAuth (Express + MongoDB + PassportJS)
2. User pays for email credits via Stripe (Stripe + MongoDB)
3. User creates a new email campaign (React + Redux)
4. User enters list of emails to send survey (React + Redux + Redux Form)
5. Emaily sends emails to list of surveyees (Email Provider)
6. Surveyees click on link in email to provide feedback (Email Provider + Express + MongoDB)
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

The following diagram shows the relationship between Node.js and Express

![Relationship between Node.js and Express](./diagrams/express_and_node.png)
