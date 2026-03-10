# kiss-ihm 5G — Contexto do Projeto

## O que é esse projeto
Retrofit de um gerenciador de receitas industriais (fábricas/processos de produção).
- **Stack legada:** VB.NET Windows Forms + OPC DA + SQL Server
- **Stack nova (5G):** C# WPF MVVM + OPC UA + SQL Server (mesmo banco)
- **Solução:** `kiss-ihm-NIT01.sln` em `C:\PROJETO OPC UA\kiss-ihm_Fonte_NIT01_OPCUA`

## Projetos na solução
| Projeto | Descrição |
|---------|-----------|
| `kiss-ihm_View5G` | WPF/XAML, MVVM, Views já iniciadas |
| `kiss-ihm_Controller5G` | Controllers singleton, async, lógica de negócio |
| `kiss-ihm_Model5G` | SqlConnector singleton → banco `aetNIT01` em `.\SQLEXPRESS` |
| `kiss-ihm_Objects5G` | POCOs/DTOs (Process, Order, PLCCommand, Bin, Product...) |
| `WEF_OPCUA` | App OPC UA com módulo de manutenção (EF Core), já conecta ao CLP |
| `kiss-ihm_View` | Legacy VB.NET — usar como **spec**, não manter |

## Endpoint OPC UA
`opc.tcp://192.168.212.15:4840` (hardcoded em dois lugares — precisa virar config)

## Banco de dados
`aetNIT01` em `.\SQLEXPRESS`, autenticação Windows.

Tabelas principais:
- `tblProcess` — processos ativos (tipos: Batch, Discrete, Mill, Transport, TransportWithMixture, Site)
- `tblPLCCommand` — fila de comandos UI→PLC (status: 0=Wait, 1=Executing, 2=Executed, 99=Error)
- `tblOrder` — pedidos de produção
- `tblOrderComponent` — componentes do pedido (Source/Sink)
- `tblEM` — DB do PLC associada ao processo (ex: EM999 → TRFA-02, DB "DB_TRFA02")
- `tblEmBIN` — relação EM ↔ silo (define índice do silo no array da DB)
- `tblProcessObjectsEM` — qual EM é origem e qual é destino por processo
- `tblUNIT` — unidades do processo, define tipo de UDT (16+16, 32+32, 64+64 int+bit)
- `tblBin` — silos
- `tblProduct` — produtos
- `tblBinClass`, `tblBinPriority`, `tblBinPriorityGroup` — classificação/prioridade de silos
- `tblOrderColor` — mapeamento de cores por status de pedido

---

# Plano de Migração — 5 Fases

## Fase 1 — Consolidar camada OPC UA ✅ (planejada)
**Problema:** Dois clientes OPC UA distintos (`OpcUaController` em Controller5G e `OpcUaConnection` em WEF_OPCUA).

**Ação:**
- Criar `IOpcUaService` em `kiss-ihm_Controller5G/OpcUA/`
- Criar `OpcUaService.cs` consolidando os dois
- Criar `TagSubscriptionManager.cs` para gerenciar MonitoredItems

```
kiss-ihm_Controller5G/OpcUA/
  IOpcUaService.cs
  OpcUaService.cs
  TagSubscriptionManager.cs
```

## Fase 2 — Mapeamento Simbólico de Tags ✅ (planejada, revisada)
**Decisão arquitetural importante:**
OPC DA usava endereços absolutos calculados via offset em runtime (dependia de tblUNIT para saber tamanho da UDT).
OPC UA com DBs otimizadas no TIA Portal usa **simbologia** — não existe endereço absoluto.

**Consequência:** Não usar `tblTagMapping` com NodeIds fixos. Usar um `SymbolicTagResolver` que monta o NodeId a partir das tabelas do banco:

```
tblEM       → nome da DB ("DB_TRFA02")
tblEmBIN    → índice do silo no array (compatível com índice 1-based)
tblUNIT     → qual UDT (metadado de configuração, não mais cálculo de offset)
tblProcessObjectsEM → Source[n] ou Destination[n]
```

NodeId resultante (exemplo):
```
ns=3;s="DB_TRFA02"."Silos"[1]."PesoAtual"
ns=3;s="DB_TRFA02"."Unidades"[1]."Parametros"."VelocidadeMax"
```

