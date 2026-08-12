

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

### por que a decisão das stacks?

O Next.js foi escolhido por ser um framework em que, por padrão, os componentes são executados no servidor e não no navegador, o que torna a aplicação mais rápida e segura, já que chaves de autenticação e informações sensíveis nunca chegam ao usuário final.

No backend, optou-se pelo NestJS. Como a intenção é adicionar cada vez mais funcionalidades ao sistema, ele foi escolhido para facilitar a organização futura do código. Além disso, o framework adota princípios de microsserviços, com uma estrutura modular que se alinha bem à proposta do sistema.

No gerador, por se tratar apenas do motor de geração de documentos, optou-se por um framework mais leve e simples, como o Express. Nas primeiras versões, era utilizado Python com FastAPI, por sua melhor integração com as LLMs disponíveis no mercado. No entanto, o JavaScript se saiu melhor na manipulação das runs presentes nos documentos .docx, o que levou à escolha de JavaScript e Express no gerador definitivo.
 
### Por que separar o gerador de documentos em um microsserviço?
 
O processamento com IA é naturalmente mais lento que uma requisição HTTP comum. Manter essa lógica dentro do backend principal criaria bloqueios e tempos de espera desnecessários. Ao isolá-la em um serviço próprio, o backend permanece responsivo enquanto o processamento ocorre de forma assíncrona. Além disso, a decisão também se deve à necessidade de evitar que a IA tivesse acesso direto ao servidor ou ao banco de dadosm, o objetivo era isolá-la ao máximo.
 
### Por que mensageria (RabbitMQ) em vez de chamada direta entre serviços?
 
Considerando os imprevistos que podem ocorrer no sistema, seria arriscado fazer uma requisição direta ao microsserviço e aguardar a resposta: o sistema ficaria travado por tempo indeterminado. Dependendo da quantidade de páginas a serem processadas ou reescritas no documento, o tempo de processamento pode variar bastante, e há ainda a dependência de uma chamada de API para utilizar a IA.

Por isso, optou-se pelo sistema de mensageria, no qual ambos os serviços seguem seu ciclo de forma assíncrona enquanto o outro executa alguma tarefa. A mensagem só sai da fila quando o consumer confirma o recebimento via ack; assim, se alguma das partes ficar indisponível, a comunicação entre os sistemas continua garantida.
 
- O backend publica a mensagem e segue livre
- O gerador consome e confirma via ACK somente após concluir
- Se cair antes de confirmar, a mensagem retorna automaticamente para a fila
- Mensagens com erro recorrente vão para uma Dead Letter Queue, isoladas para tratamento posterior
### Por que Redis?
 
O Redis foi utilizado para dados que não fariam sentido serem persistidos no banco de dados informações cuja utilidade se limita a uma janela curta de tempo, como os códigos de confirmação enviados por e-mail. Esses códigos são usados na confirmação da criação da conta no sistema, para verificar se o e-mail existe, e também na confirmação da troca de senha do usuário.
 
### Por que WebSocket em vez de HTTP tradicional?
 
Como a geração do documento pode levar algum tempo, o frontend usa WebSocket para:
 
- Bloquear ações indevidas do usuário durante o processamento, como gerar novamente antes de concluir
- Exibir status em tempo real
- Notificar com um alerta claro quando o documento é salvo com sucesso
---
 
## Status do Projeto
 
No momento ele está em produção e rodando em lauda.work e é constantemente atualizado.
 
---
 
## Contato
 
Márcio Roberto Lisboa de Abreu
Belém, Pará
[LinkedIn] · [GitHub]
 
