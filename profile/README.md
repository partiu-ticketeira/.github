## Hi there 👋

<!--

**Here are some ideas to get you started:**

🙋‍♀️ A short introduction - what is your organization all about?
🌈 Contribution guidelines - how can the community get involved?
👩‍💻 Useful resources - where can the community find your docs? Is there anything else the community should know?
🍿 Fun facts - what does your team eat for breakfast?
🧙 Remember, you can do mighty things with the power of [Markdown](https://docs.github.com/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
-->
﻿# PARTIU

O **PARTIU** é uma plataforma distribuída para venda e gerenciamento de ingressos para eventos. O sistema foi projetado para suportar muitos acessos simultâneos, evitar venda duplicada e manter a operação disponível durante a abertura de lotes concorridos.

## Visão geral

O cliente acessa a plataforma pelo aplicativo mobile ou pela web. A API REST autentica a requisição e usa o Redis como controle de admissão *fail fast*: somente uma quantidade segura de pedidos por lote é encaminhada para a fila. Trabalhadores processam os pedidos admitidos e confirmam a baixa de estoque por meio de uma atualização atômica no banco relacional.

O pagamento é realizado por um **gateway externo**. O PARTIU não precisa armazenar dados sensíveis do cartão: cria a cobrança, acompanha o status e recebe a confirmação por webhook. Somente uma confirmação válida e idempotente transforma a reserva em ingresso pago.

## Arquitetura e padrão MVC

O sistema adota uma arquitetura cliente-servidor organizada segundo o padrão MVC:

- **View:** aplicativo mobile e plataforma web. Permite que compradores consultem eventos, reservem ingressos e acompanhem pagamentos; organizadores criem eventos e gerenciem lotes.
- **Controller:** API REST distribuída, autenticação, regras de negócio, fila de pedidos e integração com o gateway de pagamento.
- **Model:** entidades, serviços de persistência, Redis para controle de pressão e banco relacional para os dados definitivos.

Essa separação permite escalar a API e os trabalhadores de forma independente, sem misturar a interface com as regras de negócio ou a persistência.

## Autenticação e segurança

A autenticação utiliza OAuth 2.0 com JWT:

1. O usuário entra com e-mail e senha ou por um provedor externo, como Google ou Facebook.
2. O serviço de autenticação emite um JWT com identidade e permissões.
3. A API valida a assinatura e a validade do token localmente, sem consultar o banco em todas as requisições.
4. As permissões são verificadas antes de operações como criar evento, editar lote ou consultar uma reserva.

O gateway recebe apenas os dados necessários para criar a cobrança. Dados completos de cartão devem permanecer no ambiente do provedor, reduzindo a responsabilidade do PARTIU sobre informações financeiras sensíveis.

## Entidades do domínio

| Entidade | Responsabilidade |
| --- | --- |
| **Pessoa** | Cadastro de compradores e organizadores, com nome, e-mail, CPF e credenciais protegidas. |
| **Organizador** | Associação entre uma pessoa e um evento, incluindo permissões de gestão. |
| **Evento** | Nome, descrição, data, local e status do evento. |
| **TipoIngresso** | Categoria ou lote, como Pista, VIP, Camarote ou Bistrô, com preço, cota total e saldo disponível. |
| **Ingresso** | Unidade associada à pessoa e ao tipo de ingresso, com status da reserva, pagamento e QR Code. |
| **Pagamento** | Identificador da cobrança no gateway, valor, status, timestamps e chave de idempotência. |
| **Transferencia** | Registro de transferência de titularidade de um ingresso, quando essa operação for permitida. |

## Fluxo detalhado de compra

1. O comprador escolhe o evento e o lote. A API valida o JWT, o preço vigente e a disponibilidade.
2. A intenção de compra recebe um identificador idempotente e é publicada na fila. O cliente recebe um status de processamento.
3. A API solicita ao Redis uma permissão limitada por lote. Se a capacidade segura estiver esgotada ou o Redis estiver indisponível, a API responde rapidamente com `429` ou `503`, sem publicar na fila nem acessar o banco.
4. Somente pedidos admitidos entram na fila. O worker executa no banco uma atualização condicional, como `quantidade_disponivel = quantidade_disponivel - 1 WHERE quantidade_disponivel > 0`.
5. Se a baixa for aceita, o sistema cria uma reserva com prazo de expiração. Se não houver saldo, a solicitação é recusada sem alterar o estoque.
6. O serviço de pagamentos cria uma cobrança no gateway e associa o identificador externo à reserva. O cliente pode ser redirecionado para checkout, PIX ou outro meio oferecido pelo provedor.
7. O gateway envia um webhook assinado quando a cobrança muda de estado. A API valida a assinatura, verifica o valor e o identificador da reserva e processa o evento de forma idempotente.
8. Em caso de pagamento aprovado, a reserva vira `pago` e o QR Code é liberado. Em caso de falha, cancelamento ou expiração, a reserva vira `cancelado` e o saldo retorna ao lote.
9. O worker libera a permissão do Redis em qualquer resultado, inclusive erro, e a API informa o estado atualizado ao cliente. Reprocessamentos não podem criar ingresso ou cobrança duplicada.

### Estados da reserva

`reservado` -> `aguardando_pagamento` -> `pago`

Uma reserva também pode seguir para `cancelado` por falha no pagamento, cancelamento explícito ou expiração do prazo. A devolução do estoque deve ser executada uma única vez, dentro de uma operação transacional ou protegida por idempotência.

## Concorrência e alta disponibilidade

O controle contra *double selling* combina três mecanismos, com responsabilidades diferentes:

- **Fila de mensageria:** Kafka absorve picos e distribui pedidos entre trabalhadores.
- **Admission control fail fast:** Redis limita a quantidade de pedidos do mesmo lote que podem avançar para a fila e o worker. Quando a capacidade é atingida, a requisição é rejeitada imediatamente, protegendo o banco contra overload.
- **Atualização atômica:** o banco só reduz a quantidade quando o saldo ainda é maior que zero, preservando a consistência ACID mesmo se houver concorrência ou repetição.

O Redis não é fonte de estoque e não substitui a restrição do banco. Ele controla pressão e aplica *fail fast*; a atualização condicional é a garantia final de consistência. A permissão deve ter TTL e ser liberada pelo worker em `finally`, inclusive quando a transação falha. A API e os trabalhadores podem ser replicados atrás de um balanceador; Redis, banco e fila devem ser executados com redundância e monitoramento.

## Casos de uso principais

- **Criar e gerenciar evento:** o organizador cadastra o evento e configura categorias, lotes, preços e cotas.
- **Reservar ingresso:** o comprador seleciona uma categoria e obtém uma reserva temporária se houver saldo.
- **Pagar:** o sistema cria a cobrança no gateway e acompanha o resultado sem expor dados de cartão ao PARTIU.
- **Confirmar pagamento:** o webhook validado altera a reserva para paga e libera o QR Code.
- **Expirar reserva:** um processo periódico identifica reservas vencidas, cancela a cobrança quando aplicável e devolve a unidade ao lote.
- **Validar acesso:** a equipe do evento consulta ou lê o QR Code e verifica se o ingresso está pago e ainda não foi utilizado.

## Diagramas do projeto

### 1. Casos de uso

![Diagrama de casos de uso do PARTIU](Casos%20de%20uso-2026-09-01-161103.png)

Este diagrama apresenta os atores e as funcionalidades externas do sistema. Ele ajuda a delimitar o que compradores, organizadores e operadores fazem na plataforma: consultar eventos, comprar ingressos, administrar lotes e validar o acesso. O gateway aparece como uma dependência externa do caso de uso de pagamento, pois a autorização financeira acontece fora do domínio do PARTIU.

### 2. Modelo de arquitetura

![Modelo de arquitetura do PARTIU](Modelo%201.png)

O primeiro modelo mostra a divisão em interface, serviços de aplicação e persistência. A leitura deve começar pelos clientes, passar pela API e chegar às entidades e serviços responsáveis por regras, autenticação e estoque. A fronteira entre a aplicação e integrações externas evidencia que autenticação e pagamentos podem evoluir sem acoplar a interface ao banco.

### 3. Diagramas Mermaid e fluxo distribuído de compra

![Fluxo distribuído de compra](modelo%202.png)

Este desenho detalha o caminho crítico: cliente, API, autenticação, fila, trabalhador, Redis e banco relacional. Os diagramas Mermaid, incluindo a arquitetura macro, o fluxo detalhado, o modelo físico e o MER conceitual de pagamentos, estão em [diagramas_mermaid.md](diagramas_mermaid.md). O ponto essencial é que o pagamento não baixa o estoque diretamente: a unidade é reservada antes, e só a confirmação validada libera o ingresso.


#### Responsabilidade de cada etapa

1. **Cliente -> API:** o aplicativo ou a plataforma web envia evento, lote, quantidade e chave idempotente. A API valida o JWT, os parâmetros, o preço vigente e a permissão do comprador.
2. **API -> Auth Provider:** a API valida assinatura, expiração e escopos do JWT. Token inválido deve ser rejeitado antes de publicar qualquer pedido.
3. **API -> Redis:** a API solicita uma permissão de admissão por `tipo_ingresso_id`. O Redis mantém um contador ou semáforo limitado, com TTL, para evitar permissões presas.
4. **Redis -> API:** se houver capacidade, o Redis confirma a permissão; se o limite estiver atingido ou o Redis estiver indisponível, a API responde `429` ou `503` imediatamente, sem publicar na fila e sem acessar o banco.
5. **API -> Fila:** somente pedidos admitidos são registrados pela chave idempotente e publicados em uma mensagem durável. Se a publicação falhar, a permissão deve ser devolvida ao Redis.
6. **Fila -> Worker:** o worker consome pedidos admitidos e confirma a mensagem apenas depois de concluir a reserva e criar ou reconhecer a cobrança. Reprocessamentos devem ser seguros.
7. **Worker -> Banco:** o worker executa uma atualização condicional, como `qtd_disponivel = qtd_disponivel - 1 WHERE qtd_disponivel > 0`, dentro de uma transação. Nenhuma linha atualizada significa lote esgotado.
8. **Banco -> Worker:** se a transação for confirmada, o worker cria o ingresso com estado `RESERVADO` e prazo de expiração. Se não houver saldo, faz rollback e informa indisponibilidade sem gerar cobrança. O Redis não é fonte de estoque.
9. **Worker -> Gateway:** o serviço de pagamentos cria a cobrança usando `idempotency_key`, valor calculado no servidor e referência da reserva. O PARTIU não recebe nem armazena dados completos do cartão.
10. **Gateway -> Checkout:** o gateway processa PIX, débito, crédito ou outro meio suportado. A criação da cobrança significa apenas `PENDENTE`, nunca pagamento aprovado.
11. **Gateway -> Webhook:** o gateway envia um evento assinado com `event_id`, status, valor e identificador externo. Reenvios são esperados e não devem causar efeitos duplicados.
12. **Webhook -> API:** a API valida assinatura, gateway, `event_id`, valor, `gateway_payment_id` e vínculo com o pagamento. O evento é persistido em `WebhooksPagamento` antes ou durante o processamento idempotente.
13. **API -> Banco:** em uma transação, a API altera o pagamento e o ingresso apenas se o estado atual permitir a transição. Evento repetido deve ser reconhecido sem criar ingresso ou devolver estoque novamente.
14. **API -> Worker:** pagamento aprovado libera o QR Code e informa o novo estado. Pagamento recusado, cancelado ou reserva expirada marca o ingresso como `CANCELADO` e devolve a unidade ao lote uma única vez.
15. **Worker -> Redis:** após persistir o resultado, inclusive em erro, o worker libera a permissão em um bloco `finally`. O TTL permanece como proteção contra processos que terminem inesperadamente.
16. **API -> Cliente:** a API retorna ou disponibiliza o estado por consulta/polling ou notificação. O cliente só deve exibir o QR Code quando o estado persistido for `PAGO`.

O gateway é um sistema externo e pode ficar indisponível ou responder fora de ordem. Por isso, a aplicação deve tratar reprocessamento, timeout, conciliação periódica e estados intermediários como `aguardando_pagamento`, em vez de considerar a criação da cobrança como pagamento confirmado.


### 4. Modelo de dados

![Diagrama do banco de dados](PARTIU_DIAGRAM_DB.png)

O diagrama de banco representa as entidades e seus relacionamentos. `Pessoa` se relaciona com compras e permissões de organização; `Evento` possui tipos de ingresso; `Ingresso` liga uma pessoa a uma categoria; e `Pagamento` registra a cobrança externa. As cardinalidades ajudam a garantir que cada ingresso pertença a uma reserva rastreável e que o pagamento possa ser reconciliado com o gateway.

## Próximas decisões técnicas

- Definir o provedor de pagamentos e o formato de assinatura dos webhooks.
- Definir o tempo de reserva, política de cancelamento e comportamento para pagamentos tardios.
- Adicionar observabilidade, métricas de fila, alertas de estoque e uma rotina de reconciliação com o gateway.