**Requisito para TIA Portal:**
- DBs nomeadas por convenção: `DB_[PROCESSO]` (ex: `DB_TRFA02`, `DB_MOINHO01`)
- Arrays para silos e unidades com índice compatível com tblEmBIN
- Campos marcados como acessíveis via OPC UA
- DBs configuradas como **otimizadas**

## Fase 3 — ProcessControllerFactory + BaseProcessController ✅ (planejada)
**Estrutura:**
```
kiss-ihm_Controller5G/ProcessControllers/
  IProcessController.cs
  BaseProcessController.cs      ← loop genérico (poll tblPLCCommand + sync PLC→DB)
  TransportProcessController.cs ← já existe, completar
  BatchProcessController.cs
  DiscreteProcessController.cs
  MillProcessController.cs
  TransportWithMixtureProcessController.cs
  SiteProcessController.cs
  ProcessControllerFactory.cs   ← instancia por ProcessType
```

**BaseProcessController** roda background Task:
1. `ProcessPendingCommandsAsync()` — lê tblPLCCommand status=0, executa, atualiza status
2. `SyncPlcStateToDbAsync()` — lê OPC UA via Subscription → atualiza tblOrder

## Fase 4 — Migração das regras de negócio (Spec-Driven) ✅ (planejada)
**Estratégia:** Usar VB.NET legado como especificação.
Para cada form legado: mapear botão → `aetOPC_WriteDirectTag(grupo, endereço, valor)` → porta para `SymbolicTagResolver` + `IOpcUaService.WriteAsync`.

A regra de negócio não muda, só o transporte.

## Fase 5 — OPC UA Subscriptions em vez de polling ✅ (planejada)
Substituir polling de tags por MonitoredItems/Subscriptions OPC UA.
UI continua com polling leve no banco (1-2s) — banco já atualizado pelo controller via subscription.

---

# Decisões Arquiteturais

| Decisão | Escolha | Motivo |
|---------|---------|--------|
| Padrão UI→PLC | Manter tblPLCCommand | Desacoplamento UI/PLC, já funciona |
| Tag mapping | SymbolicTagResolver dinâmico | OPC UA simbólico elimina cálculo de offset |
| DB TIA Portal | Otimizadas | Obrigatório para acesso simbólico |
| OPC UA client | Consolidar em IOpcUaService | Eliminar duplicidade WEF_OPCUA / Controller5G |
| Inicialização | ProcessControllerFactory no App.xaml.cs | Um background task por processo ativo |

---

# Estado Atual

## Já implementado (5G)
- [x] MVVM base (BaseViewModel, RelayCommand)
- [x] SqlConnector singleton
- [x] OpcUaController (básico, em Controller5G)
- [x] OpcUaConnection (mais completo, em WEF_OPCUA)
- [x] TransportProcessController (só CreateNewOrder)
- [x] OrderTransportView + ViewModel
- [x] Views: ModernBin*, ModernProduct*, ModernBinClass*, ModernBinPriority*
- [x] MainWindow com status OPC UA e seleção de processos

## Pendente (por fase)
- [x] **F1:** IOpcUaService, OpcUaService consolidado, TagSubscriptionManager ✅
- [ ] **F2:** SymbolicTagResolver (consulta tblEM, tblEmBIN, tblUNIT, tblProcessObjectsEM)
- [ ] **F3:** BaseProcessController, ProcessControllerFactory, controllers Batch/Discrete/Mill/TransportWithMixture/Site
- [ ] **F4:** Portar regras de negócio de cada tipo de processo (ler VB.NET como spec)
- [ ] **F5:** Subscriptions OPC UA substituindo polling de tags
- [ ] Endpoint OPC UA em config (não hardcoded)
- [ ] Inicialização dos controllers no App.xaml.cs

---

# Notas Técnicas

- `tblUNIT` define o tipo de UDT do processo — em OPC UA vira metadado de configuração, não mais cálculo de offset de memória
- Processos com `IdProcess LIKE 'PLC%'` são filtrados da listagem na UI
- `IdPLCOrder` é gerado como random 4 dígitos de Ticks no momento da criação do pedido
- `OrderStatus` enum: 0=NULL, 10=Running, 20=Waiting, 60=Finished, 80=PLCFailure
- Cada processo pode ter múltiplas unidades (tblUNIT) e múltiplos silos (tblEmBIN)
