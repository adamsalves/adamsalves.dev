+++
title = "Generating chiptune audio with the Web Audio API"
date = 2026-02-27
draft = false
description = "Retro 8-bit sound without a single .wav file: oscillators, envelopes and a tiny SoundGenerator for Phantom."
tags = ["typescript", "web-audio", "games"]
+++

For Phantom, I didn't want to load dozens of audio files. The solution was to generate everything in real time with the Web Audio API — coins, jumps and the soundtrack, all synthesized.

## The idea

An oscillator plus a gain envelope already delivers that classic arcade "blip". The trick is controlling the frequency and the decay.

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

## What I learned

- `square` and `triangle` sound more "8-bit" than `sine`.
- Always disconnect the nodes after `stop` so you don't leak memory.
- An `AudioContext` can only start after a user gesture — respect that.

And that's it: a whole soundtrack with no audio assets in the bundle.
