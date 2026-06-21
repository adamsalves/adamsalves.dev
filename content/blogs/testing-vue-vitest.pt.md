+++
title = "Testes em Vue na prática: Vitest + Testing Library"
date = 2026-01-21
draft = false
description = "Um fluxo de TDD que eu realmente mantenho: o que testar, o que ignorar e como deixar a suíte rápida."
tags = ["testes", "vitest", "tdd"]
+++

Testar Vue ficou prazeroso com o Vitest. Mas ferramenta boa não substitui critério: o que vale a pena testar?

## O que testar

Eu testo **comportamento**, não implementação. Se o usuário clica e algo aparece, é isso que o teste verifica.

```ts
import { render, screen } from "@testing-library/vue"
import Counter from "./Counter.vue"

test("incrementa ao clicar", async () => {
  const { getByRole } = render(Counter)
  await getByRole("button").click()
  expect(screen.getByText("1")).toBeTruthy()
})
```

## O que ignorar

- Detalhes internos (nomes de métodos, refs privados).
- Bibliotecas de terceiros já testadas.
- Snapshots gigantes que ninguém lê.

## Velocidade

Mantenha a suíte rápida e ela vira hábito. Lenta, vira fila de espera — e some.
