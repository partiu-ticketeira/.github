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
    API --> FILA
    FILA --> WORKERS
    WORKERS <--> REDIS
    WORKERS <--> BANCO
    WORKERS --> PAGAMENTOS
    PAGAMENTOS <--> GATEWAY
    GATEWAY -->|Webhook assinado| API
    API --> BANCO
    API -.-> OBS
    WORKERS -.-> OBS
    PAGAMENTOS -.-> OBS
```

## Arquitetura macro

O diagrama macro mostra os principais limites do sistema e o caminho das dependências. Os clientes nunca acessam diretamente o banco, o Redis ou o gateway. O balanceador distribui as requisições entre instâncias da API; a fila absorve picos; os workers processam a reserva; e o serviço de pagamentos concentra a integração com o provedor externo. A observabilidade acompanha API, workers e pagamentos para permitir diagnóstico de filas, falhas e divergências de conciliação.

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
        REDIS[("Redis (Lock Distribuído)")]
        DB[("Banco de Dados Relacional\n(Atualização Atômica)")]
    end

    %% Fluxo de Acesso e Auth
    APP -->|1. Login/Requisições| API
    WEB -->|1. Login/Requisições| API
    API <-->|2. Valida Assinatura JWT| AUTH

    %% Fluxo de Compra e Concorrência
    API -->|3. Envia pedido de compra| FILA
    FILA -->|4. Consome pedidos FIFO| WORKER
    
    WORKER -->|5. Solicita Lock para o Lote| REDIS
    REDIS -->|6. Lock Concedido| WORKER
    
    WORKER -->|7. UPDATE Ingresso SET qtd = qtd - 1\nWHERE qtd > 0| DB
    DB -->|8. Confirmação ACID| WORKER
    
    WORKER -->|9. Cria cobrança com chave idempotente| PAYMENT
    PAYMENT -->|10. Checkout e autorização| WEBHOOK
    
    %% Setas mais longas para o retorno ajudam o motor a não deformar a caixa
    PAYMENT -.->|11. Notifica mudança de status| WEBHOOK
    WEBHOOK --->|12. Valida assinatura, valor e reserva| API
    
    API -->|13. Atualiza reserva de forma idempotente| DB
    API -->|14. Libera QR Code se pago| WORKER
    WORKER -->|15. Libera Lock| REDIS
    
    %% Feedback ao cliente
    WORKER -.->|Gera Reserva / Status| API
    API -.->|Retorna Status| APP
    API -.->|Retorna Status| WEB
```

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
