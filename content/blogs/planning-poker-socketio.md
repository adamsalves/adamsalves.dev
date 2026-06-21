+++
title = "Real-time Planning Poker with Socket.IO"
date = 2026-02-08
draft = false
description = "Rooms, live voting and consensus stats — the event architecture that keeps everyone in sync."
tags = ["vue", "socket.io", "real-time"]
+++

Real-time sounds complicated until you draw the right events. In Vue Planning Poker, everything revolves around rooms and a handful of well-defined messages.

## The events

The rule that saved me: the server is the source of truth. The client only **emits intentions** and **reacts to the state** that comes back.

```js
socket.on("vote:cast", ({ user, card }) => {
  room.votes[user] = card
  io.to(room.id).emit("room:state", serialize(room))
})
```

## Consensus

When everyone has voted, I compute the mode and the deviation to show whether the team is aligned or it's worth a discussion.

> Good real-time is the kind you don't even notice: the screen is just always right.

The result is a table that feels like magic, but it's really just a good event contract.
