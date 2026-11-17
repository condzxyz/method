# Monte o seu canal diário no WhatsApp (do jeito que eu montei)

*por Conrado de Sá · @conrado_desa*

Eu montei um WhatsApp que conversa sozinho com quem chega pelo link da bio, coloca a pessoa numa lista de espera, libera em turmas, manda uma mensagem por dia no horário certo e cobra o plano pago. Nenhuma conversa passa por IA: tudo é roteirizado, então o custo por mensagem é zero.

**Como usar este texto:** abre o **Claude Code** (o app do Claude no computador, aba Code, ou no terminal), cola este arquivo inteiro e manda. Ele vai te fazer umas perguntas e montar tudo junto com você, uma fase de cada vez. Você vai precisar de um computador que fique ligado de manhã (Mac ou PC) e de um número de WhatsApp só pra isso.

> Aviso honesto: isto usa o WhatsApp por uma rota **não oficial** (a biblioteca Baileys, a mesma do WhatsApp Web). Funciona, mas vai contra os termos do WhatsApp e o número pode ser bloqueado. Por isso: número dedicado, nunca o seu pessoal, e só pra quem pediu pra receber. Se crescer muito, o caminho é migrar pra API oficial da Meta.

---

## Para o Claude: o que você vai construir comigo

Você é o meu engenheiro neste projeto. Vamos construir um **canal diário de conteúdo no WhatsApp**: a pessoa entra por um link, conversa com um roteiro automático, entra numa lista de espera e passa a receber uma mensagem por dia (ou por semana, no grátis). Trabalhe em fases, com testes em cada uma, e me mostre o resultado antes de passar pra próxima.

### Antes de escrever qualquer código, me pergunte (uma pergunta por vez, com sugestão):

1. **Sobre o que é o canal** e o que chega em cada mensagem (ex.: uma frase + um aprendizado curto + um exercício de 2 a 5 minutos).
2. **Nome** do canal e **horário** do envio (ex.: 7h, fuso de Brasília).
3. **Modelo:** grátis (ex.: 1 por semana) e pago (ex.: todo dia, R$ X por mês)? Ou só grátis no começo?
4. **Lista de espera:** quantas vagas na primeira turma (ex.: 10)? Dar alguns dias de cortesia do diário pra primeira turma (pra ter feedback rápido)?
5. **Pergunta da conversa de entrada:** o que eu pergunto pra pessoa logo no começo (ex.: "o que mais pesa na sua cabeça hoje?", com 4 ou 5 opções numeradas)?
6. **Onde roda:** meu computador fica ligado no horário do envio? É Mac ou Windows?
7. **Contas que eu já tenho:** Cloudflare, GitHub, Stripe. As que faltarem, me diga como criar.
8. **Número:** chip físico ou número virtual (algumas operadoras vendem número virtual que recebe o SMS de verificação num painel, sem chip)?

Depois disso, me mostre um resumo do que entendeu e o plano das fases. Só comece quando eu disser "pode".

---

## Regras que não mudam

**Respeito e anti-bloqueio**
- Número **dedicado**, com o **WhatsApp Business** instalado num celular. Nunca o número pessoal.
- **A pessoa sempre escreve primeiro.** O bot nunca puxa conversa com quem não escreveu.
- No máximo **1 mensagem de conteúdo por dia** por pessoa.
- **SAIR** (e PARAR, CANCELAR, "não quero mais") funciona sempre, a qualquer momento, e corta tudo que estava na fila.
- Envio da leva do dia com **espaço aleatório de 20 a 60 segundos** entre uma pessoa e outra, com "digitando…" antes.
- **Teto diário** de mensagens da leva (ex.: 150) e **teto de 12 respostas automáticas por pessoa por dia** (contra loop com outro robô ou alguém mandando sem parar).
- Só conversa 1 a 1. Nada de grupo, lista de transmissão ou status.

**Custo e previsibilidade**
- **Sem IA na conversa.** Tudo é roteiro com opções numeradas e palavras-chave. Texto livre fica guardado pra eu ler; o bot responde no máximo um "recebi" por dia.
- Nenhum conteúdo sai sem estar marcado como **aprovado** por mim.

**Segurança e o que depende do meu OK**
- Segredos (tokens, chaves) **nunca aparecem** no chat, no log ou no código. Ficam em variáveis secretas. Chave do Stripe quem cria sou eu, com **permissão mínima**.
- Você **não faz sem o meu OK explícito:** subir o servidor, publicar a página, criar coisa no Stripe, parear o número, mandar mensagem pra pessoa real.
- LGPD: página de privacidade dizendo o que guarda (número, nome do perfil, respostas) e como sair. Não guarde mais que isso.

