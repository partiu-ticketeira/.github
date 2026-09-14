```mermaid
flowchart LR
    CLIENTES[Aplicativo Mobile e Plataforma Web]
    BORDA[Balanceador de Carga / API Gateway]
    API[API REST Distribuída]
    AUTH[OAuth 2.0 / JWT]
    FILA[[RabbitMQ ou Kafka]]
    WORKERS[Workers de Compra]
    REDIS[(Redis Cluster)]
    BANCO[(Banco Relacional)]
    PAGAMENTOS[Serviço de Pagamentos]
    GATEWAY[Gateway Externo]
    OBS[Observabilidade e Alertas]

    CLIENTES --> BORDA
    BORDA --> API
    API <--> AUTH
    API -->|Solicita admissão por lote| REDIS
    REDIS -->|Aceita ou rejeita rapidamente| API
    API -->|Somente pedidos admitidos| FILA
    FILA --> WORKERS
    WORKERS <--> BANCO
    WORKERS -->|Libera permissão| REDIS
    WORKERS --> PAGAMENTOS
    PAGAMENTOS <--> GATEWAY
    GATEWAY -->|Webhook assinado| API
    API --> BANCO
    API -.-> OBS
    WORKERS -.-> OBS
    PAGAMENTOS -.-> OBS
```

## Arquitetura macro

O diagrama macro mostra os principais limites do sistema e o caminho das dependências. Os clientes nunca acessam diretamente o banco, o Redis ou o gateway. Antes de publicar uma compra, a API consulta o Redis como um **admission controller** por lote: o Redis aceita apenas a quantidade configurada de pedidos em processamento e rejeita o excedente imediatamente. Só as requisições admitidas entram na fila e chegam aos workers, evitando formar uma fila de trabalho maior do que a capacidade segura do banco. O banco continua sendo a autoridade final do estoque; o Redis apenas controla pressão e não representa saldo de ingressos.

## Fluxo distribuído de compra

```mermaid
flowchart TD
    %% Forçando a ordem dos subgráficos com links invisíveis
    View ~~~ Controller
    Controller ~~~ Model
    Controller ~~~ External

    subgraph View ["View (Interface Cliente)"]
        APP[Aplicativo Mobile]
        WEB[Plataforma Web]
    end

    subgraph Controller ["Controller (Lógica e Orquestração)"]
        API[API REST Distribuída]
        AUTH[Auth Provider: OAuth 2.0 / JWT]
        FILA[[Fila de Mensageria: RabbitMQ/Kafka]]
        WORKER[Serviços de Processamento de Pedidos]
    end

    subgraph External ["Sistemas Externos"]
        PAYMENT[Gateway de Pagamento]
        WEBHOOK[Webhook do Gateway]
    end

    subgraph Model ["Model (Camada de Dados)"]
        REDIS[("Redis (Admission Control / Fail Fast)")]
        DB[("Banco de Dados Relacional\n(Atualização Atômica)")]
    end

    %% Fluxo de Acesso e Auth
    APP -->|1. Login/Requisições| API
    WEB -->|1. Login/Requisições| API
    API <-->|2. Valida Assinatura JWT| AUTH

    %% Fluxo de Compra e Controle de Pressão
    API -->|3. Solicita admissão por lote| REDIS
    REDIS -->|4. Permissão ou FAIL FAST| API
    API -->|5. Publica somente pedidos admitidos| FILA
    FILA -->|6. Consome pedidos admitidos| WORKER
    
    WORKER -->|7. UPDATE qtd = qtd - 1\nWHERE qtd > 0| DB
    DB -->|8. Confirmação ACID| WORKER
    
    WORKER -->|9. Cria cobrança com chave idempotente| PAYMENT
    PAYMENT -->|10. Checkout e autorização| WEBHOOK
    
    %% Setas mais longas para o retorno ajudam o motor a não deformar a caixa
    PAYMENT -.->|11. Notifica mudança de status| WEBHOOK
    WEBHOOK --->|12. Valida assinatura, valor e reserva| API
    
    API -->|13. Atualiza reserva de forma idempotente| DB
    API -->|14. Libera QR Code se pago| WORKER
    WORKER -->|15. Libera permissão no Redis| REDIS
    
    %% Feedback ao cliente
    WORKER -.->|Gera Reserva / Status| API
    API -.->|Retorna Status| APP
    API -.->|Retorna Status| WEB
```

