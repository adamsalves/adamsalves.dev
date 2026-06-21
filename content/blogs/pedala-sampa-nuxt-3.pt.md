+++
title = "Migrando o Pedala Sampa para o Nuxt 3"
date = 2026-03-12
draft = false
description = "Como reescrevi um app Vue para Nuxt 3 sem perder SEO — rotas, composables e um redesign de direção wayfinding pelo caminho."
tags = ["nuxt", "vue", "migração"]
toc = true
+++

Neste post eu compartilho como abordei a migração do Pedala Sampa para o Nuxt 3: as decisões, os tropeços e o que eu faria diferente numa próxima. Sem enrolação — bora ao que importa.

## Contexto

O ponto de partida era simples na superfície, mas cheio de detalhes embaixo do capô. Eu queria uma base previsível, fácil de testar e que não me obrigasse a reaprender tudo a cada nova feature.

A ideia central foi isolar a lógica em `composables/` e deixar os componentes o mais burros possível — só apresentação.

## Mãos à obra

Na prática, o coração ficou num único composable. Repare como o estado e as ações saem juntos, prontos pra qualquer componente consumir:

```ts
// composables/useGroups.ts
export function useGroups() {
  const groups = ref<Group[]>([])

  async function load() {
    groups.value = await request(endpoint, GroupsQuery)
  }

  return { groups, load }
}
```

> Dica: comece pelo menor passo que entrega valor. O resto vem depois — e quase sempre vem mais simples do que você imaginou.

Os aprendizados que levei desse trecho:

- Estado perto da lógica, longe da UI.
- Tipar cedo evita 80% dos bugs chatos.
- Um teste pequeno hoje vale por uma tarde de debug amanhã.

## Conclusão

No fim, o que destrava o projeto raramente é a tecnologia mais nova — é a decisão de manter as coisas simples e testáveis. Se este post te poupou alguns minutos, já valeu. 🚀
