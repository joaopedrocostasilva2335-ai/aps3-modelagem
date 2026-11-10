# APS 3 — Serviço de Contratos

**Aluno:** [Seu Nome]  |  **Projeto:** Gestão de Aprendizagem e Estágios da UniSenai (Grupo 6)

## Funcionalidade
Depois que a candidatura do estudante é confirmada e considerada elegível, o **Serviço de Contratos** gera o Termo de Compromisso e envia para a plataforma de assinatura externa. Quando todas as assinaturas chegam, o contrato fica **Ativo**.

**Operações públicas por linha de vida** (regra de consistência do Diagrama de Sequência):

| Linha de vida | Operações públicas |
|---|---|
| `ServicoContratos` | `gerarTermo(candidaturaId): TermoDTO`, `registrarAssinatura(contratoId, signatario): ContratoDTO` |
| `ServicoCandidaturas` | `verificarElegibilidade(candidaturaId): ElegibilidadeDTO` |
| `Repositorio` | `salvarContrato(contrato): ContratoDTO`, `atualizarEstado(contratoId, estado): ContratoDTO` |
| `PlataformaAssinatura` | `enviarParaAssinatura(TermoDTO)` |
| `API da Aplicação` | `POST /contratos`, `POST /webhooks/assinatura` |

## 1. Diagrama de Sequência
```mermaid
sequenceDiagram
    actor Estudante
    participant API as API da Aplicação
    participant SC as ServicoContratos
    participant SCand as ServicoCandidaturas
    participant Repo as Repositorio (Supabase)
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
        API-->>Estudante: 201 Created
    else não elegível
        SC-->>API: ElegibilidadeInvalidaException
        API-->>Estudante: 422 Unprocessable Entity
    end

    Assin-)API: POST /webhooks/assinatura (contratoId, signatario)
    API->>SC: registrarAssinatura(contratoId, signatario)
    SC->>Repo: atualizarEstado(contratoId, Ativo)
    Repo-->>SC: ContratoDTO
```

## 2. Diagrama de Atividades
```mermaid
flowchart TB
    classDef bar fill:#000,stroke:#000,color:#fff;

    Inicio(("●"))

    subgraph R1["Estudante"]
        A1["Confirmar candidatura"]
        A2["Visualizar recusa"]
    end

    subgraph R2["Sistema (ServicoContratos)"]
        S1["verificarElegibilidade()"]
        D1{"Elegível?"}
        S2["gerarTermo()"]
        S3["enviarParaAssinatura()"]
        F1["FORK"]:::bar
        J1["JOIN"]:::bar
        S4["ativarContrato()"]
    end

    subgraph R3["Plataforma de Assinatura (externa)"]
        P1["Assinatura do estudante"]
        P2["Assinatura da empresa"]
        P3["Assinatura da instituição"]
    end

    Fim1(((" ")))
    Fim2(((" ")))

    Inicio --> A1 --> S1 --> D1
    D1 -->|"[elegível]"| S2 --> S3 --> F1
    D1 -->|"[não elegível]"| A2 --> Fim2
    F1 --> P1 --> J1
    F1 --> P2 --> J1
    F1 --> P3 --> J1
    J1 --> S4 --> Fim1
```

## 3. Diagrama de Estados
```mermaid
stateDiagram-v2
    [*] --> EmFormalizacao : candidaturaConfirmada [candidaturaElegivel] / criarContrato()

    EmFormalizacao : entry / gerarTermo()

    EmFormalizacao --> AguardandoAssinaturaFiep : termoGerado / enviarParaAssinatura()

    state AguardandoAssinaturaFiep {
        [*] --> SemAssinaturas
        SemAssinaturas --> ParcialmenteAssinado : assinaturaRecebida
    }

    AguardandoAssinaturaFiep --> Ativo : assinaturaRecebida [todasAssinadas] / ativarContrato()
    AguardandoAssinaturaFiep --> Rescindido : rescindir [motivoInformado] / registrarMotivo()
    Ativo --> Rescindido : rescindir [motivoInformado] / registrarMotivo()
    Ativo --> Finalizado : vigenciaEncerrada

    Rescindido --> [*]
    Finalizado --> [*]
```

## 4. C4 Nível 2 (Contêineres)
```mermaid
C4Container
    title C4 Nível 2 - Serviço de Contratos

    Person(estudante, "Estudante", "Confirma a candidatura e assina o termo")

    System_Boundary(sis, "Sistema de Estágios UniSenai") {
        Container(web, "Aplicação Web", "React", "Interface do usuário")
        Container(api, "API da Aplicação", "Node.js / NestJS", "Recebe solicitações e encaminha aos serviços")
        Container(cand, "Serviço de Candidaturas", "Node.js", "Verifica elegibilidade")
        Container(contr, "Serviço de Contratos", "Node.js", "Gera termo e controla o estado do contrato")
        ContainerDb(db, "Supabase", "PostgreSQL 16", "Candidaturas e contratos")
    }

    System_Ext(assin, "Plataforma de Assinatura Eletrônica", "Coleta as assinaturas")

    Rel(estudante, web, "Usa", "HTTPS")
    Rel(web, api, "Chama", "HTTPS/REST")
    Rel(api, contr, "gerarTermo, registrarAssinatura", "HTTPS/REST")
    Rel(contr, cand, "verificarElegibilidade", "HTTPS/REST")
    Rel(contr, db, "Lê e grava contratos", "SQL/TLS")
    Rel(cand, db, "Lê e grava candidaturas", "SQL/TLS")
    Rel(contr, assin, "Envia termo (assíncrono)", "HTTPS/REST")
    Rel(assin, api, "Notifica assinatura (webhook)", "HTTPS/JSON")
```

## 5. ADR
[ADR 0001 — Assinatura assíncrona via webhook](docs/architecture/adr-0001.md)
