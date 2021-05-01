---
layout: post
title: 'IV. Channels'
date: 2021-04-11 12:10:37 -0400
categories: project documentation
---

### Channels

Separate from the main chat is a `Channel` feature. The `Channel` feature allow for a microblogging-style social media feed.
This simplified broadcast tool allows the `Channel` owner the opportunity to use the feature as a one-to-many tool for sharing `Post`s with
all users of the service.
Non-admin users can 'like' or 'dislike' `Post`s shared in the channel. It provides a simple user interface displaying the owner's `Post`s
and the number of 'like' and 'dislikes'.

![channel intro](/assets/images/channelIntro.PNG)

Admin users, unlike passive observers, are provided a textarea input for adding `Post` content. When an admin user adds a `Post`, the `Post`'s contents are POSTed to
the application server, and the `Post` contents are stored in the database. All `Post`s are stored permanently in the same database as messages.

Displayed above the input for adding new content is a character counter that changes state to reflect the current number of non-white-space
characters currently in the input textarea. Here, input sizes have been limited to 200 characters or less—the counter changes to red when the limit is met or exceeded.
Post that meet or exceed this limit are not sent.

The `Post` schema is as follows:

```js
const PostSchema = new mongoose.Schema(
  {
    contents: {
      type: String,
      required: true,
    },
    date: {
      type: Date,
      default: Date.now,
    },
    sender: {
      // the channel's owner
      type: mongoose.Schema.Types.ObjectID,
      required: true,
      ref: 'User',
    },
    channel: {
      // the channel the `Post`s are shared in
      type: mongoose.Schema.Types.ObjectID,
      required: true,
      ref: 'Channel',
    },
    likes: {
      type: Number,
      default: 0,
    },
    dislikes: {
      type: Number,
      default: 0,
    },
  },
  {
    timestamps: true,
  }
);
```

The `Channel` schema is also quite simple:

```js
const ChannelSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      lowercase: true,
      trim: true,
      minLength: 5,
    },
    admin: {
      type: mongoose.Schema.Types.ObjectID,
      ref: 'User',
    },
    Posts: [
      {
        type: mongoose.Schema.Types.ObjectID,
        ref: 'Posts',
      },
    ],
  },
  {
    timestamps: true,
  }
);
```

When an admin adds content, they can add the post via pressing `Enter` on the textarea element.
The client-side code for sharing posts:

```js
textArea.onkeypress = async function (event) {
  const { key } = event;

  if (key === 'Enter') {
    const postcontents = textArea.value.trim();

    if (postcontents !== '') {
      // post the data to the channel route
      const response = await fetch(apiRoute, {
        method: 'POST',
        body: JSON.stringify({ postcontents }),
        headers: {
          'Content-Type': 'application/json',
        },
        credentials: 'same-origin',
      });

      if (response.ok) {
        // clear the input area
        textArea.value = '';

        // add the post to the channel posts
        chanObj.posts.push({
          contents: postcontents,
          date: new Date().toLocaleString(),
          likes: 0,
          dislikes: 0,
        });
      }
    } else {
      // empty post contents
      return;
    }
  } // key not enter
};
```

Server-side, for handling the POST request containing the contents. The request attempt is validated
to ensure that the user making the request is the `Channel`'s admin. If so, the `Post` is added:

```js
router.post('/channel/:channel/addpost', async (req, res) => {
  // <-snip->

  // parameter is the room name
  const channelName = req.params.channel;
  // ensure that the channel exists first
  const channel = await Channel.findOne({ name: channelName });

  const user = await User.findOne({ 'tokens.token': req.cookies.token });

  // now see if this user is the channel's admin
  if (!user._id.equals(channel.admin)) {
    // unauthorized request
    return res
      .status(401)
      .send({ error: "You don't have permission to add to this channel" });
  }

  // otherwise, add the post
  const postContents = req.body.postcontents;
  const post = new Post({
    contents: postContents,
    sender: user._id,
    channel: channel._id,
  });

  const savedPost = await post.save();
  channel.posts.push(savedPost._id);

  await channel.save();
  return res.status(201).send({ result: 'Post Added' });
});
```

Quick video showing the process of an Admin user adding a `Post` to the `Channel`.

<iframe width="740" height="425" src="https://www.youtube-nocookie.com/embed/rMsxKiCnjrw" title="YouTube video player" 
frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Admins can also delete posts they've created. When an admin views a `Channel` they administer, there is an additional element
on each post that allows them to delete that individual post. The accompanying code, in `channel.js`:

