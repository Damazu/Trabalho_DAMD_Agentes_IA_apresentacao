Como Arquiteto de Software Sênior, meu foco para o sistema da Igreja Batista Betel será estabelecer uma base técnica robusta que resolva o problema de concorrência e integridade de dados no agendamento, garantindo que as regras de negócio (muitas vezes complexas e políticas em uma igreja) sejam traduzidas em código modular e testável.

Abaixo, apresento o plano de ação focado na Engenharia de Backend e Arquitetura de Domínio.

---

## 1. Modelagem do Domínio (Domain-Driven Design - DDD)

O primeiro passo é isolar a complexidade do agendamento em um Bounded Context (Contexto Delimitado) específico.

Entidades Core:

Recurso (Sala, Auditório, Veículo, Equipamento): Possui capacidade e restrições.

Alocação (O evento em si): Define o "quem", "quando" e "onde".

Value Objects:

Periodo: Encapsula DataInicio e DataFim, contendo a lógica de validação de sobreposição (overlapsWith).

Aggregate Root:

Agenda: O ponto de entrada para garantir a consistência das alocações.

---

## 2. Estratégia de Concorrência e Bloqueio

Para evitar que dois ministérios reservem o mesmo salão no exato milésimo de segundo, utilizaremos uma abordagem híbrida:

Optimistic Locking (Bloqueio Otimista): Cada registro de Recurso ou Alocação terá uma coluna version. Se dois processos tentarem atualizar o mesmo recurso simultaneamente, o segundo falhará ao detectar que a versão mudou.

Database Constraints: Uma restrição de exclusão no PostgreSQL (usando EXCLUDE com GIST) para impedir períodos sobrepostos no mesmo RecursoId a nível de banco de dados.

---

## 3. Mecanismo de Resolução de Conflitos (Pattern: Strategy)

Nem todo conflito é um erro; alguns são questões de prioridade. Implementaremos o padrão Strategy para avaliar quem tem a precedência.

```csharp
public interface IConflictResolutionStrategy {
    ConflictResult Evaluate(Alocacao nova, Alocacao existente);
}

// Exemplo: Cultos de Celebração têm prioridade sobre ensaios.
public class PriorityBasedStrategy : IConflictResolutionStrategy {
    public ConflictResult Evaluate(Alocacao nova, Alocacao existente) {
        if (nova.Prioridade > existente.Prioridade) {
            return ConflictResult.SuggestRelocation(existente);
        }
        return ConflictResult.Reject(nova);
    }
}
```

---

## 4. Arquitetura em Camadas (Clean Architecture)

A lógica de alocação não deve depender de frameworks ou bancos de dados.

Domain Layer: Contém as regras puras de "o que é um conflito".

Application Layer (Use Cases): Coordena o processo:

Recebe pedido de reserva.

Carrega alocações existentes para o período via Repositório.

Chama o DomainService para validar conflitos.

Persiste ou retorna erro.

Infrastructure Layer: Implementa a persistência e mensageria (ex: enviar notificação ao líder se uma reserva for derrubada por uma de maior prioridade).

---

## 5. Plano de Implementação (Pseudocódigo Estrutural)

Para guiar os desenvolvedores, definimos a estrutura do serviço de reserva:

```csharp
public class BookingService : IBookingService 
{
    private readonly IRepository<Recurso> _recursoRepo;
    private readonly IConflictDetector _conflictDetector;
    private readonly IUnitOfWork _uow;

    public async Task<Result> CreateReservation(ReservationRequest request) 
    {
        // 1. Inicia transação para garantir atomicidade
        using var transaction = _uow.BeginTransaction();

        // 2. Bloqueia o recurso para leitura (Pessimistic Read ou Optimistic Check)
        var recurso = await _recursoRepo.GetByIdAsync(request.RecursoId);
        
        // 3. Verifica sobreposições no domínio
        var overlaps = await _recursoRepo.GetOverlappingAllocations(request.Periodo);

        if (overlaps.Any()) {
            var resolution = _conflictDetector.Resolve(request, overlaps);
            if (!resolution.IsSuccess) {
                return Result.Failure("Conflito de agenda detectado e sem prioridade.");
            }
            // Lógica para notificar/mover evento de menor prioridade
        }

        // 4. Cria a alocação
        var novaAlocacao = AlocacaoFactory.Create(request);
        recurso.AddAlocacao(novaAlocacao);

        await _uow.CommitAsync();
        return Result.Success();
    }
}
```

---

## 6. Roadmap de Ação Técnica

Semana 1: Refinamento de Esquema: Definir índices espaciais ou temporais no banco de dados para consultas rápidas de colisão de horários.

Semana 2: Domain Core: Implementar a lógica de Periodo e as regras de precedência dos ministérios (Ex: Ministério de Louvor vs. Evento Externo).

Semana 3: Idempotência de API: Garantir que se o usuário clicar duas vezes no botão "Reservar", o sistema processe apenas uma vez (usando Request-ID no header).

Semana 4: Auditoria e Log: Implementar um log de eventos (Event Sourcing simplificado) para rastrear quem alterou ou cancelou reservas, essencial para a transparência na gestão eclesial.

---

### Consideração Final de Escalabilidade

Embora o volume de transações de uma igreja não seja o de uma rede social, o custo de um erro (dois eventos no mesmo salão) é alto em termos de experiência do usuário. Esta arquitetura foca em consistência forte e rastreabilidade.