### Responsabilidade de cada etapa

1. **Cliente -> API:** o aplicativo ou a plataforma web envia evento, lote, quantidade e chave idempotente. A API valida o JWT, os parâmetros, o preço vigente e a permissão do comprador.
2. **API -> Auth Provider:** a API valida assinatura, expiração e escopos do JWT. Token inválido deve ser rejeitado antes de consultar o Redis ou publicar qualquer pedido.
3. **API -> Redis:** a API solicita uma permissão de admissão para o `tipo_ingresso_id`. O Redis mantém um contador ou semáforo com limite de requisições em processamento e TTL para evitar permissões presas.
4. **Redis -> API:** se houver capacidade, o Redis confirma a permissão; se o limite estiver atingido ou o Redis estiver indisponível, a API falha rapidamente com `429` ou `503`, sem publicar na fila e sem acessar o banco.
5. **API -> Fila:** somente depois da admissão confirmada a API registra a intenção pela chave idempotente e publica uma mensagem durável. Se a publicação falhar, a permissão deve ser devolvida ao Redis.
6. **Fila -> Worker:** o worker consome apenas pedidos que já passaram pelo controle de pressão. A mensagem só deve ser confirmada depois da reserva e da criação ou reconhecimento da cobrança.
7. **Worker -> Banco:** o worker executa a atualização condicional `qtd_disponivel = qtd_disponivel - 1 WHERE qtd_disponivel > 0` dentro de uma transação. O Redis não informa nem substitui o saldo real.
8. **Banco -> Worker:** uma linha atualizada confirma a baixa; nenhuma linha atualizada significa lote esgotado. O worker cria a reserva `RESERVADO` com prazo de expiração na mesma transação ou em uma transação claramente compensável.
9. **Worker -> Gateway:** o serviço de pagamentos cria a cobrança usando `idempotency_key`, valor calculado no servidor e referência da reserva. Dados completos do cartão permanecem no provedor.
10. **Gateway -> Checkout:** o gateway processa PIX, débito, crédito ou outro meio suportado. Criar a cobrança significa apenas `PENDENTE`, nunca pagamento aprovado.
11. **Gateway -> Webhook:** o gateway envia um evento assinado com `event_id`, status, valor e identificador externo. Reenvios e eventos fora de ordem devem ser esperados.
12. **Webhook -> API:** a API valida assinatura, gateway, `event_id`, valor, `gateway_payment_id` e vínculo com o pagamento. O evento é persistido em `WebhooksPagamento` para idempotência e reconciliação.
13. **API -> Banco:** em uma transação, a API altera pagamento e ingresso apenas se o estado atual permitir a transição. Evento repetido não cria ingresso nem devolve estoque novamente.
14. **API -> Worker:** pagamento aprovado libera o QR Code; falha, cancelamento ou expiração marca o ingresso como `CANCELADO` e devolve a unidade ao lote uma única vez.
15. **Worker -> Redis:** após finalizar o processamento, inclusive em erro, o worker libera a permissão em um bloco `finally`. O TTL permanece como proteção contra queda do processo.
16. **API -> Cliente:** a API retorna o estado por consulta, polling ou notificação. O cliente só exibe o QR Code quando o estado persistido for `PAGO`.

O Redis atua como **fail fast**, não como lock de estoque: ele limita a pressão sobre o banco antes da fila e do worker, mas não garante disponibilidade nem consistência da cota. A proteção definitiva contra *double selling* é a atualização condicional e transacional do banco. O limite do Redis deve ser configurado abaixo da capacidade sustentada do banco, monitorado e ajustado com métricas de latência, rejeições e ocupação.

