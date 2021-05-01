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

For the `user` model schema:

Usernames are converted to lowercase and must be unique; this is performed via the model schema, see `/models/user.js`. Code snippet for the `user` model.

```js
const UserSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      unique: true,
      lowercase: true,
      required: true,
      trim: true,
      minLength: 3,
    }
```

Password strength is enforced, and the following must be met for the password to be valid,
also in `/models/user.js`. If the password is weak, this is reported back to the user when attempting to sign up.

```js
password: {
    type: String,
    required: true,
    trim: true,
    validate(value) {
    if (
        !validator.isStrongPassword(value, {
        minLength: 8,
        minLowercase: 1,
        minUppercase: 1,
        minNumbers: 1,
        minSymbols: 1,
        returnScore: false,
        })
    ) {
        throw new Error('Weak password');
    }
    },
}
```

![login page](/assets/images/login.PNG)

A failed request.

![login fail](/assets/images/loginFail.png)

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

If the user opts to create a new room, a separate input field is displayed, and the room list is hidden. Rooms must be globally unique (case insensitive; all rooms are lowercase). Upon entering a room name, a request is made, if the room name is unique, it is created with the current user as the room's admin. Another request is made to the database to fetch the latest rooms; both the input and the room list are updated to reflect the change.

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

A simplified snippet of the server-side code to handle the room creation request in
the route handler in `/routes/room.js`:

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

---

### Demo - User Creation, Room Creation and Joining a Room

The following video shows the process of user sign-up, the creation and subsequent joining of a new room along with a UI easter egg on the `join` page.

<iframe width="740" height="425" src="https://www.youtube.com/embed/DvG5XR9BK18" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

For a look into the database, here's `alice2221`'s entry:

```json
{
  "_id": { "$oid": "605a9dd8e876d054ac9b416d" },
  "name": "alice2221",
  "email": "alice221@yahoo.org",
  "password": "$2a$10$DHWXKAY1e5wI5Lr1J0ZIjeUFUUSusMqH/AXLokMS.eVLhuPGvO0Lu",
  "tokens": [
    {
      "_id": { "$oid": "605a9dd9e876d054ac9b416e" },
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MDVhOWRkOGU4NzZkMDU0YWM5YjQxNmQiLCJpYXQiOjE2MTY1NTEzODV9.009WTYExU-tumRNArzNeYOjar9CnioxI40Zt_iU2XiY"
    }
  ],
  "socketIOIDs": [],
  "createdAt": { "$date": "2021-03-24T02:03:04.892Z" },
  "updatedAt": { "$date": "2021-03-24T02:03:35.702Z" },
  "__v": 3
}
```

Note a few things here: `alice2221`'s password is stored hashed + salt, and she has one active JWT. The `socketIOIDs` array is empty, as this screenshot was taken just after exiting the room. We have the `name` and `email` as entered.

Here is `alice2221`'s newly created room: `aliceroom`. This was also copied after leaving the room. so we see no `users` present. The admin of the room is `alice2221`'s `_id`, as she is the user that created it.

```json
{
  "_id": { "$oid": "605a9de4e876d054ac9b416f" },
  "name": "aliceroom",
  "admin": { "$oid": "605a9dd8e876d054ac9b416d" },
  "users": [],
  "createdAt": { "$date": "2021-03-24T02:03:16.490Z" },
  "updatedAt": { "$date": "2021-03-24T02:03:35.592Z" },
  "__v": 2
}
```

Last, here is the message `alice2221` sent during her brief foray into her room:

```json
{
  "_id": { "$oid": "605a9defe876d054ac9b4172" },
  "contents": "Wow my own room.",
  "sender": { "$oid": "605a9dd8e876d054ac9b416d" },
  "date": { "$date": "2021-03-24T02:03:27.744Z" },
  "room": { "$oid": "605a9de4e876d054ac9b416f" },
  "createdAt": { "$date": "2021-03-24T02:03:27.918Z" },
  "updatedAt": { "$date": "2021-03-24T02:03:27.918Z" },
  "__v": 0
}
```

We see that the `room` is the same `_id` as `aliceroom` and the `sender` is the same `_id` as `alice2221`.

---

### Rooms and Messaging

Users send messages to the room via the textarea element. Messages can be sent by either clicking the `send message` button or pressing the enter key.
After sending, the message is received by the server, stored in the database, and sent back to the room (with a few exceptions).
The UI updates for messages are 'honest', in that instead of just adding the sent message to the UI in an optimistic fashion,
a user doesn't see the messaged added to the `rooms`'s messages until the other participants do; once a message is visible, it's already been
received and stored by the server.

