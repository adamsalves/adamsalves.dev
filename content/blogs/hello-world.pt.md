+++
title = "Hello, world: portfólio no ar e um tema Hugo de brinde"
date = 2026-08-11
draft = false
description = "Coloquei o adamsalves.dev no ar, arrumei a casa no GitHub e o tema que nasceu do projeto acabou virando o terminal-mono, publicado sob licença MIT."
tags = ["hugo", "portfolio", "open-source"]
toc = false
+++

Primeiro post do blog, então começo pelo começo: o [adamsalves.dev](https://adamsalves.dev/) está no ar. Comecei querendo só um lugar decente pra juntar meus projetos e acabei fazendo três coisas.

## O site

É um portfólio de página única — hero em formato de terminal, projetos em cards que lembram um repositório, experiência, contato e este blog. Feito em Hugo, publicado na Vercel, estático de ponta a ponta e em dois idiomas: português em `/`, inglês em `/en/`.

A ideia foi escapar do template genérico sem inventar demais: tema escuro, fonte monoespaçada, um verde-limão de destaque e nenhuma biblioteca de JS de terceiros. O conteúdo das seções mora todo em `params`, no `hugo.toml` — na prática, atualizar o site é mexer em config, não em template. Isso foi de propósito: template a gente edita uma vez e depois esquece como funciona.

## O perfil no GitHub

Aproveitei a virada pra arrumar a casa no [GitHub](https://github.com/adamsalves). O README ganhou um resumo do que eu faço e da stack do dia a dia (React, Vue 3, TypeScript, Next.js, Nuxt, React Native, Node.js — e Go, que estou começando a estudar), e no topo deixei fixado o que eu realmente gosto de mostrar:

- [**pedala-sampa**](https://github.com/adams-alves-dev/pedala-sampa), um mapa colaborativo pra ciclistas de São Paulo, em Vue 3 / Nuxt com Leaflet e GraphQL;
- [**phantom-collector**](https://github.com/adamsalves/phantom-collector), um arcade 8-bit em Phaser 3 onde o som chiptune é gerado na hora pela Web Audio API;
- [**planning-pvoker**](https://github.com/adamsalves/planning-pvoker), um planning poker em tempo real via WebSockets, com estatísticas de consenso;
- [**terminal-mono**](https://github.com/adamsalves/terminal-mono), o tema deste site — que é o assunto do próximo tópico.

## terminal-mono

Esse não estava no plano. No meio do caminho eu percebi que o tema não dependia de nada meu: o conteúdo todo vem de `params`, as cores e fontes saem de variáveis CSS. Pra outra pessoa usar, bastava editar um arquivo de config. Aí não fazia muito sentido deixar ele preso aqui dentro.

Separei num repositório próprio, escrevi a documentação, montei um `exampleSite` bilíngue pra servir de referência e soltei sob licença MIT: [terminal-mono](https://github.com/adamsalves/terminal-mono) — com [releases versionadas](https://github.com/adamsalves/terminal-mono/releases), caso você prefira fixar uma versão específica.

O que já vem resolvido: o hero com efeito de máquina de escrever, que degrada direitinho quando não tem JS; o blog inteiro, com paginação, índice, tags, barra de progresso de leitura e compartilhamento; i18n com português e inglês inclusos; SEO (Open Graph, JSON-LD, RSS, `canonical`); acessibilidade (*skip link*, foco visível, `prefers-reduced-motion`); e o pipeline de assets do Hugo com minify, fingerprint e SRI em produção.

Instalar é rápido, via Hugo Modules:

```bash
hugo mod init github.com/seu-usuario/seu-site   # se você ainda não usa modules
hugo mod get github.com/adamsalves/terminal-mono
```

```toml
# hugo.toml
[module]
  [[module.imports]]
    path = "github.com/adamsalves/terminal-mono"
```

Precisa do Hugo **extended** ≥ 0.158. Se você preferir submódulo git, também funciona.

## Daqui pra frente

O blog começa aqui. A ideia é escrever sobre o que aparece no dia a dia de front-end — design systems, testes, Vue e React — e sobre os bastidores dos projetos, que costuma ser a parte mais interessante. E se o terminal-mono te for útil, abre uma issue, manda um PR, ou só me conta o que quebrou.
