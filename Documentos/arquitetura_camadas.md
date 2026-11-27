# Arquitetura por camadas - API Raízes do Nordeste

Aluno: Gabriel Antunes de Oliveira
RU: 4622636

Separei o projeto em 4 camadas, cada uma com uma responsabilidade. A ideia é que uma camada só "conversa" com a de baixo, e que as regras do negócio não dependam de banco nem de FastAPI.

```
API  ->  Application  ->  Domain
              |
              v
        Infrastructure
```

## API
Responsável pelas rotas/endpoints, autenticação (JWT), autorização por perfil, validação do request/response e Swagger.

Exemplos:
- /auth
- /unidades
- /produtos
- /estoque
- /pedidos
- /pagamentos

Aqui não tem regra de negócio: a rota recebe o JSON, valida o formato, confere o token/perfil e chama a camada Application.

## Application
Responsável pelos casos de uso e pela orquestração das regras (o "passo a passo" de cada ação).

Exemplos:
- criar pedido (confere unidade, produtos, estoque e calcula o total)
- processar pagamento mock (idempotência, baixa de estoque, chamada do gateway, rollback)
- atualizar status do pedido
- registrar movimentação de estoque
- registrar log de auditoria das ações sensíveis

## Domain
Responsável pelas entidades, enums e regras do negócio.

Exemplos:
- Usuario
- Unidade
- Produto
- Estoque
- Pedido / ItemPedido
- Pagamento
- LogAuditoria
- CanalPedido, StatusPedido, StatusPagamento, Perfil (enums)

Exemplo de regra que fica aqui: a ordem dos status do pedido (AGUARDANDO_PAGAMENTO -> PAGO -> EM_PREPARO -> PRONTO -> ENTREGUE).

## Infrastructure
Responsável pelo acesso ao banco, persistência e integrações externas.

Exemplos:
- SQLite
- SQLAlchemy (models e sessão do banco)
- repositórios
- seed (dados iniciais: unidades, produtos, usuários de teste)
- gateway de pagamento mock (implementa a interface GatewayPagamento)
- gravação dos logs/auditoria no banco

## Como pretendo organizar as pastas em `Codigo/`

```
Codigo/
└── app/
    ├── api/             rotas (auth, pedidos, pagamentos...)
    ├── application/     serviços / casos de uso
    ├── domain/          entidades e enums
    ├── infrastructure/  banco, repositórios, gateway mock, seed
    └── main.py          cria o FastAPI e registra as rotas
```

## Exemplo do fluxo crítico passando pelas camadas

1. **API:** `POST /pagamentos` recebe o pedido, o `Idempotency-Key` e a `simulacao`, e confere o token.
2. **Application:** `PagamentoService` verifica a chave, baixa o estoque e chama o gateway.
3. **Domain:** `Pedido.mudarStatus()` valida se pode ir pra PAGO.
4. **Infrastructure:** o mock responde aprovado/recusado e o repositório salva o pagamento, o pedido e o log.

Essa separação já aparece no diagrama de classes (`PagamentoService` = application, `GatewayPagamentoMock` = infrastructure).
