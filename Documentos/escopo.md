# Escopo - o que vou implementar e o que fica como proposta

Não dá pra fazer o sistema inteiro da rede num projeto da faculdade, então separei o que vou programar de verdade e o que vai ficar só documentado.

O roteiro pede pra escolher um fluxo e fechar ele de ponta a ponta. Escolhi o **Fluxo A: Pedido -> Pagamento mock -> Atualização de status**, porque é o coração do negócio (é por onde entra o dinheiro) e ele acaba puxando junto estoque e canal do pedido.

Como o escopo ficou grande pra mim, dividi em duas partes. Primeiro faço o núcleo (Fluxo A) funcionar e testar inteiro. Só depois mexo no complementar.

Obs.: tudo aqui é **planejamento**. Nada foi programado ainda. O que estiver concluído de verdade vai sendo marcado na coluna "Status atual" do [requisitos.md](requisitos.md).

## Parte 1 - Núcleo (Fluxo A, obrigatório)

| Parte | O que vai ter | Requisitos |
|---|---|---|
| Autenticação | cadastro, login com JWT, senha com bcrypt, perfis | RF01, RF02, RF03, RNF01-03 |
| Unidades | só listar. As unidades entram no banco pelo seed (dados já inseridos ao criar o banco) | RF04 (parte) |
| Produtos / cardápio | cadastrar produto e ver cardápio por unidade | RF06, RF07 |
| Estoque | entrada, saldo por unidade, bloqueio sem estoque e baixa quando o pedido é pago | RF09-RF12 |
| Pedidos | criar com `canalPedido` obrigatório, calcular total, mudar status, listar com filtro, consultar | RF13-RF15, RF17, RF18 |
| Pagamento mock | aprova ou recusa, registra o pagamento, muda o status, com idempotência | RF19-RF21, RNF10 |
| Auditoria básica | log de criação de pedido e mudança de status | RF30, RNF06 |
| Erros e docs | erro padrão em JSON, paginação e Swagger | RNF07-RNF09, RNF11 |

## Parte 2 - Complementar (só depois do núcleo pronto)

| Parte | O que vai ter | Requisitos |
|---|---|---|
| Cancelamento | regras de quem cancela e quando (cliente antes de pagar, gerente depois) | RF16 |
| Fidelidade simples | consentimento, ganhar pontos no pagamento, ver saldo | RF22-RF24 |
| Produto sazonal | só ativação manual pelo campo `ativo` (o gerente liga/desliga). Não é controle por datas | RF08 |

Tecnologias: Python + FastAPI, SQLite com SQLAlchemy, JWT, Postman para os testes e GitHub para versionar.

### Como o pagamento mock vai decidir

O resultado não pode ser aleatório, senão o professor roda o teste e cada vez dá uma coisa. Então quem decide é um campo `simulacao` enviado no `POST /pagamentos`:

| `simulacao` | O que o mock faz | Status do pedido | HTTP |
|---|---|---|---|
| `APROVAR` (padrão se não mandar) | aprova | PAGO | 201 |
| `RECUSAR` | recusa (ex.: "saldo insuficiente") | continua AGUARDANDO_PAGAMENTO, pagamento salvo como RECUSADO | 201 com status RECUSADO no corpo |
| `FALHAR` | finge que o serviço externo caiu | continua AGUARDANDO_PAGAMENTO | 502 com erro padrão |

Assim os três casos (aprovado, recusado e falha) ficam reproduzíveis na coleção do Postman, sempre com o mesmo resultado. Num sistema real esse campo não existiria, quem decide é o gateway.

### Estoque com dois pedidos ao mesmo tempo

O problema: a unidade tem 1 bolo de macaxeira, e o totem e o app fazem pedido quase no mesmo segundo. Os dois veem "tem 1" e os dois passam. Aí o estoque fica negativo.

