# FICHA DE DECISÕES · War Room FiapBank (CP6 · 3 aulas)

**Grupo (nome da equipe plantonista):** FIPAK

**Turma:** 2CCPO **Repo:** `cp6-warroom--fipak-`

**Integrantes (nome + RM):**

| Nome | RM |
|---|---|
| Antonio Lucas | 565516 |
| Bento Donato | 561621 |
| Enzo Ribeiro | 564216 |
| Guilherme Califoni | 565157 |
| Gustavo Braga | 562247 |
| Gustavo Schimith | 564800 |
| Kaio Correa | 563443 |
| Lucas Mendes | 563667 |

**Convenção de commits** (1 commit por rodada; relâmpagos podem ir junto com a rodada seguinte):

```
decisao: R1 - opcao C (<resumo da justificativa em uma frase>)
decisao: R2 - opcao A (<resumo em uma frase>)
pos-mortem: relatorio de incidente da madrugada
```

---

## 📁 Dossiê técnico do FiapBank (MVP em produção)

**Stack:** Java 17 + Spring Boot + Spring Data JPA + Oracle. API com endpoints em
`/api/contas` e `/api/transferencias` (cenário visto desde a Aula 13).

**Contrato e regras de negócio que o banco prometeu aos clientes e aos reguladores:**

| Regra | Como deve ser |
|---|---|
| Transferência **PIX** | taxa **R$ 0,00** |
| Transferência **TED** | taxa fixa **R$ 5,00** |
| Saldo | **nunca fica negativo**: transferência/saque sem saldo é recusado com erro claro |
| Número de conta | **sequencial e único** (1001, 1002, 1003...), gerado pelo sistema |
| Extrato de transferência | grava **quem pagou** e **quem recebeu**, na ordem certa |
| CPF | **dado sensível**: nunca aparece nas respostas da API |
| Consultas ao banco | sempre parametrizadas, e cada operação **usa e libera** a conexão |
| Suíte de testes | roda antes de todo deploy; **verde** é pré-requisito pra subir |

---

# 📝 REGISTRO DE DECISÕES

*Placar inicial: 🔥 7 · 💰 0 · 🧹 2*

---

## O saldo que virou negativo (19h12)

**Tipo:** ( rodada ) · **Voto:** ( C )

**Justificativa:**

O `catch (SaldoInsuficienteException)` vazio engolia a exceção que protegia a regra, e o `origem.debitar(valor + taxa)` rodava SEMPRE, deixando o saldo negativo. Corrigir a causa (a exceção deve interromper a operação) e escrever um teste com `assertThrows` trava a invariante e impede a regressão (Aulas 11 e 15). O trade-off é gastar 40 min a mais agora, em troca de subir com rede de segurança e reduzir a dívida técnica, em vez de um hotfix sem teste.

**Placar do grupo após esta decisão:** 🔥 8 (+1) · 💰 R$ 35 mil (+35) · 🧹 1 (-1)

---

## A injeção que derreteu o banco (21h02)

**Tipo:** ( rodada ) · **Voto:** ( A )

**Justificativa:**

A busca por titular concatenava a entrada do usuário no SQL (`"... WHERE titular = '" + nome + "'"`), então `' OR '1'='1` virava comando (SQL Injection). `PreparedStatement` com parâmetros vinculados separa estrutura e dados e neutraliza a injeção na raiz, cumprindo o contrato "consultas sempre parametrizadas". O trade-off é manter o JDBC puro, sem migrar para a derived query do Spring Data JPA, o que não reduz a dívida, em troca de uma correção rápida (~40 min) e de baixo risco de madrugada.

**Placar do grupo após esta decisão:** 🔥 8 (0) · 💰 R$ 35 mil (0) · 🧹 1 (0)

---

## ⚡ As conexões que não fecham (22h40) · Relâmpago B

**Tipo:** ( relâmpago ) · **Voto:** ( A )

**Justificativa:**

O hotfix de madrugada reescreveu o acesso JDBC sem `close()`, esgotando o pool de conexões ("pool exhausted"). `try-with-resources` fecha `Connection`, `Statement` e `ResultSet` (todos `AutoCloseable`) mesmo se houver exceção, garantindo a regra "cada operação usa e libera a conexão". O trade-off é gastar ~30 min corrigindo os 3 métodos, em vez de mascarar com um pool maior (B) ou piorar a latência com delay (C).

**Placar do grupo após esta decisão:** 🔥 8 (0) · 💰 R$ 35 mil (0) · 🧹 1 (0)

---

## A noite das contas gêmeas (22h50)

