Como Especialista em UX/UI e Front-end, meu foco para a Agenda Unificada é reduzir a carga cognitiva. Uma agenda pode se tornar um caos visual rapidamente; portanto, a arquitetura deve priorizar o "escaneamento" rápido e a navegação deve ser infalível, especialmente para quem não utiliza o mouse.

Aqui está o roteiro de Arquitetura de Informação (AI) e o protocolo de acessibilidade.

---

## 1. Arquitetura de Informação (AI) - Mapa Mental

A estrutura é organizada em três níveis de profundidade para evitar que o usuário se perca.

### Nível 1: Visão Geral (Dashboard)

Barra Superior (Global): Busca global, Notificações (WebSockets em tempo real), Perfil e Configurações de Acessibilidade.

Barra Lateral (Navegação):

Troca de Visualização (Dia, Semana, Mês, Lista).

Filtros de Calendários (Pessoal, Trabalho, Webhooks Externos).

Botão de Ação Primária: "Novo Compromisso" (Destaque visual alto).

### Nível 2: Área de Trabalho (O Calendário)

Grade Principal: Onde os eventos são renderizados.

Resumo Lateral (Contextual): Lista de "Próximos Eventos do Dia" para facilitar a leitura rápida sem precisar interagir com a grade.

### Nível 3: Detalhes e Ações (Modais/Panels)

Drawer de Detalhes: Abre ao clicar/focar em um evento (evita mudar de página, mantém o contexto).

Formulário de Criação: Organizado por etapas (O que? Quando? Onde? Convidados?).

---

## 2. Regras de Navegação por Teclado (WCAG 2.1)

Para garantir que a aplicação seja acessível (Nível AA/AAA), a navegação deve seguir uma ordem lógica de tabulação (tabindex) e fornecer feedback visual claro.

### A. Ordem do Foco (Tab Flow)

Skip to Content: O primeiro item ao dar "Tab" deve ser um link invisível "Pular para o conteúdo principal".

Sidebar de Filtros: Navegação pelos calendários ativos.

Controles de Data: Setas para "Mês Anterior", "Hoje", "Próximo Mês".

Grade do Calendário: O foco entra na célula do dia atual.

Botão "Novo Evento".

### B. Interação na Grade (Grid Navigation)

A grade de calendário não deve ser um emaranhado de tabs. Usaremos o padrão ARIA Grid:

Setas (Direção): Movem o foco entre os dias (Cima, Baixo, Esquerda, Direita).

Enter / Espaço: Abre o Drawer de detalhes do evento ou o formulário de criação para aquele dia.

Home / End: Move para o primeiro ou último dia da semana visualizada.

Page Up / Page Down: Muda para o mês anterior ou próximo.

### C. Feedback Visual de Foco (Focus Indicator)

Regra de Ouro: Nunca remova o outline: none.

Estilo: O elemento focado deve ter um anel de alto contraste (ex: 2px solid #0056b3) com um offset de 2px para não encostar no conteúdo.

Baixa Visão: O indicador de foco deve ter uma taxa de contraste de pelo menos 4.5:1 contra o fundo.

---

## 3. Estados de Interface e Acessibilidade Sensorial

### Estados de Erro (À prova de erros)

Erro de Validação: Não use apenas cor (vermelho). Adicione um ícone de alerta e um texto descritivo (ex: "Data de término não pode ser anterior à de início").

Anúncios de Tela (Aria-live): Quando um novo evento chegar via WebSocket, o leitor de tela deve anunciar: "Novo compromisso: Reunião de Equipe às 14h adicionado".

### Estados de Carregamento (Skeleton Screens)

Enquanto os dados via Webhook são processados, exibimos Skeleton Screens (caixas cinzas pulsantes) que mantêm a estrutura da página, evitando que o conteúdo "pule" quando carregar (prevenindo problemas de CLS - Cumulative Layout Shift).

---

## 4. Tabela de Atalhos Rápidos (Cheat Sheet para o Usuário)

Implementaremos um modal de ajuda (acessível via ?) com os seguintes atalhos:

| Tecla | Ação |
| :--- | :--- |
| N | Criar Novo Evento |
| T | Ir para "Hoje" (Today) |
| M | Visualização por Mês |
| W | Visualização por Semana |
| D | Visualização por Dia |
| ESC | Fechar qualquer Modal ou Drawer |
| / | Focar na busca |

Recomendação Front-end:

Utilizar a tag `<main>` para o conteúdo principal, `<nav>` para menus e aria-current="page" para indicar qual visualização (Dia/Mês) está ativa. Cada célula do calendário deve ter um rótulo acessível claro, por exemplo: aria-label="15 de Outubro, 3 eventos".

O objetivo final: Um usuário com deficiência motora deve conseguir agendar uma reunião usando apenas uma mão no teclado, com a mesma velocidade de um usuário com mouse.
