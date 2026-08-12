

Readme · MD
# LAUDA — Sistema de Geração Inteligente de Documentos
 
> Sistema que permite ao usuário cadastrar um documento original e gerar novas versões automaticamente com Inteligência Artificial, mantendo estrutura, formatação e coerência com o conteúdo original, a partir de um simples prompt.
 
---
 
## O Problema
 
Formatar documentos manualmente toda vez que o conteúdo muda é lento e repetitivo. O objetivo deste projeto é economizar tempo: o usuário cadastra um documento base uma única vez, e a partir daí só precisa descrever o que quer mudar. A IA reescreve o conteúdo mantendo formatação, estrutura e coerência com o restante do texto.
 
Este repositório documenta a arquitetura e as decisões técnicas do projeto. O código-fonte do backend e do serviço gerador é privado, por se tratar de propriedade intelectual de um produto em fase de lançamento. Trechos de código são apresentados aqui como print, para ilustrar decisões específicas.
 
---
 
## Arquitetura Geral
 
O sistema é dividido em três serviços independentes, comunicando-se de forma assíncrona:
 
```
┌─────────────┐        ┌──────────────┐        ┌────────────────────┐
│  Frontend   │──HTTP──▶│   Backend    │──AMQP──▶│  Gerador de Docs   │
│  (Next.js)  │◀─WS────│   (NestJS)   │◀──AMQP──│     (Express)      │
└─────────────┘        └──────┬───────┘        └─────────┬──────────┘
                               │                           │
                          ┌────▼────┐                 ┌────▼────┐
                          │  Redis   │                 │ RabbitMQ│
                          │ (cache)  │                 │ (fila)  │
                          └─────────┘                 └─────────┘
                               │
                          ┌────▼────┐
                          │ Postgres │
                          └─────────┘
```
 
Fluxo resumido:
 
1. Usuário cadastra o documento original e envia um prompt via frontend (Next.js)
2. O backend (NestJS) recebe a requisição e publica uma mensagem na fila (RabbitMQ)
3. O serviço gerador (Express) consome a mensagem, processa o conteúdo com IA mantendo a formatação original
4. O resultado é enviado de volta ao backend, que notifica o frontend em tempo real via WebSocket
5. Dados temporários (cadastro, recuperação de senha) ficam em Redis, evitando sobrecarga no banco relacional
---
 
## Stack Técnica
 
| Camada | Tecnologia |
|---|---|
| Frontend | Next.js (React) |
| Backend | NestJS |
| Serviço Gerador (IA) | Express |
| Mensageria | RabbitMQ |
| Cache | Redis |
| Banco de Dados | PostgreSQL |
| Comunicação em tempo real | WebSocket |
| Containerização | Docker |
| Versionamento | Git / GitHub |
 
---
 
## Decisões Técnicas
 
### Por que separar o gerador de documentos em um microsserviço?
 
O processamento com IA é naturalmente mais lento que uma requisição HTTP comum. Manter essa lógica dentro do backend principal criaria bloqueios e tempos de espera desnecessários. Isolando em um serviço próprio, o backend continua responsivo enquanto o processamento acontece de forma assíncrona.
 
### Por que mensageria (RabbitMQ) em vez de chamada direta entre serviços?
 
Se o gerador demorar ou cair no meio do processamento, uma chamada HTTP direta deixaria o backend esperando indefinidamente. Com fila:
 
- O backend publica a mensagem e segue livre
- O gerador consome e confirma via ACK somente após concluir
- Se cair antes de confirmar, a mensagem retorna automaticamente para a fila
- Mensagens com erro recorrente vão para uma Dead Letter Queue, isoladas para tratamento posterior
### Por que Redis para cadastro e recuperação de senha?
 
São dados de vida curta, como um código de confirmação que expira em minutos. Persistir isso em um banco relacional geraria escrita e leitura desnecessária em uma estrutura pensada para dados de longo prazo. O cache resolve isso com muito mais velocidade e sem sobrecarregar o Postgres.
 
### Por que WebSocket em vez de HTTP tradicional?
 
Como a geração do documento pode levar algum tempo, o frontend usa WebSocket para:
 
- Bloquear ações indevidas do usuário durante o processamento, como gerar novamente antes de concluir
- Exibir status em tempo real
- Notificar com um alerta claro quando o documento é salvo com sucesso
---
 
## Trechos de Implementação
 
(Espaço reservado para prints de código específicos: consumer da fila com ACK, lógica de cache, tratamento da Dead Letter Queue)
 
---
 
## Status do Projeto
 
Em fase final de testes antes do lançamento público. Frontend disponível em repositório separado e público.
 
---
 
## Contato
 
Márcio Roberto Lisboa de Abreu
Belém, Pará
[LinkedIn] · [GitHub]
 
