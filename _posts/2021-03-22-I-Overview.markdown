---
layout: post
title: 'I. Overview and Introduction'
date: 2021-03-22 20:57:16 -0400
categories: project documentation
---

Digital communication has experienced widespread growth in adoption, especially in 2020 as a response to home-working arrangements for office workers resulting from the coronavirus pandemic.
Services like Slack, Discord, and Microsoft’s Teams increased in popularity and utilization as a large portion of office workers adopted to working from home. Using services like those
mentioned provides users the ability to communicate in real-time via messaging and file-sharing, backed by a reliable service infrastructure.
The modern web provides a rich development platform, with modern browsers supporting 2d and 3d graphics, animations, single-page applications, and, most importantly, real-time communication
protocols. With minimal expertise, developers can use the resources available in modern browsers to leverage this features, and, with minimal costs, create rich applications that implement
many of the features present in modern instant messaging applications. With technology like WebRTC and the WebSocket protocol, wide browser support, and a large open-source community,
applications built for real-time communication has never been more accessible.

WebSocket is a protocol built on top of HTTP(S) that provides full-duplex communication over a single TCP connection. This bidirectional communication protocol is widely used in modern web
applications for pushing data directly from the server to the client, along with allowing the client to send data directly to the server. WebSocket is often used to provide involve live
updates on changing data to display data real-time on the UI, or, quite commonly, for instant messaging. WebSocket support has existed in most modern browsers for more than a decade, but
adoption and integration of WebSocket has increased as the web continues to grow as the de facto platform for application development. Numerous open-source libraries exist in many popular
mainstream programming language for the implementation of both clients and servers.

Motivated by the ubiquity of these (and many other) services that serve the same purpose and leveraging the utility provided by the modern browser environment,
my Senior Project involves the creation of a WebSocket-powered Instant Messaging application. For ease of development and deployment, the application is written in Node.js
(in an Object-Oriented fashion). The application will allow users to create accounts, and send messages in real-time to rooms they join. Users can share private
messages with other users within the room both users currently share.

In the application, users register to use the service, creating an account. Users can than join existing rooms or create a new room. Uniqueness of rooms is enforced.
The instant messaging approach separates messages by room; users in the ‘main’ room, for example, will not see messages sent in the ‘other’ room. Messages, however,
are globally visible to all room participants. While data are stored in MongoDB, much of it is ephemeral—only messages and users are persistently stored
(this may change, limiting the lifetime of messages), room documents in the database are used to record the state of a given room (updated upon users joining and leaving).
Rooms persist after creation, creating a limited namespace for rooms.

Considering users leaving and joining, rooms persist even when empty (as is the default state). The UI are designed in such a manner to display to users the
number of users in a given room, the current rooms, and other metrics, provided via the API. Many API routes will require authentication (involving access to rooms, or messages);
various routes for the rooms APIs may be publicly accessible to show current “occupancy” for users in rooms.

The UI includes a ‘traditional’ approach to user sign-up and sign in. Users will have a simple sign-in, sign-up forms, and join room forms. Failed login attempts/authentication attempts
are communicated back to the user in a clear and attractive manner, using a ‘toast’ or modal elements to communicate errors. The UI for the room-based instant messaging includes both a re-sizable textarea element for entering messages, an element that contains and displays the messages sent in the room (including those sent by the user), toast elements to notify users when a new user joins the room, HTML5 notifications (if approved) when the tab is not visible, and another element showing the usernames of users currently in the room.

The room-based chat is the application’s primary purpose, although the private messaging offered within rooms is at near feature parity with the primary chat, excluding ‘special’ messages
and file sharing. The private message interface is a small, toggle-able element that allows users to maintain separate threads within which to exchange private messages. Toast elements are
also displayed when a user receives a private message, alerting them that they have a new message from the sender.

Elements that display the state, like the user list and the count of users in rooms, are updated when the state of the application changes (user leaves/joins etc). The list of messages is
populated via the messages API route, showing the most recent 10 messages in order (populated on initially joining the room). When a message is sent by a user and received successfully by the
server, the message is sent to the users in the room. When a user receives a message in the room’s chat, the message is appended to the list of the messages. Users can also fetch older
messages (10 per request) via a button in the UI. These messages are prepended at the top of the thread, maintaining the order of the messages.

Persisting sessions is critically important. JSON Web Tokens are used for sessions, and stored with the corresponding user document in the database (they are also stored on the client via
secure, samesite, HTTP-Only cokies, for ease of session maintenance, as much of the application is handled in a “traditional” manner, opposed to a Single-Page approach, while being mindful of
security concerns). Users must be able to participate in several rooms simultaneously on a single session.

Modern Browser APIs are used to create a more engaging experience (CSS animations and transitions, HTML5 Notification API, etc). Considering the nature of the application, limited time are
spent trying to make the site and application mobile-friendly. All of the styling is handled independently, opposed to using frameworks for this. This is intended as both an educational
experience, and an exercise.

File sharing functionality is included as well, allowing users to share files up to 10MB in size. File uploads are stored on S3, and accessible globally—this is both done as a simple
development solution and to avoid utilizing persistent storage on the Node server. When a user opts to share a file, the file is sent to S3. Upon upload, a publicly accessible URL is returned
to the chat from the sender where the file can be accessed, by any user in the room. Files are stored permanently, with no option to delete within the application itself.

The application is written in such a manner that it aids ease of deployment. Environment variables for configuration are stored in an environment file, allowing for easy configuration of the
MongoDB instance, AWS credentials for file uploads, JWT secret, and others. This approach, while less than ideal, makes it much simpler to replicate the project, opposed to using a
‘serverless’ approach or further separating differing functionality into independent services (CDN for static assets, for example). All services used are managed services, further allowing for
a simple deployment. The MongoDB instance is deployed on MongoDB’s Atlas offering, the file storage uses AWS S3, and the application server deployment uses Heroku. Equivalent services for the
database and application server deployment could be used without issue.

Dependencies include SocketIO, a widely-used and supported library that simplifies working with the WebSocket protocol, ExpressJS for general application and API development, along with
Mongoose, an ORM for interacting with MongoDB. Front end JavaScript are written without any dependencies, as most interaction with the DOM is relatively simple, revolving around updating the
state of a few UI elements (messages, users list, and ‘toast’ style notifications upon others joining and leaving the room, and making API requests/handing WebSocket events. This is likely to
require substantially more complex code for managing state, but has the positive side-effect of smaller assets than those generated by the build systems modern frameworks use. As mentioned,
for ease of use and ease of development, front-end assets are served from the Node server, opposed to an external managed service.
