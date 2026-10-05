# Finanças da Família — versão completa (Etapas 1 a 4)

Sistema web de controle financeiro familiar que roda no **Google Apps Script** e guarda os dados numa **planilha do Google** que pertence a você. Funciona no celular, tablet e computador.

**O que a Etapa 1 entrega**

- Login com a conta Google de cada pessoa (sem senhas novas)
- Tela de criação da família, convite de membros e perfis (Administrador, Membro, Visualizador)
- Dashboard com 8 indicadores clicáveis, 6 gráficos, mensagens automáticas e filtros (mês, ano, período, pessoa, categoria, conta)
- Lançamento rápido de receitas e despesas pelo botão **+**, com sugestão de categoria pelo histórico
- Listas de receitas e despesas com editar, duplicar, marcar como paga/recebida, excluir e desfazer
- Contas bancárias com saldo inicial
- As 17 abas da planilha criadas automaticamente, histórico de alterações e dados de demonstração removíveis
- Tema claro e escuro

**O que a Etapa 2 acrescenta**

- **Contas a pagar e a receber**: abas separadas, alertas (atrasadas, hoje, amanhã, 3 e 7 dias), dias restantes, **pagamento parcial**, adiar vencimento e **calendário mensal**
- **Cartões de crédito**: vários cartões, faturas com fechamento e vencimento, **compras parceladas** distribuídas sozinhas nas faturas, pagamento total ou parcial da fatura, limite usado/disponível e alertas em 70%, 85% e 95%
- **Transferências** entre contas e pessoas da família (não contam como receita nem despesa)
- **Reservas**: saldo de cada conta, entradas e saídas, saldo projetado, extrato e **conferência com o banco** (conciliação)
- **Lançamentos recorrentes** (semanal, quinzenal, mensal, trimestral, anual) gerados sozinhos, sem duplicar
- **Despesas parceladas** (carnê, boleto) e **categorias personalizadas**

---

**O que a Etapa 3 acrescenta**

- **Orçamento** por categoria e subcategoria, com barra de progresso e alertas em 70%, 85%, 100% e ao ultrapassar; copiar do mês anterior
- **Metas** (reserva de emergência, viagem, imóvel…): progresso, prazo restante, **quanto guardar por mês**, contribuições, retiradas e ajustes com histórico
- **Dívidas e empréstimos**: saldo devedor, parcelas pagas e restantes, juros separados da amortização, **amortização extra** (reduzindo o prazo ou a parcela) e gráfico do saldo caindo
- **Relatórios** (12 tipos) com filtros, **comparação entre períodos** e exportação para **Excel, PDF e CSV**, além de impressão

---

**O que a Etapa 4 acrescenta**

- **Central de notificações** no sino, com "marcar como lida" individual e escolha dos avisos de cada pessoa
- **Aviso de possíveis lançamentos duplicados**
- **Histórico de alterações** na tela, com filtros
- **Cópias de segurança** (manuais ou semanais) e exportação completa em Excel/CSV
- **Automação diária** com registro de cada execução
- **Revisão de segurança**: quem tem acesso à planilha, correções com um toque, proteção das abas e registro de edições feitas à mão

---

## 1. Instalação (uma vez, pelo administrador — cerca de 10 minutos)

Faça tudo **no computador**, com a conta Google que será a dona dos dados.

### Passo 1 — Criar a planilha
1. Acesse **sheets.google.com** e crie uma **planilha em branco**.
2. Dê um nome a ela, por exemplo *Finanças da Família*.

### Passo 2 — Abrir o Apps Script
1. Na planilha, clique em **Extensões ▸ Apps Script**.
2. Uma nova aba abre com um arquivo chamado `Código.gs`.

### Passo 3 — Colar os arquivos
Crie estes 5 arquivos no editor, copiando o conteúdo de cada arquivo recebido:

