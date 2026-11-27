# Requisitos - API Raízes do Nordeste

Aluno: Gabriel Antunes de Oliveira
RU: 4622636
Trilha: Back-end
Etapa 1 - Planejamento

## 1. Entendendo o problema

A Raízes do Nordeste começou como uma lanchonete de família em Recife (Dona Francisca e os dois filhos) e hoje é uma rede com várias unidades em capitais e no interior. Cresceu rápido e os sistemas não acompanharam.

O que eu entendi lendo o estudo de caso:

- O cliente pede por vários canais: app, totem, balcão, pick-up (retirada) e web. Mas a experiência tem que ser a mesma: ver o cardápio da unidade, escolher, pedir, acompanhar e receber certo.
- Cada unidade é um pouco diferente. Tem loja com cozinha completa e loja reduzida, produto que só sai no São João, receita que muda por região. Ou seja, o cardápio e o estoque são **por unidade**.
- Cada unidade controla seu estoque, sua equipe e suas regras. A matriz quer ver tudo junto (vendas por unidade/região, produtos mais vendidos, relatórios) e quer auditoria de coisas sensíveis tipo cancelamento e desconto.
- Querem um programa de fidelidade com pontos, mas com LGPD levada a sério (consentimento, cuidado com dados, auditoria de acesso).
- Pagamento não é processado pelo sistema deles. O sistema só pede o pagamento pra um serviço externo, recebe se foi aprovado ou não, registra e atualiza o pedido.
- Em horário de pico o sistema não pode cair.

### Usuários (atores)

| Ator | O que faz |
|---|---|
| Cliente | Vê cardápio, faz pedido (app, web, totem), acompanha status, junta pontos |
| Atendente (balcão) | Faz pedido pro cliente que está na loja |
| Cozinha | Vê os pedidos e muda o status (preparando, pronto) |
| Gerente | Cuida do estoque e dos produtos da unidade dele, cancela pedido |
| Administrador (matriz) | Cadastra unidades, produtos e usuários, vê relatórios |
| Gateway de pagamento | Sistema externo que aprova ou recusa o pagamento (aqui vai ser um mock) |

### Problemas principais da empresa

1. Pedido vindo de vários canais sem um lugar central.
2. Estoque diferente em cada loja, risco de vender o que não tem.
3. Matriz sem visão consolidada e sem rastro do que foi cancelado/alterado.
4. Fidelidade precisa de dados pessoais, então tem que respeitar a LGPD.
5. Pagamento depende de terceiro, que pode falhar ou recusar.
6. Sistema precisa aguentar pico sem cair.

## 2. Requisitos Funcionais (RF)

**Usuários e acesso**
- RF01 - Cadastrar usuário com nome, e-mail e senha.
- RF02 - Fazer login com e-mail e senha e receber um token.
- RF03 - Cada usuário tem um perfil: CLIENTE, ATENDENTE, COZINHA, GERENTE ou ADMIN. O que cada um pode fazer depende do perfil.

**Unidades**
- RF04 - Cadastrar e listar as unidades da rede (nome, cidade, estado, tipo de cozinha).
- RF05 - Ativar/desativar uma unidade.

**Produtos e cardápio**
- RF06 - Cadastrar produtos (nome, descrição, preço, categoria).
- RF07 - Consultar o cardápio de uma unidade, mostrando só o que está ativo e com estoque naquela loja.
- RF08 - Produto sazonal (ex.: junino) aparecer só no período certo. São duas formas diferentes de fazer isso:
  - **ativação manual:** o gerente liga/desliga o produto pelo campo `ativo`. É o que pretendo fazer.
  - **controle por datas:** o produto teria data de início e fim e o sistema mostraria/esconderia sozinho. Isso fica só planejado.

**Estoque**
- RF09 - Registrar entrada de estoque de um produto em uma unidade.
- RF10 - Registrar saída (manual ou automática quando o pedido é pago).
- RF11 - Consultar o saldo de estoque por unidade.
- RF12 - Não deixar criar pedido se não tiver estoque suficiente.

