+++
title = "Gerando áudio chiptune com a Web Audio API"
date = 2026-02-27
draft = false
description = "Sons retrô 8-bit sem um único arquivo .wav: osciladores, envelopes e um pequeno SoundGenerator para o Phantom."
tags = ["typescript", "web-audio", "games"]
+++

No Phantom, eu não queria carregar dezenas de arquivos de áudio. A solução foi gerar tudo em tempo real com a Web Audio API — moedas, pulos e a trilha, tudo sintetizado.

## A ideia

Um oscilador mais um envelope de ganho já entrega aquele "blip" clássico de fliperama. O truque é controlar a frequência e o decaimento.

```ts
function blip(ctx: AudioContext, freq: number) {
  const osc = ctx.createOscillator()
  const gain = ctx.createGain()
  osc.type = "square"
  osc.frequency.value = freq
  gain.gain.setValueAtTime(0.2, ctx.currentTime)
  gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.15)
  osc.connect(gain).connect(ctx.destination)
  osc.start()
  osc.stop(ctx.currentTime + 0.15)
}
```

## O que aprendi

- `square` e `triangle` soam mais "8-bit" que `sine`.
- Sempre desconecte os nós depois do `stop` para não vazar memória.
- Um `AudioContext` só pode iniciar após um gesto do usuário — respeite isso.

E pronto: trilha inteira sem nenhum asset de áudio no bundle.
