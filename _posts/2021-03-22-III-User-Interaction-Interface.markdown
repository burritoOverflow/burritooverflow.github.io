---
layout: post
title: 'III. User Interaction / Interface'
date: 2021-03-22 23:10:37 -0400
categories: project documentation
---

### Initialization - Sign-up / Login

The page served to a user when first navigation to the application is the login screen (if the user doesn’t have a valid JWT stored. The user is shown a simple form, a header displaying the
application’s name, and a link to sign up (for new users). Login requires an email and password. In the event a login fails, the user is informed via a toast message, displaying ‘Login Failed.
’ This is depicted in the second screenshot. Form submission is not performed; the login process is accomplished via AJAX. If the request succeeds, the user is briefly shown a toast indicating
success, and redirected to the root path (now with a JWT stored as an HTTP cookie), allowing them to access the application.

![login page](/assets/images/login.PNG)

A failed request.

![login fail](/assets/images/loginFail.PNG)

The login process is handled by `login.js`. The simplified code snippet for handling the login process is as follows:

```js
function doLogin(loginObj) {
  const loginApiUrl = '/api/users/login';
  fetch(loginApiUrl, {
    method: 'POST',
    body: JSON.stringify(loginObj),
    headers: {
      'Content-type': 'application/json; charset=UTF-8',
    },
  }).then((response) => {
    if (response.ok) {
      showSnackbarAndRedirect('Welcome back!', true);
    } else {
      showSnackbarAndRedirect('Login failed', false);
    }
  });
}
```

Server side, an overview of how this process is handled, first with simplified psuedocode:

```
if request.route == '/':
    if request.containsToken:
        # token provided currently pertains to a user
        if request.token == valid:
            serve(index.html)
    serve(login.html)
```

If the request does not contain a token, or the token is not valid, the user is redirected to the login page shown above. A truncated code snippet of this process within the route handler:

```js
if (req.cookies.token) {
  const userObj = await verifyUserJWT(req.cookies.token);
  // valid user
  if (userObj.name) {
    res.cookie('username', userObj.name, {
      httpOnly: true,
      secure: true,
      sameSite: true,
    });
    res.cookie('displayname', userObj.name);
    // send to the application page
    res.sendFile(path.join(__dirname, '..', 'html', 'index.html'));
  }
  // not a valid user
} else {
  res.redirect('/login');
}
```

With a valid token, users are served the main page, where they can choose a room to
join, create a new room, or logout. The username field is populated with their
username (cannot be changed); the input below the username allows for the user to
choose between current rooms. Alternatively, the user can click on a list element
within the `room status` element and populate that field with the selection.

If the user opts to logout, the current token is removed from the database, and the
session expires (the cookies are removed as well, along with preferences stored in
localstorage).

![room join](/assets/images/roomJoin.PNG)

If the user opts to create a new room, a seperate input field is displayed, and the room list is hidden. Rooms must be globally unique (case insensitive; all rooms are lowercase). Upon entering a room name, a request is made, if the room name is unique, it is created with the current user as the room's admin. Another request is made to the database to fetch the latest rooms; both the input and the room list are updated to reflect the change.

![room join 2](/assets/images/roomJoin2.PNG)

The code to handle the client-side request:

```js
async function addNewRoom(roomName) {
  const roomObj = { name: roomName };
  const roomRes = await fetch('/api/room', {
    method: 'POST',
    credentials: 'same-origin',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(roomObj),
  });
  return roomRes.json();
}
```

A simplified snippet of the server-side code to handle the room creation request in the route handler in `/routes/room.js`:

```js
if (req.cookies.token) {
  // verify the user making the request is a valid user
  const userObj = await verifyUserJWT(req.cookies.token);
  if (!userObj.name) {
    res.status(401).send({ error: 'Unauthorized' });
  }

  const { _id } = await User.findOne({ name: userObj.name });
  const roomAdmin = _id;
  const newRoomObj = { name: req.body.name, admin: roomAdmin };
  const existingRoom = await findRoomByName(req.body.name);

  // room exists, not created
  if (existingRoom) {
    res.status(401).send({
      error: `Cannot create. Room ${req.body.name} Exists`,
    });
  }

  // create the new room
  const room = new Room(newRoomObj);
  await room.save();

  res.status(201).send({
    result: `${req.body.name} created`,
  });
} else {
  // no token provided
  res.status(401).send({ error: 'Unauthorized' });
}
```