| No editor do Apps Script | Como criar | Copiar o conteúdo de |
|---|---|---|
| `Código.gs` | já existe — apague tudo e cole | `Codigo.gs` |
| `Index` | botão **＋ ▸ HTML**, nome `Index` | `Index.html` |
| `Estilos` | botão **＋ ▸ HTML**, nome `Estilos` | `Estilos.html` |
| `App` | botão **＋ ▸ HTML**, nome `App` | `App.html` |
| `appsscript.json` | veja abaixo | `appsscript.json` |

> Os nomes `Index`, `Estilos` e `App` precisam ser **exatamente** esses (com maiúsculas), sem “.html” — o editor adiciona sozinho.

Para o `appsscript.json`: clique em **⚙️ Configurações do projeto** (menu da esquerda), marque **“Mostrar arquivo de manifesto appsscript.json no editor”**, volte ao **Editor**, abra o `appsscript.json`, apague tudo e cole o conteúdo recebido.

Clique em **💾 Salvar projeto**.

### Passo 4 — Executar a instalação
1. No topo do editor, na lista de funções, escolha **`instalar`** e clique em **▶ Executar**.
2. O Google pedirá autorização: **Revisar permissões ▸ escolha sua conta ▸ Avançado ▸ Acessar (não seguro) ▸ Permitir**.
   *O aviso de “app não verificado” aparece porque o script é seu, não de uma empresa. É normal.*
3. Volte à planilha: as 17 abas estarão criadas.

### Passo 5 — Publicar como aplicativo
1. No editor, clique em **Implantar ▸ Nova implantação**.
2. Na engrenagem ⚙️ ao lado de “Selecionar tipo”, escolha **App da Web**.
3. Preencha:
   - **Executar como:** *Usuário que acessa o app da Web*
   - **Quem pode acessar:** *Qualquer pessoa com Conta do Google*
4. Clique em **Implantar** e copie a **URL do app da Web** (termina em `/exec`).

> “Qualquer pessoa com Conta do Google” **não** abre seus dados: quem não foi convidado vê a tela “Acesso não liberado”, porque a planilha só é compartilhada com os membros convidados.

### Passo 6 — Criar a família
1. Abra a URL no navegador.
2. Preencha o nome da família, seu nome e, se quiser, a primeira conta bancária.
3. Marque **“Incluir dados de demonstração”** se quiser ver o sistema já com exemplos (dá para remover depois em Configurações).

### Passo 7 — Colocar no celular como app
- **Android (Chrome):** abra a URL ▸ menu ⋮ ▸ **Adicionar à tela inicial**.
- **iPhone (Safari):** abra a URL ▸ botão Compartilhar ▸ **Adicionar à Tela de Início**.

---

## 2. Convidar seu esposo (ou outros membros)

1. No sistema, vá em **⚙️ Configurações ▸ Membros**.
2. Informe nome, **e-mail Google** da pessoa e o perfil (**Membro** registra e edita; **Visualizador** só consulta).
3. Clique em **Convidar**. O sistema compartilha a planilha automaticamente e mostra o link — use **“Enviar pelo WhatsApp”**.
4. A pessoa abre o link, entra com a conta Google convidada, autoriza (mesmo aviso do Passo 4) e clica em **Aceitar convite**.

Para tirar o acesso: **Revogar acesso** — a planilha deixa de ser compartilhada com aquele e-mail.

---

## 3. Uso no dia a dia (manual simplificado)