An overview of the messaging process, from the client, in `chat.js`:

```js
function sendMessage(message) {
  const msgObj = { message, msgSendDate: +new Date() };
  socket.emit('clientChat', msgObj, (serverMsg) => {
    showUserToast(serverMsg);
  });
}
```

It's quite simple. The client emits a `clientChat` event when this function is invoked. The callback provided
runs if there's an error (communicated back via the server) showing the user a toast with the error message. The `sendMessage` function is called in two places
for the `send message` button click, or when the enter key is pressed and the textarea is focused.

```js
msgBtn.addEventListener('click', () => {
  // make sure a message is present
  const msgStr = msgInput.value.trim();
  if (msgStr === '') {
    return;
  }

  if (determineIfDelayedMessage(msgStr)) {
    sendDelayedMessage(msgStr);
  } else {
    sendMessage(msgStr);
  }

  // clear the input after send
  msgInput.value = '';
  // --snip--
```

and

```js
msgInput.addEventListener('keypress', (e) => {
  const { key } = e;
  if (key === 'Enter') {
    const msgStr = msgInput.value.trim();
    if (msgStr !== '') {
      if (determineIfDelayedMessage(msgStr)) {
        sendDelayedMessage(msgStr);
      } else {
        sendMessage(msgStr);
      }

      // clear the message that was sent
      msgInput.value = '';
  // --snip--

```

A greatly simplified snippet of the server-side handling of this event in `index.js` (leaving out various checks for special messages):

```js
socket.on('clientChat', async (msgObj, callback) => {
  const { message } = msgObj;

  // --snip--

  addMessage(socket, msgObj, socketUser.room);
  // emit the message
  io.to(socketUser.room).emit('chatMessage', msgObj);
});
```

The function that stores messages in MongoDB:

```js
async function addMessage(socket, msgObj, roomName) {
  const { message } = msgObj;
  const _user = await User.findOne({
    'socketIOIDs.sid': socket.id,
  });

  const room = await Room.findOne({ name: roomName });

  const _message = new Message({
    contents: message,
    sender: _user._id,
    date: new Date(msgObj.msgSendDate),
    room: room._id,
  });

  const savedMessage = await _message.save();
}
```

The user's `_id` is located via the socketIO id provided in the first argument. The message contents are destructured
from the second argument, and the `room` `_id` is located via a query for the `roomName` provided via the third argument.
Additionally, the current date is added to the message--this date reflects the time that the message was sent from the client.
This message is then saved. Note this format matches the JSON format we showed earlier of `alice2221` demo message.

After logging and storing the message, the server emits a `chatMessage` event to the `room`. When
clients receive this event, the contents of the message are added to the UI:

`chat.js`

```js
socket.on('chatMessage', (message) => chatMessageReceived(message));
```

---

### File Sharing

Within the main chat, users can share files with the group. A file input element exists near the chat, allowing users to select a file to share.
File size limits are limited to 5MB. The file selected by the user is POSTed to the application server,
where it's uploaded to S3. Upon completion, the URL where the file is
accessible is shared to the chat, allowing any of the
chat's users to download the file.

```js
const fileInput = document.getElementById('file-upload-input');
const onSelectFile = () => uploadFile(fileInput.files[0]);
fileInput.addEventListener('change', onSelectFile, false);
```

The upload file function:

```js
function uploadFile(file) {
  const data = new FormData();
  data.append('upload', file);

  // post data
  fetch('/api/messages/file', {
    method: 'POST',
    body: data,
  })
    .then((response) => response.json())
    .then(
      (resJSON) => {
        if (resJSON.error) {
          showUserToast(resJSON.error);
        } else {
          const userMessage = `shared ${resJSON.originalName} : ${resJSON.url}`;
          sendMessage(userMessage);
        }
        fileInput.value = '';
    )
    .catch(
      (error) => console.log(error)
}
```

The route for handing file uploads (truncated):