```js
const deletePostSpan = document.createElement('span');
deletePostSpan.classList.add('delete-post');
deletePostSpan.innerText = 'X';
postLi.appendChild(deletePostSpan);
postLi.style.paddingTop = 0;

const { channelName } = this;
const _this = this;

deletePostSpan.onclick = async function () {
  const res = await deletePost(post._id, channelName);
  // if success, we need to delete the post (the parent element containing all post contents)
  if (res) {
    // remove the line element between this post and the previous
    if (postLi.previousSibling) {
      postLi.previousSibling.remove();
    } else {
      // this is the first post element
      // so we also need its next sibling
      if (postLi.nextSibling) {
        postLi.nextSibling.remove();
      }
    }

    // finally, remove the post and update the state
    postLi.remove();
    _this.deletePost(post._id);

    // we'll do a quick styling on success
    const channelPosts = document.getElementById('channel-main');
    channelPosts.classList.add('delete-border');

    // revert the added temp style
    setTimeout(() => {
      channelPosts.classList.remove('delete-border');
    }, 1200);
  }
};
```

On click, a DELETE request is sent--if successful, the element is removed from the DOM and the client-side state.

Server-side, in `channel.js` (truncated):

```js
router.delete('/channel/:channel/:postid', async (req, res) => {
  // determine if the user making the request is the admin of the channel
  const channel = await Channel.findOne({ name: req.params.channel });
  if (!channel) {
    return res
      .status(400)
      .send({ error: `Channel ${req.params.channel} does not exist` });
  }

  // provided a valid channel, check that the user is the admin of the channel
  const { admin } = channel;
  const user = await User.findOne({ 'tokens.token': req.cookies.token });

  // now see if this user is the channel's admin
  if (!user._id.equals(admin)) {
    return res.status(401).send({
      error: "You don't have permission to delete posts from this channel",
    });
  }

  // we have a valid user, now we need to delete the post
  const post = await Post.findOneAndDelete({ _id: req.params.postid });
  if (!post) {
    return res.status(400).send({ error: 'Invalid Post id provided' });
  }

  // since the post has been deleted update the channel's latest update time
  channelsLastUpdate.set(req.params.channel, +new Date());
  return res.status(200).send({ result: 'Post deleted' });
});
```

Note that the timestamp for the channel is updated for the channel, so clients will request the updated data on the next polling event.
An overview of polling and timestamps are covered in the section that follows.

---

Clients poll the server for updates every 30 seconds. Every 30 seconds, the `getAllPostsInChannel` function is invoked to fetch the latest timestamp.

```js
async function setPollingInterval(chanObj) {
  const { channelName } = chanObj;
  setInterval(async () => {
    // check the timestamp
    const latestUpdate = await getLatestUpdateTime(channelName);

    if (latestUpdate > chanObj.latestUpdateTime) {
      // otherwise, fetch the data
      const { data } = await getAllPostsInChannel(channelName);
      // update the ui with the latest posts
      chanObj.posts = data;
      chanObj.latestUpdateTime = latestUpdate;
      resetChannelPosts();
      chanObj.displayPosts(document.getElementById('channel-posts'));
      scrollToBottomPosts();
    } else {
      return;
    }
    // poll every 30 seconds
  }, 30 * 1000);
}
```

The application server stores the timestamps for the latest post from each channel. The timestamp for the channel the user is viewing is
served to clients on each request during the interval. When the timestamp in the request is more recent than the timestamp the client has stored,
the client then requests the `Channel` posts and updates the UI with the latest posts.

```js
router.get('/channel/:channel/updatetime', async (req, res) => {
  // <-snip->

  // return the latest update for the channel
  return res.status(200).send({
    updateTime: channelsLastUpdate.get(channelName),
  });
});
```

The `channelsLastUpdate` is a `Map` created on server start via the `setLatestPostTimes` function, and subsequently updated when posts are added to channels.
`setLatestPostTimes` runs multiple queries for the latest post in each channel, and sets the timestamp as the value for the channel name key. The value is
returned to the requesting client from the channel route param in the `updatetime` route handler above.

