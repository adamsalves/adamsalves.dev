+++
title = "Planning Poker em tempo real com Socket.IO"
date = 2026-02-08
draft = false
description = "Salas, votação ao vivo e estatísticas de consenso — a arquitetura de eventos que mantém todo mundo em sincronia."
tags = ["vue", "socket.io", "real-time"]
+++

Tempo real parece complicado até você desenhar os eventos certos. No Vue Planning Poker, tudo gira em torno de salas e de um punhado de mensagens bem definidas.

## Os eventos

A regra que me salvou: o servidor é a fonte da verdade. O cliente só **emite intenções** e **reage ao estado** que volta.

```js
socket.on("vote:cast", ({ user, card }) => {
  room.votes[user] = card
  io.to(room.id).emit("room:state", serialize(room))
})
```

## Consenso

Quando todo mundo vota, calculo a moda e o desvio para mostrar se o time está alinhado ou se vale uma discussão.

> Real-time bom é aquele que você nem percebe: a tela só está sempre certa.

O resultado é uma mesa que parece mágica, mas é só um bom contrato de eventos.
