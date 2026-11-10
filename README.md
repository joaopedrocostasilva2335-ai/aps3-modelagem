# APS 3 — Serviço de Contratos

**Aluno:** João Pedro Costa Silva  |  **Projeto:** Gestão de Aprendizagem e Estágios da UniSenai (Grupo 6)

## Funcionalidade
Depois que a candidatura do estudante é confirmada e considerada elegível, o **Serviço de Contratos** gera o Termo de Compromisso e envia para a plataforma de assinatura externa. Quando as assinaturas chegam, o contrato fica **Ativo**.

| Linha de vida | Operações públicas |
|---|---|
| `ServicoContratos` | `gerarTermo(candidaturaId): TermoDTO`, `registrarAssinatura(contratoId, signatario): ContratoDTO` |
| `ServicoCandidaturas` | `verificarElegibilidade(candidaturaId): ElegibilidadeDTO` |
| `Repositorio` | `salvarContrato(contrato): ContratoDTO` |
| `PlataformaAssinatura` | `enviarParaAssinatura(TermoDTO)` |
| `API` | `POST /contratos`, `POST /webhooks/assinatura` |

## 1. Diagrama de Sequência
```mermaid
sequenceDiagram
    actor Estudante
    participant API
    participant SC as ServicoContratos
    participant SCand as ServicoCandidaturas
    participant Repo as Repositorio
    participant Assin as PlataformaAssinatura

    Estudante->>API: POST /contratos (candidaturaId)
    API->>SC: gerarTermo(candidaturaId)
    SC->>SCand: verificarElegibilidade(candidaturaId)
    SCand-->>SC: ElegibilidadeDTO

    alt elegível
        SC->>Repo: salvarContrato(contrato)
        Repo-->>SC: ContratoDTO
        SC-)Assin: enviarParaAssinatura(TermoDTO)
        SC-->>API: TermoDTO
    else não elegível
        SC-->>API: ElegibilidadeInvalidaException
    end

    Assin-)API: POST /webhooks/assinatura (contratoId)
    API->>SC: registrarAssinatura(contratoId, signatario)
    SC-->>API: ContratoDTO
```

## 2. Diagrama de Atividades
```mermaid
flowchart TB
    classDef bar fill:#000,stroke:#000,color:#fff;

    Inicio(("●"))

    subgraph R1["Estudante"]
        A1["Confirmar candidatura"]
        A2["Ver recusa"]
    end

    subgraph R2["Sistema"]
        D1{"Elegível?"}
        S1["Gerar termo"]
        F1["FORK"]:::bar
        J1["JOIN"]:::bar
        S2["Ativar contrato"]
    end

    subgraph R3["Plataforma de Assinatura"]
        P1["Assinatura do estudante"]
        P2["Assinatura da empresa"]
    end

    Fim1(((" ")))
    Fim2(((" ")))

    Inicio --> A1 --> D1
    D1 -->|"[elegível]"| S1 --> F1
    D1 -->|"[não elegível]"| A2 --> Fim2
    F1 --> P1 --> J1
    F1 --> P2 --> J1
    J1 --> S2 --> Fim1
```

## 3. Diagrama de Estados
```mermaid
stateDiagram-v2
    [*] --> EmFormalizacao : candidaturaConfirmada [elegivel] / gerarTermo()

    EmFormalizacao : entry / criarContrato()

    EmFormalizacao --> AguardandoAssinatura : termoGerado / enviarParaAssinatura()

    state AguardandoAssinatura {
        [*] --> SemAssinaturas
        SemAssinaturas --> ParcialmenteAssinado : assinaturaRecebida
    }

    AguardandoAssinatura --> Ativo : assinaturaRecebida [todasAssinadas] / ativarContrato()
    Ativo --> Rescindido : rescindir [motivoInformado] / registrarMotivo()
    Ativo --> Finalizado : vigenciaEncerrada

    Rescindido --> [*]
    Finalizado --> [*]
```

## 4. C4 Nível 2 (Contêineres)
```mermaid
C4Container
    title C4 Nível 2 - Serviço de Contratos

    Person(estudante, "Estudante", "Confirma a candidatura")

    System_Boundary(sis, "Sistema de Estágios UniSenai") {
        Container(api, "API da Aplicação", "Node.js", "Recebe solicitações e encaminha aos serviços")
        Container(cand, "Serviço de Candidaturas", "Node.js", "Verifica elegibilidade")
        Container(contr, "Serviço de Contratos", "Node.js", "Gera termo e controla o estado do contrato")
        ContainerDb(db, "Supabase", "PostgreSQL 16", "Candidaturas e contratos")
    }

    System_Ext(assin, "Plataforma de Assinatura", "Coleta as assinaturas")

    Rel(estudante, api, "Usa", "HTTPS/REST")
    Rel(api, contr, "gerarTermo", "HTTPS/REST")
    Rel(contr, cand, "verificarElegibilidade", "HTTPS/REST")
    Rel(contr, db, "Grava contratos", "SQL/TLS")
    Rel(contr, assin, "Envia termo (assíncrono)", "HTTPS/REST")
    Rel(assin, api, "Webhook de assinatura", "HTTPS/JSON")
```

## 5. ADR 0001 — Assinatura assíncrona via webhook


### Status
Proposto

### Contexto
O Serviço de Contratos envia o Termo de Compromisso para uma plataforma de assinatura externa. Os signatários levam minutos ou dias para assinar, e a plataforma pode ficar lenta ou indisponível. Uma chamada síncrona prenderia a requisição do estudante.

### Decisão
O termo é enviado de forma assíncrona (`enviarParaAssinatura`) e a API responde na hora. As assinaturas voltam por webhook (`POST /webhooks/assinatura`), que chama `registrarAssinatura`. O contrato só vai para `Ativo` quando todas as assinaturas chegarem.

### Consequências
**Vantagens:**
- A API não fica presa à plataforma externa.
- Uma falha da plataforma não derruba a candidatura.

**Riscos aceitos:**
- O estado do contrato só atualiza quando o webhook chega.
- O webhook precisa ser autenticado e tratado de forma idempotente.
