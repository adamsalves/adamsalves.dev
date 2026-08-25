+++
title = "A fonte era o caminho crítico: auditando o terminal-mono"
date = 2026-08-25
draft = false
description = "Abri uma auditoria contra o meu próprio tema. A fonte do Google custava 1,6 s por visita fria, o token de texto secundário reprovava no WCAG nas cinco paletas, e o site não tinha nada a dizer pra um answer engine."
tags = ["hugo", "performance", "aeo", "open-source"]
toc = true
+++

No [post anterior](/blogs/hello-world/) eu contei que o tema deste site tinha virado repositório próprio. Ele está na v0.6.0 agora, e quase tudo que entrou nas últimas semanas saiu de uma auditoria que abri contra mim mesmo: Lighthouse 12.8 no `exampleSite` publicado, e o checker do [aeo.js](https://aeojs.org/) na mesma URL.

O resultado inicial não era ruim. Performance entre 91 e 93, acessibilidade 95, SEO 100. Mas três auditorias saíam zeradas nas três páginas que eu medi — `render-blocking-resources`, `render-blocking-insight` e `network-dependency-tree-insight` — e o AEO deu 55 de 100, com Schema Presence em zero.

O que segurava a nota não era o JavaScript. TBT em 0 ms, DOM com 201 elementos. Era a fonte.

## A fonte era o caminho crítico, não o CSS do tema

O `head.html` carregava JetBrains Mono com um `<link rel=stylesheet>` pro `fonts.googleapis.com`. Isso monta uma cadeia serial que o browser só descobre depois de parsear o HTML:

```
localhost/                                  706 ms
 └─ fonts.googleapis.com/css2?family=…      694 ms   ← bloqueia o render
     └─ fonts.gstatic.com/….woff2         1.602 ms   ← só começa aqui
```

O `terminal.min.css` baixava em paralelo e terminava em 1.261 ms. Então a fonte era o caminho crítico e o CSS do próprio tema não era. Os dois `preconnect` que já estavam lá resolvem o handshake e não removem o salto: o browser ainda precisa baixar e parsear o CSS do Google pra descobrir qual `.woff2` pedir.

A estimativa do Lighthouse pra tirar o bloqueio ficava entre 1.330 e 1.590 ms, dependendo da página.

Fiz o self-host. Dois subsets, `latin` com 31,4 KB e `latin-ext` com 11,6 KB, os dois variáveis cobrindo `400 800` — um arquivo por subset serve todos os pesos que o tema usa. Só o `latin` entra no `preload`, porque preload que a página não pede é round trip jogado fora e um aviso no console.

Um detalhe que eu não esperava encontrar no caminho: a URL antiga pedia `wght@400;500;600;700;800`, e o Google respondia com **30 blocos `@font-face` e 12,4 KB de CSS** pra chegar no mesmo arquivo que `wght@400..800` devolve em seis blocos.

Medido depois: mobile de **92 pra 100**, FCP de **2,5 s pra 1,1 s**, Speed Index de **4,4 s pra 1,1 s**, e a maior cadeia crítica de **1.602 ms pra 44 ms**. As três auditorias zeradas passaram.

Os binários são os mesmos que o `fonts.gstatic.com` serve para a v24, commitados sem modificação, com um `SHA256SUMS` que o CI confere. Isso não é zelo decorativo: o header `immutable` que eu recomendo no README só é seguro se trocar os bytes significar escrever outro nome de arquivo. Agora trocar em cima quebra o build em vez de envenenar o cache de quem já visitou.

## O typewriter reconstruía o mesmo nó 442 vezes

O hero digita 442 caracteres. Cada tick escrevia `tailEl.innerHTML`, e o `wrap()` de um segmento inacabado sempre produz a mesma forma — um `span` com uma cor. Eram 442 passadas no parser de HTML pra reconstruir um nó idêntico ao que tinha acabado de ser destruído.

Agora a cauda é um `span` com um nó de texto, os dois construídos uma vez, e o loop só reatribui o `.data`. Em três rodadas de Lighthouse num build de produção, mobile: **parsing de HTML de 62 ms pra 12 ms**, consistente e bem fora da variação entre rodadas.

O loop também saiu do `setTimeout(…, 15)` pro `requestAnimationFrame`, escrevendo uma vez por frame em vez de uma vez por caractere — e só nos frames em que algum caractere venceu, porque reatribuir a mesma string ainda suja o nó, e a pausa de abertura são uns vinte e cinco frames de exatamente isso. Ele para de vez numa aba em segundo plano, onde o timer antigo seguia digitando pra ninguém.

E o `.term__body` ganhou `contain: layout style`. O `min-height` acima dele já garante que a caixa não muda de tamanho enquanto enche; isso é só avisar o browser. Sem o `contain`, cada escrita invalidava o layout do documento inteiro — 172 invalidações numa rodada.

## Duas coisas não sobreviveram à medição

Essa é a parte que eu quase deixei de fora, e é a que eu mais queria ter lido em outro lugar antes.

O relatório original atribuía **1,4 s de main thread** ao typewriter. Esse número veio de um A/B contra `--force-prefers-reduced-motion`, que simplesmente não roda a animação. Então ele é o custo de a animação **existir**, não o custo de como ela está escrita. A reescrita vale os 50 ms de parsing acima e o trabalho que ela para de fazer numa aba escondida. Não vale 1,4 s, e eu teria escrito que valia se não tivesse medido de novo.

O `forced-reflow-insight` também não sai do zero. Ele dispara em cerca de metade das rodadas do Lighthouse, antes e depois, porque o que está sendo forçado é o primeiro layout do documento — que a reserva de altura do hero precisa de verdade. Tentei reaproveitar uma sonda persistente em vez de criar uma por chamada: não moveu a auditoria em nada e deixou o `0123456789` da régua dentro do `textContent` do terminal pelo resto da vida da página, legível por qualquer coisa que extraia texto do DOM renderizado — o que neste tema agora inclui os answer engines. Reverti. Adiar a primeira medição pro `requestAnimationFrame` limparia a auditoria entregando a reserva, que é justamente o layout shift que ela existe pra evitar.

Deixei as duas registradas no changelog em vez de sumir com elas.

## `--dim` reprovava no WCAG nas cinco paletas

O Lighthouse apontou duas ocorrências, `.term__title` na home e `.toc__label` no post. Mas o problema era do token, não dos dois seletores: `--dim` é lido por 13 regras do `terminal.css` — rodapé, meta dos posts, paginação, datas da timeline, seletor de idioma, `figcaption`, números de linha do Chroma. Tudo texto de 12 a 13,5 px, onde o mínimo é 4,5:1 e não os 3:1 que texto grande ganha.

Media entre 3,44:1 e 4,07:1 dependendo da paleta, e o pior caso era contra `--surface-2` — o fundo **mais claro**, não o `--bg`. Nenhuma das cinco passava contra nenhum dos fundos.

Subi cada valor o mínimo pra limpar 4,5:1 contra esse pior caso, preservando o matiz: `#6b7263 → #798071` no lime, `#776a56 → #887b67` no amber, `#6d6390 → #8076a3` no cyberpunk, `#62737f → #6e7f8b` no ice, `#6a6a6a → #7d7d7d` no mono. Acessibilidade foi a 100 em todas as páginas.

Duas dessas correções não estavam no relatório e apareceram porque eu conferi a rampa inteira em vez do token acusado: o `--dim-2` do mono reprovava do mesmo jeito, em 4,45:1, e clarear o `--dim` do amber empurrou ele **acima** do `--dim-2` que ele deveria estar abaixo — uma paleta que passa no AA contradizendo os próprios nomes de token.

Entrou junto um `check_contrast.py` no CI. Ele afirma três coisas sobre as cinco paletas de uma vez: todo token de texto limpa 4,5:1 contra todo fundo declarado, a rampa `--dim < --dim-2 < --muted-2 < --muted < --prose < --soft < --text` está ordenada por luminância, e qualquer valor que o parser não entenda é falha nomeada em vez de token ignorado. Esse último ponto é o checker checando a si mesmo: um `--dim` escrito como `rgb(106,106,106)` nunca entrava no parser, e uma folha de estilo reprovando no AA voltava limpa.

## O tema não dizia o que era

A outra metade da auditoria era AEO — Answer Engine Optimization, a sigla nova pra "o que acontece quando quem lê a sua página é um modelo respondendo a pergunta de alguém".

O diagnóstico era curto e feio:

- o `robots.txt` era o stub padrão do Hugo, literalmente `User-agent: *` e nada mais, sem linha `Sitemap:`;
- não existia `llms.txt` nem `llms-full.txt`, 404 nos dois;
- o JSON-LD saía só na home e só como `Person`. Os posts não diziam que eram posts. Daí o zero em Schema Presence.

O tema builda com `hugo` e nada mais, e o aeo.js é pacote npm de 6,5 MB sem integração Hugo — adotá-lo como dependência de build significaria trazer Node pra dentro, e o widget "Human/AI" que ele injeta contradiz o "no third-party JS" que está no README, no `theme.toml` e na descrição do repositório. Então implementei as saídas nativamente em Hugo e uso o checker só como verificador.

O `robots.txt` agora nomeia os crawlers de IA em dois grupos, porque não são o mesmo pedido. Answer engine busca a página pra responder uma pergunta agora e devolve a citação pro leitor: `OAI-SearchBot`, `Claude-User`, `PerplexityBot`, `Applebot` e mais seis. Crawler de dataset coleta a página pro corpus, sem citação e sem referral: `GPTBot`, `ClaudeBot`, `Google-Extended`, `CCBot` e mais quatro. `[params.aeo] allowAI` e `allowTraining` ligam e desligam os dois de forma independente, os dois com default `true` — que é o que o `User-agent: *` pelado já queria dizer, então atualizar o tema não muda em silêncio o que o seu site publica.

Grupo de robots.txt não herda, e isso importa: um caminho excluído só no `*` continuaria aberto exatamente pros crawlers que você acabou de nomear. Então `disallow` é repetido em todos os grupos permitidos.

O JSON-LD passou a cobrir o que o tema realmente renderiza — `BlogPosting` nos posts, `WebPage` nas outras páginas simples, `CollectionPage` no índice do blog e nas tags, `BreadcrumbList` em tudo menos a home. Os nós são ligados em vez de repetidos: o publisher carrega um `@id` e o `author` do post aponta pra ele. As migalhas vêm de `.Ancestors`, a árvore de conteúdo real, e não da string da URL — assim uma migalha não consegue apontar pra algo que não é página.

O 404 é a única página que não emite nada. Ela não está na árvore de conteúdo, então uma migalha ali descreve uma hierarquia que não a contém, e um nó `WebSite` convida o crawler a tratar um erro como documento.

Uma coisa que só apareceu quando fui escrever o teste: `truncate` escapa uma string simples e deixa um `template.HTML` em paz. O `headline` é o único campo que o tema transforma em vez de copiar, e ele saía como `Vue &amp;amp; Vitest` pra um título tão comum quanto `Vue & Vitest` — entidade HTML dentro de string JSON, onde o consumidor lê literal, contradizendo o `name` construído do mesmo título no mesmo nó. O `name` tinha o problema oposto, carregando qualquer markup que o front matter escrevesse.

## llms.txt, e o que o spec pede de verdade

Os três arquivos são Output Formats do Hugo. O `/llms.txt` é o índice que um answer engine lê num pedido em vez de rastrear: título, resumo e cada post como item linkado com uma linha de contexto. O `/llms-full.txt` é o conteúdo por trás desses links num arquivo só, cada post precedido da própria URL canônica. E cada post publica um `index.md` ao lado do HTML.

Tudo por idioma. Este site publica `/llms.txt` e `/en/llms.txt`, cada um listando os seus próprios posts.

Duas decisões que pareciam detalhe e não eram:

O `llms.txt` linka pros gêmeos markdown e não pro HTML, que é o que o spec pede — e a URL canônica é a primeira linha dentro de cada gêmeo, pra que uma citação que siga o link ainda saiba pra onde apontar.

Os posts no `llms-full.txt` são separados por uma linha `--- post: <url> ---` em vez de um `---` pelado. Divisória temática é markdown comum, que alguém escreve dentro de um post sem pensar neste arquivo — e era indistinguível da linha que separa dois posts. Sublinhado de heading em setext também era.

E todos os três respeitam o `[params.aeo] disallow` e a mesma condição de build que o `robots.txt` usa. Um caminho excluído dos crawlers cujo corpo inteiro está no `llms-full.txt` não está excluído, e answer engine é justamente a audiência que a chave nomeia.

## O checker também erra

Vale dizer, porque eu perdi tempo com isso: parte da distância entre 55 e 100 era bug do verificador, não do tema.

O aeo.js reprovou "Canonical URL set", "JSON-LD schema found" e "Description length" — os três corretos no site. As regex dele exigem atributo com aspas, e `hugo --minify` emite `rel=canonical` sem aspas, que é HTML5 válido. Ele também varre `new URL(target).origin`, então um site publicado sob subcaminho é avaliado em arquivos que ele nem consegue ver.

O score sai no summary do job de deploy, com essas duas limitações impressas do lado. Como informação, nunca como gate. O check que manda é o `check_aeo.py` do próprio repo, que roda sobre os bytes que estão indo pro ar.

## Onde isso está agora

A v0.6.0 está publicada, este site já roda ela, e a issue que puxou tudo isso está fechada. O tema continua com `hugo` como único requisito de build e sem nenhuma requisição a terceiros — agora inclusive a fonte.

Se você usa o terminal-mono e alguma coisa aqui quebrou pro seu lado, [abre uma issue](https://github.com/adamsalves/terminal-mono/issues). A lista de mudanças completa, com os números e o que não sobreviveu à medição, está no [CHANGELOG](https://github.com/adamsalves/terminal-mono/blob/main/CHANGELOG.md).