- **Registrar um gasto:** toque em **+ ▸ Despesa**, digite o valor (só números: `8990` vira `89,90`), a descrição e a categoria. Se já pagou, deixe **Paga**; se é um boleto, escolha **A pagar** e informe o vencimento.
- **Registrar uma entrada:** **+ ▸ Receita**. Se ainda vai receber, marque **Prevista**.
- **Conta chegou:** **+ ▸ Conta a pagar**. Quando pagar, toque em **✓ Paguei** na lista.
- **Corrigir algo:** toque no lançamento para editar, duplicar ou excluir. Depois de excluir aparece **Desfazer** por alguns segundos.
- **Ver de onde vem um número:** toque em qualquer card do Início. Saldo, economia e renda comprometida mostram a conta feita.
- **Trocar o mês:** setas ‹ › no topo. Para o ano inteiro ou um intervalo, use **Ano** ou **Período**.
- **Atualização:** se outra pessoa lançar algo, sua tela atualiza sozinha em até 30 segundos. O botão ⟳ atualiza na hora.
- **Indicador no topo:** 🟢 Sincronizado · 🔵 Sincronizando · 🔴 Erro (toque para tentar de novo) · 🟠 Aguardando conexão.

### Como os números são calculados
| Indicador | Fórmula |
|---|---|
| Saldo disponível | saldo inicial das contas + receitas **recebidas** − despesas **pagas** |
| Economia | receitas recebidas − despesas do período (pagas ou em aberto) |
| Renda comprometida | despesas (assumidas ou pagas, você escolhe em Configurações) ÷ receitas × 100 |
| Contas pendentes | despesas “A pagar” com vencimento até o fim do período (inclui as atrasadas) |

Valores são calculados em centavos inteiros, sem erros de arredondamento. Transferências entre contas da família se anulam no total.

---

## 3.1 Novidades da Etapa 2 no dia a dia

- **Pagar uma parte de uma conta:** em **Contas a pagar**, toque na conta, digite o valor pago (menor que o total) e confirme. O que falta continua em aberto. Para corrigir, use **Desfazer** no pagamento.
- **Compra no cartão:** **+ ▸ Compra no cartão**. Escolha o cartão, o valor total e as parcelas. O sistema mostra em qual fatura cai a primeira parcela.
- **Regra da fatura:** compras feitas **antes** do dia de fechamento entram na fatura daquele mês; no dia do fechamento ou depois, vão para a seguinte.
- **A compra no cartão conta como despesa uma vez só** (no mês de cada parcela). Ela só sai do saldo da conta quando você **paga a fatura** em **Cartões ▸ Pagar fatura**.
- **Contas fixas que se repetem** (aluguel, salário, internet): no lançamento, escolha **Repetir ▸ Todo mês**. O sistema cria sozinho os próximos lançamentos até o fim do mês seguinte, uma vez por dia, quando alguém abre o app. Para parar, edite um deles e escolha **Não repete**.
- **Mudou o valor de uma conta fixa?** Edite a próxima e marque **"Aplicar também às próximas ocorrências em aberto"**.
- **Conferir com o banco:** em **Reservas**, toque na conta, abra **Conferir com o banco** e digite o saldo que aparece no app do banco. Se houver diferença, o sistema avisa e oferece lançar um ajuste.
- **Cartões e categorias** só o administrador cadastra. Membros registram compras, pagamentos e transferências.

## 3.2 Novidades da Etapa 3 no dia a dia

- **Orçamento:** em **Metas ▸ Orçamento**, toque em **+ Orçar categoria** e diga quanto querem gastar no mês (ex.: Alimentação R$ 1.500). Pode orçar a categoria inteira ou só uma subcategoria (ex.: Delivery). No mês seguinte, use **Copiar do mês anterior** e ajuste.
- **O que conta como gasto do orçamento:** todas as despesas do mês pela data da compra — pagas, em aberto e parcelas de cartão.
- **Metas:** em **Metas ▸ Metas**, crie o objetivo com valor e prazo. O app mostra quanto guardar por mês. Quando separar dinheiro, toque em **+ Guardar**. As metas acompanham o dinheiro separado; elas **não** mexem no saldo das contas.
- **Dívidas:** cadastre com o **saldo devedor de hoje** (o que o banco diz que falta) e, se souber, os **juros ao mês**. Ao pagar uma parcela, deixe marcado **"Lançar também como despesa paga"** para o valor sair do saldo da conta.
- **Amortizar:** pagar um valor a mais abate direto do saldo. Escolha **reduzir o prazo** (economiza mais juros) ou **reduzir a parcela**.
- **Relatórios:** escolha o tipo, o período e, se quiser, **Comparar com** (período anterior, mesmo período do ano passado ou datas à sua escolha). **Excel/PDF/CSV** criam uma planilha no seu Google Drive e oferecem o download. **Imprimir** abre a impressão do navegador — dá para escolher "Salvar como PDF".