**Pedidos**
- RF13 - Criar pedido com unidade, itens, quantidades e o canal de origem (`canalPedido`: APP, TOTEM, BALCAO, PICKUP, WEB). O canal é obrigatório: se não vier ou vier um valor fora da lista, a API retorna 422 com o erro padrão. Ele fica salvo no pedido pra dar pra saber de onde veio cada venda.
- RF14 - Calcular o total do pedido pelo preço do produto no banco (não confiar no valor que vem do cliente).
- RF15 - Atualizar o status do pedido seguindo a ordem: AGUARDANDO_PAGAMENTO -> PAGO -> EM_PREPARO -> PRONTO -> ENTREGUE. Também pode ir pra CANCELADO.
- RF16 - Cancelar pedido (cliente só cancela antes de pagar; gerente pode cancelar depois).
- RF17 - Listar pedidos com filtro por canal, status e unidade (ex.: `?canalPedido=TOTEM`), com paginação.
- RF18 - Consultar um pedido pelo id para acompanhar o status.

**Pagamento (externo / mock)**
- RF19 - Solicitar o pagamento de um pedido para um serviço externo simulado.
- RF20 - Receber a resposta (aprovado ou recusado), registrar o pagamento e atualizar o status do pedido.
- RF21 - Não cobrar o mesmo pedido duas vezes (**idempotência**). A requisição de pagamento manda uma chave única no header `Idempotency-Key`. Se a mesma chave chegar de novo (ex.: o totem travou e o cliente apertou "pagar" outra vez, ou a internet caiu e o app reenviou), a API devolve o mesmo resultado do primeiro pagamento em vez de criar outro. Se o pedido já estiver PAGO e vier uma chave nova, a API recusa com 409.

**Fidelidade**
- RF22 - Cliente aceita ou não participar do programa (consentimento registrado com data).
- RF23 - Ganhar pontos quando o pedido é pago (ex.: 1 ponto a cada R$ 1).
- RF24 - Consultar saldo e histórico de pontos.
- RF25 - Resgatar pontos como desconto simples em um pedido.

**Promoções / campanhas**
- RF26 - Cadastrar promoções (percentual de desconto, período, unidade ou rede toda).
- RF27 - Campanhas segmentadas por perfil/frequência do cliente (só com consentimento).

**Matriz / gestão**
- RF28 - Relatório de vendas por unidade e por região.
- RF29 - Relatório de produtos mais vendidos.
- RF30 - Registrar log de auditoria de ações sensíveis (criar/cancelar pedido, mudar status, ajuste de estoque, desconto).

## 3. Requisitos Não Funcionais (RNF)

- RNF01 - **Segurança:** senha salva com hash (bcrypt), nunca em texto puro.
- RNF02 - **Autenticação** por token JWT com tempo de expiração.
- RNF03 - **Autorização** por perfil em cada endpoint (ex.: só GERENTE/ADMIN mexem no estoque).
- RNF04 - **LGPD:** coletar só o necessário (nome, e-mail, senha; CPF não é obrigatório), deixar claro pra que serve, registrar consentimento e não devolver senha/hash nas respostas.
- RNF05 - **LGPD:** permitir anonimizar os dados do cliente quando ele pedir pra sair, mantendo os pedidos para histórico/fiscal.
- RNF06 - **Auditoria:** guardar quem fez, o que fez e quando nas ações sensíveis.
- RNF07 - **Padrão de erro:** toda resposta de erro no mesmo formato JSON (error, message, details, timestamp, path).
- RNF08 - **Status HTTP coerentes:** 200, 201, 204, 400/422, 401, 403, 404, 409.
- RNF09 - **Documentação:** API documentada no Swagger (o FastAPI já gera em `/docs`).
- RNF10 - **Tolerância a falha no pagamento:** se o serviço de pagamento falhar ou demorar, o pedido não se perde, fica AGUARDANDO_PAGAMENTO e dá pra tentar de novo. Por causa da idempotência (RF21), tentar de novo não gera cobrança dupla.
- RNF11 - **Desempenho:** listagens com paginação para não trazer tudo de uma vez no horário de pico.
- RNF12 - **Disponibilidade e escalabilidade:** API sem estado (stateless, por causa do JWT), o que permite subir mais de uma instância atrás de um balanceador no futuro.
- RNF13 - **Organização em camadas:** separar rotas (API), serviços (regras), modelos e acesso ao banco.
- RNF14 - **Reprodutível:** README explicando como instalar, rodar e testar, com `.env.example`.