```js
async function setLatestPostTimes() {
  channelsLastUpdate = new Map();
  const channelMap = new Map();
  const channels = await Channel.find({});
  const channelIds = new Array();

  // get all channel ids
  for (let idx = 0; idx < channels.length; idx++) {
    const { _id, name } = channels[idx];
    // channel map contains channel's id as key and the channel's
    // name as the value
    channelMap.set(_id.toString(), name);
    channelIds.push(_id);
  }

  // create a query for each
  const queries = new Array();
  channelIds.forEach((ch) => {
    // we only need the latest post from each channel
    const getPostsQuery = Post.find({ channel: ch })
      .sort({
        date: -1,
      })
      .limit(1);
    queries.push(getPostsQuery.exec());
  });

  Promise.all(queries).then((response) => {
    response.forEach((doc) => {
      if (doc.length) {
        const { date, channel } = doc[0];
        // get the channel name from the channel id
        const chanName = channelMap.get(channel.toString());
        // for handling the case where there is a channel w/o posts
        channelMap.delete(channel.toString());
        channelsLastUpdate.set(chanName, +new Date(date));
      }
    });
    // check for any remaining keys in the channel map
    // these are for leftover channels not covered by the query
    // those with 0 posts
    const channelIdKeys = [...channelMap.keys()];
    if (channelIdKeys.length) {
      for (const [_, cName] of channelMap) {
        // set the channel with no posts to no update time
        channelsLastUpdate.set(cName, null);
      }
    }
  });
}
```

The update for new posts in the `addpost` route:

```js
channelsLastUpdate.set(channelName, +new Date());
```

Non-admin clients can 'like' and 'dislike' `Posts`. This is shown in the following video:

<iframe width="740" height="425" src="https://www.youtube-nocookie.com/embed/D5yCFCAZkZo" title="YouTube video player" 
frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

These 'reactions' are sent via POST and the count is stored with the `Post` in the database. Only non-admin users have event handlers on the
reaction UI elements:

```js
likes.addEventListener('click', async () => {
  const res = await addReaction(post._id, 'like', chanName);
  if (res) {
    console.log(res);
    const likeCounter = likes.innerHTML.slice(3);
    let counter = parseInt(likeCounter);
    ++counter;
    likes.innerHTML = String.fromCodePoint(0x1f44d) + ' ' + counter;

    // toggle the liked class style on like
    div.classList.add('liked');
    postLi.classList.add('liked-border');
    setTimeout(() => {
      // remove after 900 ms
      div.classList.remove('liked');
      postLi.classList.remove('liked-border');
    }, 900);
  }
});

dislikes.addEventListener('click', async () => {
  const res = await addReaction(post._id, 'dislike', chanName);
  if (res) {
    const dislikeCounter = dislikes.innerHTML.slice(3);
    let counter = parseInt(dislikeCounter);
    ++counter;
    dislikes.innerHTML = String.fromCodePoint(0x1f44d) + ' ' + counter;

    div.classList.add('disliked');
    postLi.classList.add('disliked-border');
    setTimeout(() => {
      div.classList.remove('disliked');
      postLi.classList.remove('disliked-border');
    }, 940);
  }
});
```

The `addReaction` function, used within the event handlers:

```js
async function addReaction(postid, reaction, chanName) {
  const apiRoute = '/api/channel/' + chanName + '/reaction';
  const response = await fetch(apiRoute, {
    method: 'POST',
    body: JSON.stringify({ reaction, postid }),
    headers: {
      'Content-Type': 'application/json',
    },
    credentials: 'same-origin',
  });
  const jsonData = await response.json();

  if (response.ok) {
    return jsonData;
  }
}
```

Server-side, the `reaction` route handler checks that the `Post` and the channel are both valid:

```js
router.post('/channel/:channel/reaction', async (req, res) => {
  const { reaction, postid } = req.body;
  // we only need this to see if it's a valid channel
  const channelName = req.params.channel;
  const postToUpdate = await Post.findById({ _id: postid });

  if (!channelsLastUpdate.has(channelName) || !postToUpdate) {
    return res.status(404).send({ error: 'Invalid post or channel' });
  }

  // update the metric
  switch (reaction) {
    case 'like':
      postToUpdate.likes += 1;
      await postToUpdate.save();
      return res.status(201).send({ updated: postToUpdate.likes });
    case 'dislike':
      postToUpdate.dislikes += 1;
      await postToUpdate.save();
      return res.status(201).send({ updated: postToUpdate.dislikes });
    default:
      return res.status(400).send({ error: 'invalid reaction provided' });
  }
});
```