## 3.3 Novidades da Etapa 4 no dia a dia

- **🔔 Sino = central de notificações.** O número vermelho mostra quantos avisos você ainda não leu: contas vencendo (hoje, amanhã, em 3 e em 7 dias), atrasadas, faturas, limite do cartão, orçamento no limite, metas, parcelas de dívidas e receitas que não entraram.
- **Marcar como lida:** toque no ✓ ao lado do aviso (ou em **Marcar todas como lidas**). Vale **só para você** — seu esposo continua vendo os dele. Se a situação piorar (ex.: de "vence amanhã" para "vence hoje"), um aviso novo aparece.
- **Escolher avisos:** no sino, aba **⚙️ Escolher avisos** (ou em Configurações ▸ Meus avisos). Cada pessoa escolhe os seus.
- **👯 Possível duplicado:** quando dois lançamentos têm o mesmo valor, descrição parecida e datas próximas (ex.: vocês dois lançaram a mesma compra), o app avisa. Abra, compare e exclua um — ou toque em **Não é duplicado**.
- **Para o administrador, em Configurações:**
  - **🛡️ Segurança e acesso:** mostra quem consegue abrir a planilha e aponta problemas com botão de correção (pessoa estranha com acesso, alguém revogado que ainda acessa, visualizador que pode editar…).
  - **🔒 Proteger agora:** esconde as abas de dados e mostra um aviso a quem tentar editar a planilha à mão. O app continua gravando normalmente. Edições feitas à mão ficam registradas no histórico.
  - **💾 Cópias de segurança:** **Fazer cópia agora** cria uma cópia completa no seu Drive, com download em Excel (tudo) ou CSV (aba por aba). Dá para ligar a **cópia automática semanal**.
  - **⚙️ Automação diária:** cria os lançamentos recorrentes e a cópia semanal mesmo sem ninguém abrir o app (veja como ativar abaixo).
  - **🕑 Histórico de alterações:** quem fez o quê e quando, com filtros por pessoa, tipo e data.

### Ativar a automação diária (opcional, uma vez, no computador)
1. Na planilha, abra **Extensões ▸ Apps Script**.
2. No menu da esquerda, clique no relógio **⏰ Acionadores** e depois em **+ Adicionar acionador** (canto inferior direito).
3. Escolha: função **automacaoDiaria** · origem do evento **Baseado no tempo** · tipo **Contador de dias** · horário **6h às 7h**.
4. Clique em **Salvar** e autorize com a sua conta. Em Configurações ▸ Automação diária aparece a data da última execução.

## 4. Atualizar o sistema nas próximas etapas

Quando receber arquivos novos: cole-os por cima no editor, salve e vá em **Implantar ▸ Gerenciar implantações ▸ ✏️ Editar ▸ Versão: Nova versão ▸ Implantar**. **A URL continua a mesma** e nenhum dado é perdido — as abas só ganham colunas novas, nunca perdem.

**Da Etapa 3 para a 4** mudam os mesmos três arquivos (`Código.gs`, `App` e `Estilos`). Nenhuma coluna nova e nenhuma permissão nova do Google.

**Da Etapa 2 para a 3** também só mudam `Código.gs`, `App` e `Estilos`. Se o Google pedir autorização de novo ao abrir, é porque o app passou a criar planilhas de relatório no seu Drive — pode permitir.

**Da Etapa 1 para a 2** só mudam três arquivos: `Código.gs`, `App` e `Estilos`. `Index` e `appsscript.json` continuam iguais. Na primeira abertura depois da atualização, o sistema acrescenta sozinho as colunas novas nas abas.