```js
router.post(
  '/messages/file',
  upload.single('upload'),
  async (req, res) => {
    const uploadDateStr = String(+new Date());

    const fileBuffer = req.file.buffer;
    const fileExtension = req.file.originalname.slice(
      req.file.originalname.indexOf('.')
    );
    let filename = req.file.originalname.slice(
      0,
      req.file.originalname.indexOf('.')
    );

    // replace spaces with underscores
    filename = filename.split(' ').join('_');
    const uploadFilename = `${filename}_${uploadDateStr}.${fileExtension}`;

    const s3Response = await s3
      .upload({
        Bucket: process.env.S3_BUCKET_NAME,
        Key: uploadFilename,
        Body: fileBuffer,
      })
      .promise();

    // build the public URL
    const publicURLPrefix = process.env.PUBLIC_S3_URL;
    const filePublicURL = `${publicURLPrefix}${uploadFilename}`;

    // on success, return the public url in the response
    res.status(201).send({
      originalName: req.file.originalname,
      url: filePublicURL,
    });
  },
  (error, req, res, next) => {
    // return error if failed
    res.status(400).send({ error: error.message });
  }
);
```

The end result is the chat element containing the URL for the uploaded file.

![chat file upload](/assets/images/fileShareThread.PNG)

---

### Private Messaging

Users can send private messages to other users in the same room. Private messages differ from the messages previously demonstrated in several
ways:

1. Private messages are only visible between the sender and recipient.

2. Private messages are not stored persistently in the database. The server simply sends the private messages to the recipient.

3. Private messages are totally ephemeral; they reside only on the script and on the involved clients in localstorage.

The following video demonstrates the private messaging functionality and compares it to the room-wide messaging functionality.

<iframe width="740" height="425" src="https://www.youtube.com/embed/Wq000OT_cLw" title="YouTube video player" frameborder="0" 
allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The private message element can be toggled by pressing the `'p'` key (outside of an input element), or clicking the toggle button.
On the client, the sending functionality is handled by the `sendPM` function.

```js
function sendPM() {
  const content = document.getElementById('pm-textarea').value;
  if (content.trim() === '') {
    return;
  }

  if (pmReciever === undefined) {
    return;
  }

  socket.emit('private message', {
    content,
    to: pmReciever.id,
    toName: pmReciever.username,
  });

  const toNameLower = pmReciever.username.toLowerCase();
  const msgDate = new Date().toLocaleString();

  const pm = new PrivateMessage(toNameLower, msgDate, content);
  pmMap.addPM(toNameLower, pm);

  const pmList = document.getElementById('pm-list');
  const pmLi = document.createElement('li');
  pmLi.classList.add('sent-pm-li');
  pmLi.innerText = `${msgDate} ${content}`;
  pmList.appendChild(pmLi);

  document.getElementById('pm-textarea').value = '';
}
```

On the server, the `'private message'` event is handled succintly:

```js
socket.on('private message', ({ content, to }) => {
  // note that this is only sent to the 'to'
  socket.to(to).emit('private message', {
    content,
    from: socket.id,
    fromName: socket.username,
  });
});
```

When the client receives the `'private message'` event, the message is stored locally, the UI updated, and the user
notified.

```js
socket.on('private message', (pm) => {
  const fromNameLower = pm.fromName.toLowerCase();
  const pmsg = {
    content: pm.content,
    date: new Date().toLocaleString(),
  };

  const privMsg = new PrivateMessage('You', pmsg.date, pmsg.content);
  pmMap.addPM(fromNameLower, privMsg);
  const { room } = parseQSParams();
  const lsPMStr = `pms${room}`;
  pmMap.setLocalStorage(lsPMStr);

  const message = `PM from ${pm.fromName}`;
  updatePmLiElement(fromNameLower);

  if (document.hidden) {
    displayNotification(message);
  } else {
    showUserToast(message);
  }

  // if the currently selected PM user is the user that sent the
  // PM, update the PM list when recieved
  if (pmReciever.username === fromNameLower) {
    const pmList = document.getElementById('pm-list');
    const pmLi = document.createElement('li');
    pmLi.innerText = `${pmsg.date} ${pmsg.content}`;
    pmList.appendChild(pmLi);
  }
});
```

Two classes exist for the purpose of handling Private Messages, `PrivateMessage` and `PrivateMessageMap`. The former
is used for each individual message; the latter is used for storing all of the user's private messages from this room.

Each Private Message object stores the recipient `to` the `date` and the message `contents`.

```js
class PrivateMessage {
  constructor(to, date, contents) {
    this._to = to;
    this._date = date;
    this._contents = contents;
  }

  // snip

  toJSON() {
    const { to, date, contents } = this;
    return { to, date, contents };
  }
}
```

`PrivateMessageMap` is a simple `Map` that stores the username of the other user involved in the Private Message thread
as key and an array of the `PrivateMessage` objects as value--the storage is demonstrated in the event hander snippet above.
On initial load, if localstorage contains pm contents for the current `room`, the contents are parsed and stored in the
current `PrivateMessageMap`. Each subsequent message sent or received is added to the map, and the contents are then serialized
and added in localstorage.