**Tipo:** ( rodada ) · **Voto:** ( B )

**Justificativa:**

O `getInstancia()` do Singleton devolvia sempre `new GeradorNumeroConta()`, então o contador recomeçava em 1001 e gerava contas duplicadas. Trocar por número aleatório/UUID elimina a colisão sem depender de estado compartilhado na aplicação e é rápido de implementar. O trade-off é que o número deixa de ser sequencial, quebrando a promessa do contrato e deixando extratos ilegíveis para auditoria (🧹 +1).

**Placar do grupo após esta decisão:** 🔥 8 (0) · 💰 R$ 35 mil (0) · 🧹 2 (+1)

---

## O comprovante que aponta pra pessoa errada (00h47)

**Tipo:** ( rodada ) · **Voto:** ( B )

**Justificativa:**

O extrato era gravado com `origem` e `destino` trocados na construção da `Transferencia` (os saldos estavam certos, só o registro estava invertido). Corrigir apenas a gravação (A) deixaria o histórico errado, e o contrato prevê que o extrato registra "quem pagou e quem recebeu, na ordem certa", o que interessa a clientes e reguladores. Corrigir e reprocessar os extratos antigos devolve a integridade total e reduz a dívida de dados (🧹 −1), ao custo de ~2h de trabalho (💰 +25 mil).

**Placar do grupo após esta decisão:** 🔥 9 (+1) · 💰 R$ 60 mil (+25) · 🧹 1 (-1)

---

## A fábrica que devolve null (01h30)

**Tipo:** ( rodada ) · **Voto:** ( A )

**Justificativa:**

O `default -> null` da `ContaFactory` deixava o erro estourar longe da causa (`Entity must not be null` no `save()`). Fail-fast, com exceção clara na própria factory ("Tipo de conta inválido: UNIVERSITARIA"), faz o erro nascer e morrer no mesmo lugar (Aulas 11 e 14), vale para qualquer chamador e não deixa validação espalhada. O trade-off é aceitar o custo de reputação (🔥 −1), porque a Universitária continua indisponível, mas sem dívida nova e sem contas criadas com o tipo errado.

**Placar do grupo após esta decisão:** 🔥 8 (-1) · 💰 R$ 60 mil (0) · 🧹 1 (0)

---

## ⚡ O cadastro com metade dos campos (02h20) · Relâmpago C

**Tipo:** ( relâmpago ) · **Voto:** ( B )

**Justificativa:**

O padrão Builder (criacional, Aula 14) com valores padrão concentra defaults e validação no `build()`, deixando claro o que é obrigatório e evitando o "construtor telescópico", que cresce a cada combinação de campos. Aceitar `null` (C) espalharia `NullPointerException` pelo sistema. O trade-off é escrever uma classe Builder agora, em vez de um construtor extra rápido, em troca de manutenção sem dívida (🧹 0).

**Placar do grupo após esta decisão:** 🔥 8 (0) · 💰 R$ 60 mil (0) · 🧹 1 (0)

---

## ⚡ O teste verde que não testa nada (03h20) · Relâmpago D

**Tipo:** ( relâmpago ) · **Voto:** ( A )

**Justificativa:**

O teste do depósito roda e fica verde, mas não tem nenhum `assert`, então não verifica nada e dá falsa segurança (Aula 15). Adicionar o `assertEquals` com o saldo esperado transforma o teste em uma verificação real do comportamento. O trade-off são ~10 minutos de trabalho, contra o risco de o depósito quebrar sem ninguém perceber (B) ou perder cobertura apagando o teste (C).

**Placar do grupo após esta decisão:** 🔥 8 (0) · 💰 R$ 60 mil (0) · 🧹 0 (-1)

---

## O teste vermelho às 03h45 (a janela fecha às 04h)

**Tipo:** ( rodada ) · **Voto:** ( B )

**Justificativa:**

O teste `tedDeveCobrarTaxaDe5Reais` falha porque o serviço usa um número mágico `50.00`, e o contrato fixa a TED em R$ 5,00: o teste está certo e o código errado. Desabilitar, ignorar o vermelho ou ajustar o teste para 50.0 (A, C, D) subiria o bug no ar, cobrando 10x a taxa na Black Friday e quebrando a regra "verde é pré-requisito pra subir". O trade-off é perder a janela de deploy (🔥 −1, 💰 +30 mil), aceito para preservar o contrato e a confiança na suíte (🧹 −1).

**Placar do grupo após esta decisão:** 🔥 7 (-1) · 💰 R$ 90 mil (+30) · 🧹 0 (-1, sem ficar negativo)