Como pretendo evitar:
1. Na criação do pedido o sistema só **confere** se tem estoque (pra já avisar o cliente com 409 se não tiver). Isso ainda não garante nada.
2. A baixa de verdade acontece no pagamento, dentro de **uma transação só** no banco, e **antes** de chamar o mock.
3. A baixa é feita com um UPDATE condicional, que confere e desconta no mesmo comando:
   `UPDATE estoque SET quantidade = quantidade - 2 WHERE unidade_id = 1 AND produto_id = 5 AND quantidade >= 2`
   Se nenhuma linha for alterada, é porque outro pedido levou antes. Aí a transação é desfeita e a API retorna 409 ESTOQUE_INSUFICIENTE, sem cobrar o cliente.
4. Se o mock recusar ou falhar, a transação é desfeita (rollback) e o estoque volta.
5. No banco também vai ter `CHECK (quantidade >= 0)` como última proteção.

O SQLite trava o banco inteiro na hora de escrever, então na prática dois pagamentos não gravam juntos. Mas o UPDATE condicional funciona igual se um dia trocar pra PostgreSQL, que é o que importa.

Limitação: entre criar o pedido e pagar, o estoque não fica "reservado". Reserva com prazo de validade fica como melhoria planejada.

### Idempotência no pagamento

Idempotência é quando repetir a mesma requisição não muda o resultado. No pagamento isso é importante porque o cliente pode clicar duas vezes ou o app pode reenviar se a internet cair, e ninguém quer ser cobrado duas vezes.

Como vou fazer:
1. Quem chama `POST /pagamentos` manda um header `Idempotency-Key` com um valor único (ex.: um UUID gerado pelo app/totem).
2. Essa chave fica salva na tabela de pagamentos com restrição UNIQUE.
3. Se chegar uma requisição com uma chave que já existe, a API não chama o mock de novo, só devolve o pagamento que já foi registrado.
4. Se o pedido já está PAGO e chega uma chave diferente, retorna 409 (pedido já pago).

### canalPedido

O campo `canalPedido` (APP, TOTEM, BALCAO, PICKUP, WEB) vai ser um ENUM obrigatório no pedido, no banco, no DER e no Swagger. Também vai dar pra filtrar a listagem por ele (`GET /pedidos?canalPedido=TOTEM`).

## Fica como proposta (só documentado)

| Item | Por que não vou fazer agora |
|---|---|
| Promoções e campanhas segmentadas (RF26, RF27) | precisa de regras de perfil/segmentação mais complexas. Vou explicar como aplicaria o desconto no pedido |
| Resgate de pontos (RF25) | preferi focar em ganhar e consultar pontos; o resgate como desconto no pedido fica documentado |
| Produtos sazonais por período (RF08) | o controle automático por datas (início/fim) fica como melhoria. O que vai ter é só a ativação manual (parte 2) |
| Relatórios da matriz (RF28, RF29) | o foco é o fluxo do pedido; os dados já ficam salvos e dariam pra gerar depois |
| Cadastrar unidade pela API (RF04) | **limitação:** não vai ter `POST /unidades`. As unidades de exemplo vêm do seed. Pra abrir uma unidade nova teria que mexer no seed/banco |
| Desativar unidade (RF05) | simples, mas não é prioridade |
| Anonimização automática (RNF05) | vou descrever a estratégia na parte de LGPD |
| Alta disponibilidade real (RNF12) | sem deploy/servidor de verdade; só explico como escalaria (stateless, banco maior tipo PostgreSQL, balanceador) |
| Reserva de estoque ao criar o pedido | o estoque só é descontado no pagamento; reservar com prazo de validade deixaria tudo mais complexo |
| Refresh token / logout | o token simples com expiração já atende o mínimo |

A prioridade e a situação de cada requisito estão na tabela do final do [requisitos.md](requisitos.md).

## Ordem que pretendo seguir

Núcleo:
1. Estrutura do projeto e banco
2. Login e perfis
3. Unidades, produtos e estoque
4. Pedido com canal
5. Pagamento mock (com idempotência) e status
6. Auditoria básica
7. Testes do fluxo A no Postman

Complementar (só se o núcleo estiver ok):
8. Cancelamento com regras
9. Fidelidade
10. Ativação manual de produto

Final:
11. README completo e entrega