---

## Arquitetura

```
link da bio / página ──► WhatsApp do bot ◄──► serviço no meu computador (Baileys)  ◄──►  servidor na nuvem (Cloudflare Workers + D1)
                                               puxa a fila a cada 1 min                inscritos, fila, conteúdo, pagamentos,
                                               manda com espaço aleatório              decide quem recebe o quê e quando
                                               repassa o que chega                              ▲
                                                                                  Stripe ──────┘ (webhook: pagou → libera)
```

- **Servidor na nuvem** (Cloudflare Workers + banco D1, TypeScript). O plano grátis dá conta no começo. Guarda tudo e decide a agenda. Rotas protegidas por um token que só o computador conhece.
- **Serviço no computador** (Node, sempre ligado: launchd no Mac, Agendador de Tarefas ou PM2 no Windows). Mantém o WhatsApp conectado, puxa o que tem pra mandar, manda, confirma e repassa o que chega. Tem um **modo ensaio** que faz tudo igual, mas não manda nada pelo WhatsApp. Tem um painel local (ex.: http://127.0.0.1:8790) com conexão, fila, avisos e as últimas mensagens.
- **Página** estática (GitHub Pages serve) com o botão que abre o WhatsApp já com a mensagem escrita, e um **link curto** pra bio (uma página que só redireciona pro `wa.me/<número>?text=…`).
- **Stripe** (opcional, pro pago): Payment Links e webhook.

---

## O que o bot faz

1. **Conversa de entrada (o funil, sem IA)** pra quem chega pelo link da bio:
   - **Abertura:** quem eu sou, o que chega e quando, e a pergunta com opções numeradas (1 a 5).
   - **Convite:** uma frase sobre a resposta da pessoa, o que tem no grátis e no pago, "a primeira turma tem só N vagas", e "Quer entrar na lista de espera? Responde SIM".
   - **Entrou:** posição na fila ("por enquanto você está dentro das N vagas", se estiver) e um **link pessoal de indicação**: cada pessoa que entra por ele sobe quem indicou 3 posições.
   - NÃO encerra com educação. Resposta que não é SIM nem NÃO: pergunta de novo no máximo 2 vezes, depois só guarda.
2. **Lista de espera e turmas.** Comando no computador pra ver a fila, **prévia** de quem seria liberado, liberar os N primeiros (ou números específicos) e **agendar** uma liberação (ex.: quinta às 6h55). A boas-vindas sai com a leva, dentro do horário (ex.: 7h às 20h). Liberou à noite, sai às 7h do dia seguinte.
3. **Cortesia:** liberar com N dias de diário grátis (N mensagens), aviso nos 3 últimos dias e depois volta pro grátis.
4. **Envio:** pago recebe todo dia no horário; grátis recebe 1 por semana (ex.: segunda). O pago começa por uma **jornada** (ex.: 30 dias) e depois segue contínuo.
5. **Comandos:** `1` (fiz o exercício: conta a sequência de dias), `STATUS`, `SAIR`, `VOLTAR`, `AJUDA`, `DIÁRIO`/`ASSINAR` (link do pago ou, se ainda fechado, anota o interesse).
6. **Feedback:** pergunta "o que você está achando?" num dia fixo (ex.: dia 4 do diário e 2ª semanal); texto livre recebe "recebi" 1x por dia; comando pra eu ler tudo que escreveram, por turma.
7. **Pagamento:** link pessoal com `client_reference_id=<código da pessoa>` (liga o pagamento ao WhatsApp sozinho). Quem paga pela página sem código cai numa página de confirmação que mostra "ATIVAR <últimos 10 caracteres do checkout>" com botão pro WhatsApp. Cartão recorrente + Pix avulso (ex.: 30 dias). Quem paga sai da lista de espera na hora.
8. **Pago abre numa data:** antes disso, nenhuma mensagem fala de preço; na hora, a página troca "abre em breve" pelos botões e quem respondeu DIÁRIO recebe 1 aviso.
9. **Avisos pra mim** (notificação no computador e no painel): conteúdo sem aprovar perto de sair, conteúdo acabando, pagou e não ativou, número recusado, WhatsApp desconectado.

---

## Banco (D1)

- `assinantes`: endereço do WhatsApp (jid, e o alternativo), telefone, nome do perfil, código pessoal, status (ativo, espera, saiu, inválido), pago até, forma (cartão, pix, cortesia), contadores (dia da jornada, semanal), sequência, ordem na fila, indicações, quem indicou, resposta da pergunta de entrada, turma, quando foi liberado.
- `envios`: **fila de saída com lease** (pendente → entregue ao computador → enviado/falhou/cancelado), chave única por pessoa por dia (não duplica), prioridade (resposta antes da leva), tentativas.
- `recebidas` (com id da mensagem, pra não processar 2x), `desconhecidos` (etapa da conversa de entrada de quem ainda não entrou), `pagamentos`, `alertas`, `stripe_eventos` (idempotência), `kv`.

**Confiabilidade:** o contador da jornada só anda quando o computador **confirma** que mandou. Envio pego e não confirmado volta pra fila depois do prazo; 4 tentativas e desiste com aviso. Mensagem do dia que não saiu até o horário de corte (ex.: 20h) fica pro dia seguinte, **sem pular conteúdo**. Webhook do Stripe idempotente e com verificação de assinatura.

---

## WhatsApp: número e pareamento (o que eu aprendi errando)

- No celular: **WhatsApp Business** com o número do bot. Ele é um app separado do WhatsApp normal: **não aceite "usar o mesmo número"** quando ele oferecer, toque em "usar outro número". Ative a **verificação em duas etapas**. Abra o app pelo menos 1x por semana (se o celular principal ficar uns 14 dias sem conectar, o WhatsApp desconecta os aparelhos ligados).
- O computador entra como **aparelho conectado**, pareado **por código de 8 letras** (`requestPairingCode`), não por QR (o QR falhou pra mim). Pegue a versão atual do WhatsApp Web antes de conectar.
- Reconexão com espera crescente (5 s até 5 min). **Pare de tentar** em "desconectado" (401), "proibido" (403) e "conexão substituída" (440), e me avise.
- Nunca rode o pareamento e o serviço ao mesmo tempo. Pareia primeiro, liga o serviço depois.
- Impede o computador de dormir enquanto o serviço roda (no Mac, `caffeinate`), e me lembre de deixar na tomada.

---

## Conteúdo

- Um arquivo `CONTEUDO.md` com cada mensagem: id, tema, o texto principal, a fonte (se for citação: livro e página ou versículo, **conferida**, nada de frase "atribuída"), as linhas de aprendizado, o exercício e **Status: rascunho/aprovado**.
- Um script que confere o arquivo (campos, tamanho, fonte, palavras proibidas que eu definir) e gera o conteúdo do servidor. Outro que gera uma **prévia** de como cada mensagem chega no WhatsApp (negrito com `*`, itálico com `_`).
- Sem promessa de saúde ou de resultado. Se for um tema sensível, uma linha "isto não substitui um profissional" na página.

---

## Ordem de construção

1. **Servidor + banco + testes** com um SQLite em memória no lugar do D1 e um Stripe de mentira. Pronto quando: todos os testes passam e o TypeScript compila.
2. **Serviço do computador em modo ensaio** (WhatsApp falso que grava num arquivo). Pronto quando: um ensaio pela rede com o servidor local (`wrangler dev`) faz entrar, liberar, responder 1, STATUS e SAIR.
3. **Simulação de uma semana** com o relógio acelerado e pessoas fictícias, com checagens automáticas: ninguém recebe mais de 1 conteúdo por dia, espaço ≥ 20 s, nada antes do horário, nada depois do SAIR, computador desligado de manhã e a mensagem sai quando volta.
4. **Número e pareamento** (com o meu OK): WhatsApp Business no celular, código de 8 letras, serviço ligado.
5. **Servidor no ar** (com o meu OK) e **link da bio** + página.
6. **Stripe** (com o meu OK): produto, preço recorrente e avulso, 2 Payment Links (nome e telefone obrigatórios, cupom permitido, volta pra página de confirmação), webhook com os eventos de pagamento, assinatura e reembolso. Teste com um cupom de 100% antes de divulgar.
7. **Primeira turma**: eu testo com o meu número, aprovo o conteúdo e libero (ou agendo) os primeiros.

Em cada fase: me diga o que fez, o que testou e o que falta, em português simples.

---

## Custos que eu tive

- Cloudflare (servidor e banco): o plano grátis dá conta no começo.
- Número virtual: cerca de R$ 30 por mês (ou um chip pré-pago).
- Stripe: só a taxa por venda.
- IA na conversa: zero, porque não tem.

---

*Se montar o seu, me marca no Instagram (@conrado_desa). Quero ver.*
