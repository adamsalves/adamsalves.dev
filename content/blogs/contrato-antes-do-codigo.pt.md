+++
title = "Contrato antes do código: como eu conduzo IA nos projetos"
date = 2026-08-27
draft = false
description = "As técnicas que eu de fato uso no dia a dia com IA — contexto como orçamento, recuperação antes de geração, subagente com contexto limpo e loop com verificação — e os números que elas produziram em 7.905 comandos."
tags = ["ai", "rag", "agents", "workflow"]
toc = true
+++

Quase tudo que se escreve sobre trabalhar com IA fala de prompt. Na prática, o prompt é a menor das decisões. O que decide o resultado é o que entra no contexto, em que ordem, quem verifica a saída, e quantas vezes o ciclo roda antes de alguém aceitar o trabalho.

Este post é sobre as técnicas que sobraram depois de um ano fazendo isso todo dia, com os números que elas produziram.

## Contrato antes do código

A primeira técnica não tem nada de técnica: escrever a decisão antes de pedir a execução. Um documento curto com duas seções que fazem o trabalho pesado, "Decisões Fechadas" e "Fora de Escopo", e nada começa antes dele existir.

Isso resolve um problema específico do trabalho com agente. Um modelo é extraordinariamente bom em construir o que você pediu, inclusive quando o que você pediu está errado, e ele não vai parar no meio para perguntar se a premissa faz sentido. Escrever a decisão antes é o único momento em que o custo de mudar de ideia ainda é uma linha de texto.

O "Fora de Escopo" carrega mais peso do que parece. Sem ele, um agente com contexto de sobra encontra trabalho adjacente para fazer, e o diff cresce em direções que ninguém pediu.

## Recuperação antes de geração

Grep é busca por string. Quando você pergunta "quem chama esta função", o grep devolve toda linha que contém o nome dela — a definição, os comentários, a string de log, o teste que menciona no título — e o modelo lê tudo isso para descartar quase tudo.

Em vez disso, os repositórios em que trabalho carregam um grafo do próprio código, construído com Tree-sitter e consultado antes de qualquer leitura de arquivo. O maior deles tem 8,5 MB de nós e arestas indexados. As perguntas que eu faço a ele não são textuais, são estruturais: quem chama isto, o que isto importa, quais testes cobrem isto, qual o raio de impacto desta mudança.

É RAG, no sentido que importa: recuperar o pedaço certo antes de gerar, em vez de despejar o repositório no contexto e esperar atenção. A diferença em relação ao RAG de documento é que o índice aqui não é um embedding de texto solto — é a estrutura real do código, então "quem chama esta função" tem uma resposta exata em vez de uma aproximação por similaridade. Busca semântica existe por cima, para quando eu não sei o nome do que procuro.

O efeito prático: uma pergunta que custava ler seis arquivos passa a custar uma consulta e uma resposta de algumas centenas de tokens.

## Onde o token realmente vaza

Toda operação de shell que eu rodo passa por um proxy que filtra a saída antes dela chegar ao contexto. Ao longo de 7.905 comandos, ele cortou **3,0 milhões de tokens, 47,3% do total**.

O interessante não é a média, é a distribuição:

```
test runner              123 execuções    98,2%
gh pr diff                73 execuções    24,5%
grep                     511 execuções    10,3%
leitura de arquivo     1.240 execuções     8,3%
```

Cento e vinte e três execuções de teste economizaram 1,6 milhão de tokens sozinhas, mais da metade de tudo. Ler arquivo, feito dez vezes mais, economizou 250 mil.

A lição está na diferença entre 98,2% e 8,3%. A saída de um test runner é barra de progresso, timing por arquivo, stack trace repetida e um resumo — quase tudo formatação, e o sinal cabe em três linhas. Código-fonte é quase todo sinal, e tentar comprimir aquilo custa entendimento. Otimização de contexto que rende mora no ruído das ferramentas, não no material que você quer que o modelo entenda.

## Subagente é isolamento, não paralelismo

A parte "agentic" do meu trabalho é menos empolgante do que o termo sugere. Não é uma frota de agentes cooperando. São duas coisas.

A primeira é despachar cada task para um subagente com contexto próprio, para que o entulho de uma não contamine a próxima. A segunda é a que importa de verdade: **o code review roda num subagente que nunca viu o código ser escrito.**

Isso não é cerimônia. Um modelo revisando o próprio trabalho carrega no contexto o raciocínio que produziu aquele código, e esse raciocínio é exatamente o que precisa ser questionado. Ele relê e concorda consigo mesmo. Com contexto limpo, o mesmo modelo lê o diff como código de estranho e devolve veredito explícito — "alterações necessárias" ou "aprovado com sugestões". Assim já saíram falha de autorização entre inquilinos e credencial em log, achados que a sessão que escreveu o código tinha deixado passar.

O gate final não é agente nenhum: nunca commit direto na main, toda mudança por PR. A IA propõe e não publica.

## O loop, e o orçamento escrito dentro dele

Loop engineering, na prática, é decidir onde o ciclo para. O meu é plano em checkbox, uma task por vez, cada uma em TDD com o vermelho e o verde registrados, e uma conferência explícita antes do commit.

A conferência é o passo que quase todo mundo pula, e é o único que impede o loop de convergir para "o agente disse que passou". Duas verificações minhas custaram um incidente cada: conferir o `ls` do diretório contra a lista de arquivos do plano, porque subagente cria arquivo fora da convenção e o diff não mostra o que você não pediu; e reproduzir o ambiente de CI antes de declarar verde, porque o test runner local lê variável de ambiente que o CI não tem.

O que eu não esperava é que o orçamento de contexto precisasse ficar escrito dentro do próprio loop. As skills que eu carrego em todo projeto terminam com um bloco literal:

> Target: complete any review/debug/refactor task in ≤5 tool calls and ≤800 total output tokens.

Sem um teto declarado, um agente com ferramentas boas usa todas elas. O limite não torna a resposta pior, torna a busca mais deliberada — e a instrução que vem antes dele, "comece sempre por `get_minimal_context`", é o que faz caber.

E há um loop mais lento por fora de todos: cada correção que eu dou vira memória e passa a valer nas sessões seguintes. São 159 acumuladas. O ganho não é o modelo aprender, é eu não digitar a mesma correção pela quarta vez.

## Onde isso encosta no trabalho

Nenhuma dessas técnicas é sobre escrever código mais rápido. Elas são sobre a única coisa que fica cara quando o código passa a aparecer rápido: decidir o que aceitar.

Contrato antes da execução decide o que vale construir. Recuperação estruturada e filtro de saída decidem o que o modelo vê. Contexto limpo no review e gate humano no merge decidem o que entra. O loop com verificação decide quando parar.

O custo é real e vale dizer: escrever a decisão antes leva tempo que em tarefa pequena não se paga, e eu não faço. Em mudança grande, sai mais barato que descobrir na metade.