## 4. Prioridade e situação de cada requisito

Separei os requisitos em três grupos de prioridade:

- **1 - Núcleo:** o mínimo pro Fluxo A (pedido -> pagamento mock -> status) funcionar de ponta a ponta. É obrigatório e vem primeiro.
- **2 - Complementar:** só começo depois que o núcleo estiver funcionando e testado no Postman.
- **3 - Planejado:** fica só documentado (explico como faria), sem código.

A coluna **Status atual** mostra o que já foi feito de verdade. Por enquanto está tudo "Não iniciado", porque ainda estou no planejamento. Vou atualizando a cada etapa; só vou marcar "Concluído" quando estiver no código e testado.

| Requisito | Prioridade | Status atual | Observação |
|---|---|---|---|
| RF01, RF02, RF03 | 1 - Núcleo | Não iniciado | cadastro, login JWT e perfis |
| RF04 (listar) | 1 - Núcleo | Não iniciado | `GET /unidades`; as unidades entram no banco pelo seed |
| RF04 (cadastrar pela API) | 3 - Planejado | - | **limitação:** não vai ter `POST /unidades`; nova unidade só pelo seed |
| RF06, RF07 | 1 - Núcleo | Não iniciado | produto e cardápio por unidade |
| RF09, RF11, RF12 | 1 - Núcleo | Não iniciado | entrada, saldo e bloqueio sem estoque |
| RF10 | 1 - Núcleo | Não iniciado | baixa automática do estoque quando o pedido é pago |
| RF13 | 1 - Núcleo | Não iniciado | `canalPedido` obrigatório |
| RF14, RF15, RF17, RF18 | 1 - Núcleo | Não iniciado | total, status, listagem com filtro, consulta |
| RF19, RF20 | 1 - Núcleo | Não iniciado | pagamento mock aprovado e recusado |
| RF21 | 1 - Núcleo | Não iniciado | idempotência com `Idempotency-Key` |
| RF30 | 1 - Núcleo | Não iniciado | log simples de criação de pedido e mudança de status |
| RNF01 a RNF04 | 1 - Núcleo | Não iniciado | hash, JWT, perfis, LGPD mínimo |
| RNF06 a RNF11 | 1 - Núcleo | Não iniciado | auditoria, erro padrão, status HTTP, Swagger, falha no pagamento, paginação |
| RNF13, RNF14 | 1 - Núcleo | Não iniciado | camadas e README |
| RF16 | 2 - Complementar | Não iniciado | regras de quem pode cancelar e quando |
| RF22, RF23, RF24 | 2 - Complementar | Não iniciado | consentimento, pontos e saldo |
| RF08 | 2 - Complementar | Não iniciado | só ativação manual (campo `ativo`); controle por datas é planejado |
| RF05 | 3 - Planejado | - | desativar unidade |
| RF08 (datas) | 3 - Planejado | - | produto aparecer/sumir sozinho pelo período |
| RF25 | 3 - Planejado | - | resgate de pontos |
| RF26, RF27 | 3 - Planejado | - | promoções e campanhas |
| RF28, RF29 | 3 - Planejado | - | relatórios da matriz |
| RNF05 | 3 - Planejado | - | anonimização (estratégia descrita na parte de LGPD) |
| RNF12 | 3 - Planejado | - | alta disponibilidade real (sem servidor/deploy) |

Sei que deixar itens como planejado pode não dar a nota cheia. Preferi fechar bem o fluxo principal antes de qualquer outra coisa, que é o que o roteiro também recomenda. Se sobrar tempo, o próximo a sair do "planejado" seria o resgate de pontos (RF25).