## Modelo de banco de dados

O diagrama abaixo representa as entidades implementadas no backend e seus relacionamentos. As chaves estrangeiras estão indicadas nos campos `FK`; a cardinalidade considera o comportamento definido pelos relacionamentos SQLAlchemy.

```mermaid
erDiagram
    PESSOAS {
        int id PK
        string nome
        string email UK
        string cpf UK
        string celular
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    EVENTOS {
        int id PK
        string nome
        text descricao
        datetime data
        string local
        string status
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    ORGANIZADORES {
        int id PK
        int pessoa_id FK
        int evento_id FK
        string role
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    TIPOS_INGRESSO {
        int id PK
        int evento_id FK
        string nome
        int lote
        decimal preco
        int qtd_total
        int qtd_disponivel
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    PAGAMENTOS {
        int id PK
        int pessoa_id FK
        string metodo_pagamento
        int parcelas
        decimal valor_base
        decimal valor_taxa
        decimal valor_total
        int qtd_ingressos
        string status
        string gateway
        string gateway_payment_id UK
        string idempotency_key UK
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    WEBHOOKS_PAGAMENTO {
        int id PK
        string gateway
        string event_id UK
        string event_type
        int pagamento_id FK
        json payload
        string status
        datetime processed_at
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    INGRESSOS {
        int id PK
        int tipo_ingresso_id FK
        int pagamento_id FK
        int pessoa_id FK
        string hash_qr
        string status
        datetime expiracao_reserva
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    TRANSFERENCIAS {
        int id PK
        int ingresso_id FK
        int pessoa_origem_id FK
        int pessoa_destino_id FK
        string status
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }

    PESSOAS ||--o{ ORGANIZADORES : "gerencia eventos"
    EVENTOS ||--o{ ORGANIZADORES : "possui organizadores"
    EVENTOS ||--o{ TIPOS_INGRESSO : "possui lotes"
    TIPOS_INGRESSO ||--o{ INGRESSOS : "disponibiliza"
    PESSOAS ||--o{ INGRESSOS : "compra"
    PESSOAS ||--o{ PAGAMENTOS : "realiza"
    PAGAMENTOS o|--o{ INGRESSOS : "quita"
    PAGAMENTOS o|--o{ WEBHOOKS_PAGAMENTO : "recebe eventos"
    INGRESSOS ||--o{ TRANSFERENCIAS : "pode ser transferido"
    PESSOAS ||--o{ TRANSFERENCIAS : "origina"
    PESSOAS ||--o{ TRANSFERENCIAS : "recebe"
```

### Regras representadas

- Um evento pode ter vários organizadores e vários tipos de ingresso; cada registro de `Organizador` vincula uma pessoa a um evento e define sua função.
- Cada ingresso pertence a um tipo de ingresso e a uma pessoa. O `pagamento_id` é opcional durante a reserva e fica associado quando a cobrança é criada ou confirmada.
- Um pagamento pertence a uma pessoa e pode quitar vários ingressos da mesma compra. Seus estados previstos são `PENDENTE`, `APROVADO`, `RECUSADO` e `ESTORNADO`.
- O ingresso controla `RESERVADO`, `PAGO`, `CANCELADO` e `UTILIZADO`, além do prazo de expiração e do QR Code.
- Uma transferência registra o ingresso, a pessoa de origem, a pessoa de destino e seu estado (`PENDENTE`, `CONCLUIDA` ou `CANCELADA`).

Os campos `gateway`, `gateway_payment_id` e `idempotency_key` em `Pagamentos`, junto com a tabela `WebhooksPagamento`, suportam a integração com o gateway externo. O `event_id` único permite registrar e ignorar reenvios do mesmo webhook, enquanto `processed_at` permite acompanhar a conciliação e o reprocessamento seguro.
