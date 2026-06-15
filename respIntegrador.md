Olá, como **Engenheiro DevOps e Arquiteto de Cloud**, meu foco é garantir que o sistema de gestão da Igreja Batista Betel seja resiliente, seguro e que o custo de operação seja o mais baixo possível (afinal, recursos em instituições devem ser otimizados).

O grande desafio aqui não é apenas o código, mas a **disponibilidade**. Se um gestor tentar resolver um conflito de escala no domingo de manhã e o sistema estiver fora do ar, o processo falha.

Aqui está o plano de infraestrutura e automação para este projeto:

---

### 1. Desenho da Infraestrutura (Cloud AWS)

Para garantir custo otimizado e alta disponibilidade, utilizaremos uma arquitetura **Serverless** (Sem servidor). Isso significa que a igreja só pagará quando alguém estiver usando o sistema.

*   **Frontend:** Hospedado no **AWS S3** com **CloudFront** (CDN). Isso garante que o sistema carregue instantaneamente em qualquer lugar e tenha custo quase zero.
*   **Backend (API):** Utilizaremos **AWS Lambda** com **API Gateway**. Essa estrutura escala automaticamente: se 1 gestor estiver usando ou se os 3 ministérios decidirem escalar todos os membros simultaneamente, o sistema não trava.
*   **Banco de Dados:** **Amazon RDS (PostgreSQL)**. Para a gestão de conflitos de agenda, um banco relacional é vital para garantir a consistência dos dados (evitar que o sistema diga que alguém está livre quando não está).
*   **Segurança:** **AWS Cognito** para gerenciar o login dos gestores e da Pastoral, garantindo autenticação segura e MFA (Múltiplo Fator de Autenticação).

---

### 2. Esteira de Integração e Entrega Contínua (CI/CD)

Como DevOps, meu objetivo é que qualquer correção ou nova funcionalidade (como um novo tipo de alerta de conflito) chegue aos gestores de forma rápida e sem erros.

*   **Ferramenta:** **GitHub Actions**.
*   **Fluxo:**
    1.  **Build & Test:** Sempre que um desenvolvedor subir código, testes automatizados verificarão a lógica de conflito de agendas.
    2.  **Security Scan:** Varredura automática em busca de vulnerabilidades antes de ir para o ar.
    3.  **Deploy Blue/Green:** O novo código é implantado ao lado do antigo. O tráfego só é direcionado para a nova versão se ela estiver 100% saudável. Isso garante **Zero Downtime** (sem tempo de inatividade).

---

### 3. Estratégia de Monitoramento e Observabilidade

Não podemos esperar um gestor reclamar que o sistema está lento.

*   **Logs e Métricas:** Usaremos o **CloudWatch** para monitorar erros nas APIs e tempo de resposta na verificação de escalas.
*   **Alertas:** Configurarei alertas via **SNS (Simple Notification Service)** integrados ao WhatsApp ou E-mail da Pastoral. Se o sistema detectar uma tentativa de acesso não autorizada ou um erro crítico, os administradores saberão em segundos.
*   **Dashboard de Saúde:** Um painel simples onde a Pastoral pode ver se o sistema está operando normalmente e quantos voluntários estão ativos no momento.

---

### 4. Solução de Conflitos sob a Ótica de Infraestrutura

Para resolver o problema técnico de dois ministérios tentarem alocar a mesma pessoa:

*   **Locks de Banco de Dados:** Implementaremos uma estratégia de "Optimistic Locking". Se o Gestor do Louvor e o da Mídia tentarem salvar uma escala para o mesmo membro no mesmo milissegundo, o sistema processará o primeiro e alertará o segundo instantaneamente que o status de disponibilidade do membro mudou.
*   **Cache de Disponibilidade:** Utilizaremos o **Redis** para manter em memória as escalas do dia. Isso permite que, ao abrir a agenda, o aviso de "Membro já alocado" apareça em milissegundos, antes mesmo do gestor tentar salvar.

---

### 5. Otimização de Custos

*   **Ambiente de Desenvolvimento:** Será desligado automaticamente fora do horário comercial.
*   **Escalabilidade Vertical do RDS:** O banco de dados terá sua capacidade aumentada automaticamente apenas nos períodos de pico de escalas (ex: quintas e domingos) e reduzida nos demais dias para economizar.

---

### Resumo do Plano DevOps:

1.  **Infra:** AWS Serverless (Pago por uso).
2.  **Segurança:** Dados sensíveis criptografados e acesso via MFA.
3.  **Atualização:** Automação total via GitHub Actions (sem sustos na hora de atualizar).
4.  **Resiliência:** Sistema distribuído em múltiplas zonas de disponibilidade.

Com essa estrutura, a Igreja Batista Betel terá uma ferramenta profissional, digna de grandes empresas, mas com um custo operacional adequado à sua realidade.

Podemos prosseguir com a definição das ferramentas de **IaC (Infrastructure as Code)**, como Terraform, para deixar essa infraestrutura documentada?