---

## 5. Segurança e limites — leia uma vez

- **Quem acessa:** só contas Google convidadas. Cada ação é verificada no servidor (perfil, família, dono do lançamento) — esconder botões é só conveniência.
- **Nada sensível no navegador:** não há chaves nem senhas no código da página.
- **Duas pessoas ao mesmo tempo:** as gravações passam por um bloqueio; se alguém editar o mesmo lançamento que você, o sistema avisa em vez de sobrescrever.
- **Internet caiu:** o sistema tenta de novo sozinho e **não duplica** lançamentos.
- **Exclusões** ficam registradas (a linha é marcada, não apagada) e tudo vai para a aba **Histórico de alterações**.
- ⚠️ **Por que os membros têm acesso à planilha?** Para o app reconhecer cada pessoa pela própria conta Gmail, o Google exige que a planilha esteja compartilhada com ela. Não dá para eliminar isso sem um servidor próprio. O que o app faz para reduzir o risco:
  - **Proteger a planilha** (Configurações ▸ Segurança) esconde as abas de dados e mostra um aviso a quem tentar editar à mão;
  - toda edição manual fica registrada no **Histórico de alterações** e aparece como alerta na revisão de segurança;
  - a **revisão de acesso** mostra quem consegue abrir a planilha e remove quem não deveria com um toque;
  - visualizadores recebem acesso só de leitura.
- **Cópia de segurança:** em Configurações ▸ Cópias de segurança (manual ou semanal automática). O Google também guarda o histórico de versões (**Arquivo ▸ Histórico de versões**), útil para voltar a planilha a um dia anterior.
- **Notificações lidas e preferências** ficam guardadas na conta Google de cada pessoa (não na planilha).
- **Desempenho:** confortável até algumas dezenas de milhares de lançamentos. A camada de armazenamento está isolada no código para uma migração futura para banco de dados, se um dia for preciso.

---

## 6. Problemas comuns

| Sintoma | Solução |
|---|---|
| “Acesso não liberado” | A pessoa entrou com outra conta Google. Troque de conta no navegador ou convide o e-mail certo. |
| “Não foi possível identificar sua conta Google” | Faça login no Google no navegador e abra o link de novo. |
| Página em branco / erro `include` | Confira se os arquivos se chamam exatamente `Index`, `Estilos` e `App`. |
| Mudanças no código não aparecem | Faltou criar **Nova versão** em Gerenciar implantações. |
| Gráficos não aparecem | A rede bloqueou a biblioteca de gráficos (cdn.jsdelivr.net). Teste em outra rede. |

---

## 7. Arquivos

| Arquivo | Função |
|---|---|
| `Codigo.gs` | Servidor: API, login, permissões, cálculos, planilha |
| `Index.html` | Estrutura da página |
| `Estilos.html` | Visual (claro/escuro, celular/computador) |
| `App.html` | Telas e interações |
| `appsscript.json` | Configuração do app da Web |
| `testes/teste-backend.js` | 20 testes automáticos da Etapa 1 |
| `testes/teste-etapa2.js` | 31 testes automáticos da Etapa 2 (faturas, pagamentos parciais, saldos, recorrências, permissões) |
| `testes/teste-etapa3.js` | 21 testes automáticos da Etapa 3 (orçamento, metas, juros e amortização, relatórios, exportação) |
| `testes/teste-etapa4.js` | 14 testes automáticos da Etapa 4 (notificações por pessoa, duplicados, histórico, backup, automação, acesso e proteção) |

## 8. Ideias para o futuro
- Envio de comprovantes direto pelo app (hoje: link do Drive no lançamento). Exige uma permissão extra do Google Drive.
- Avisos por e-mail ou WhatsApp. Exige permissões extras e um serviço de envio.
- Banco de dados próprio, se o volume de lançamentos crescer muito ou se quiserem tirar o acesso dos membros à planilha.
