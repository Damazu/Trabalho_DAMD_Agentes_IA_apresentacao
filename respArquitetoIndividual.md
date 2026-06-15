Como Arquiteto de Software, para lidar com a complexidade de uma Agenda Unificada que recebe dados de múltiplas fontes (Webhooks) e precisa de reatividade (WebSockets), aplicarei o Domain-Driven Design (DDD).

O objetivo é evitar o "Modelo Anêmico" e garantir que as regras de colisão e disponibilidade fiquem protegidas dentro do Domínio.

## 1. Design Estratégico: Subdomínios e Contextos

Dividiremos o sistema em Bouded Contexts (Contextos Delimitados) claros para evitar o acoplamento:

Contexto de Agendamento (Core Domain): Gestão do ciclo de vida do Evento.

Contexto de Disponibilidade/Alocação (Core Domain): Algoritmos de detecção de conflitos e reserva de recursos (salas, pessoas, equipamentos).

Contexto de Integração (Supporting Subdomain): Tradução de Webhooks externos (Google, Outlook) para o formato interno.

Contexto de Notificação (Generic Subdomain): Gerencia disparos de WebSockets e E-mails.

## 1. Estrutura de Módulos (Tactical Design)

Seguiremos a Clean Architecture dentro de cada módulo do DDD:

```
src/
└── Modules/
    ├── Scheduling/
    │   ├── Domain/              # Entidades, Value Objects, Domain Services
    │   │   ├── Aggregates/      # EventAggregate.ts
    │   │   ├── Events/          # EventCreated.ts (Domain Events)
    │   │   └── Repository/      # IEventRepository.ts (Interfaces)
    │   ├── Application/         # Use Cases (Commands/Queries)
    │   │   ├── CreateEvent/
    │   │   └── SyncExternalEvent/
    │   └── Infrastructure/      # Implementações (TypeORM, Redis, etc)
    ├── Allocation/
    │   ├── Domain/
    │   │   ├── Services/        # AvailabilityService.ts (Regra de conflito)
    │   │   └── Entities/        # ResourceAllocation.ts
    │   └── Application/
    │       └── Handlers/        # Reage ao evento EventCreated do módulo Scheduling
```

## 1. Modelo de Domínio: Eventos vs. Alocações

Aqui aplicamos a lógica de negócio pesada. Um Evento é a intenção; uma Alocação é o fato consumado no tempo e espaço.

### A. Entidade: Event (Aggregate Root)

O Evento coordena o "quê".

Campos: Id, Title, Description, OwnerId, Status (Draft, Confirmed, Cancelled).

Comportamento: confirm(), cancel(), reschedule().

### B. Entidade: Allocation

A Alocação gerencia o "quem/onde" e evita o double-booking.

Campos: ResourceId (pode ser um UUID de usuário ou sala), TimeRange (Start/End), EventId.

Restrição: Duas alocações para o mesmo ResourceId não podem ter sobreposição de TimeRange.

### C. Value Object: TimeRange (O Coração da Validação)

Para evitar lógica duplicada, o TimeRange encapsula a verificação de períodos.

```typescript
// Exemplo de lógica no Domínio (Pseudocódigo)
class TimeRange {
  constructor(private readonly start: Date, private readonly end: Date) {
    if (end <= start) throw new DomainError("Data de fim deve ser após o início");
  }

  overlapsWith(other: TimeRange): boolean {
    return this.start < other.end && other.start < this.end;
  }
}
```

## 1. Fluxo de Negócio entre Módulos

Quando um Webhook externo (ex: Calendário do Google) envia um novo compromisso:

Integration Layer: Recebe o Webhook e dispara um SyncExternalEventCommand.

Scheduling Module:

Cria a entidade Event.

Dispara um Domain Event: EventCreated.

Allocation Module (Subscriber):

Escuta o EventCreated.

O AvailabilityService verifica se o ResourceId (usuário) está livre naquele TimeRange.

Se livre, cria a Allocation e marca o evento como Confirmed.

Se houver conflito, marca o evento como Conflict e dispara um alerta via WebSocket.

## 1. Algoritmo de Verificação de Conflitos (Domain Service)

Para garantir escalabilidade, a busca de conflitos no PostgreSQL usa índices GiST (Generalized Search Tree) com o operador de range:

```sql
-- Exemplo de query eficiente no Infrastructure Layer
SELECT count(*) FROM allocations 
WHERE resource_id = $1 
AND time_period && tsrange($2, $3); -- Operador && verifica sobreposição
```

## 1. Comunicação via WebSockets e Webhooks

Webhooks (Ingress): Atuam como ACL (Anti-Corruption Layer). Eles convertem o payload sujo de terceiros para os comandos do nosso domínio.

WebSockets (Egress): O módulo de Notificações assina os eventos de integração (AllocationConfirmed, AllocationFailed). Ao detectar uma falha de alocação (conflito), ele envia imediatamente uma mensagem para o socket do usuário: { "type": "CONFLICT_DETECTED", "payload": { "eventId": "..." } }.

Vantagem desta Arquitetura:

Se amanhã precisarmos mudar de um Monólito Modular para Microsserviços, os contextos já estão separados. A lógica de "não permitir dois eventos no mesmo horário" está protegida no Domínio de Alocação, independentemente de onde o dado venha (Interface Humana ou Webhook).