```js
  setLocalStorage(lsKeyname) {
    const jsonMap = Object.fromEntries(this._pmMap);
    const mapStr = JSON.stringify(jsonMap);
    localStorage.setItem(lsKeyname, mapStr);
  }

  parseFromlocalStorage(lsKeyname) {
    const lsPMs = localStorage.getItem(lsKeyname);
    if (!lsPMs) {
      return false;
    }

    const jsonPMs = JSON.parse(lsPMs);
    this._pmMap = new Map(Object.entries(jsonPMs));
    return true;
  }
```

The private message element provides a simple UI that lists the users and the message threads between them.
In each thread, sent messages are displayed in black, and received messages are displayed in grey.

![room join](/assets/images/pm1.PNG)

---

### Additional Features and Functionality

#### Keybindings

Several keybindings have been added to increase usability and allow users to easily use additional functionality; these keypresses
only function when the user is not focused on the textarea input:

`x` - Make the browser fullscreen.

`q` - 'focus' mode. This dramatically changes the UI, hiding elements that are not the message thread nor the textarea input:

![focus mode](/assets/images/focusMode.PNG)

`m` - focus on the textarea element--for quickly entering message input.

`s` - open the file share prompt, instead of clicking on it manually.

`f` - focus on the 'search message' element, allowing for a quick search of message contents in the current elements in the thread.

`t` - scroll to the top of the message thread.

`b` - scroll to the bottom of the message thread.

`k` - scroll up by ~the height of a single message.

`j` - scroll down by ~the height of a single message.

#### Exploding Messages

Users can send a special type of message that does not persist (is not stored), and is removed from the DOM in 45 seconds. These 'exploding' messages require a special syntax:

```
/explode | /expire <message>
```

Client-side, this is handled in the `chatMessageReceived` function; if the message object has the expiration flag set
the client sets a timeout, and styles the element in a distinct fashion:

```js
if (message.expireDuration) {
  createdLi.classList.add('expire-msg');
  createdLi.childNodes.forEach((child) => {
    if (!child.classList) {
      return;
    }

    // ignore the date element
    if (!child.classList.contains('date-span')) {
      child.style.backgroundColor = 'white';
    } else {
      const secondsExpire = Number(message.expireDuration) / 1000;
      child.innerText = `Remains for: ${secondsExpire}`;

      // update the remaining time on the message once per second
      setInterval(() => {
        let updateTime = Number(child.innerText.split(' ')[2]);
        --updateTime;
        child.innerText = `Remains for: ${updateTime}`;
      }, 1000);
    }
  });

  setTimeout(() => {
    createdLi.remove();
  }, Number(message.expireDuration));
}
```

The exploding message style:

![exploding message](/assets/images/explodeMessage.PNG)

#### Change UI Color

A dropdown is placed near the bottom of the window, allowing users to set their preferred accent color. On select, the user's preference is stored in
localstorage, and the value is retrieved in the future, maintaining the color choice:

![color change](/assets/images/colorDropDown.png)

The event handler for the color change is implemented in the `init` function; on click, the color
is stored as the user's preferred color, and the UI accent colors are changed to reflect this choice:

```js
for (let i = 1; i < dropdown.childNodes.length; i += 2) {
  // add a click handler to each
  dropdown.childNodes[i].addEventListener('click', (e) => {
    e.preventDefault();
    const textColorChoice =
      dropdown.childNodes[i].innerText.toLowerCase() + choiceStr;

    const colorHex = getComputedStyle(
      document.documentElement
    ).getPropertyValue(`--${textColorChoice}`);

    document.documentElement.style.setProperty('--primaryColor', colorHex);

    // we also need to set the snackbar color
    const snackBar = document.getElementById('snackbar');
    snackBar.style.backgroundColor = colorHex;

    // set the selection in the user's localstorage
    localStorage.setItem('colorChoice', colorHex);
  });
}
```

---

### Published Events

The following events are published to Redis, allowing subscribed users (clients viewing the index page) to view real-time updates of events
occurring in `Rooms` and `Channels`. Room join events, room leave events, and a generic message when a new message is added in a room are published.

in `index.js` a Redis client is create on server start. The `setUpdate` method is invoked on the `RedisUtils` instance during the
aforementioned events.

```js
const redisPublishClient = new RedisUtils(
  process.env.REDIS_HOSTNAME,
  process.env.REDIS_PORT,
  process.env.REDIS_PASSWORD
);
redisPublishClient.connectToRedis();
```
