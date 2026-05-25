# Visual Architecture Guide

This document provides visual diagrams to help understand the career-ops system architecture.

> **Note:** These diagrams use Mermaid syntax and will render in GitHub, VS Code, and most modern markdown viewers.

---

## 1. System Overview

```mermaid
graph TB
    User[👤 User] --> Input[Input Methods]
    Input --> Manual[Paste URL/JD]
    Input --> Scanner[Portal Scanner]
    Input --> Batch[Batch Import]
    
    Manual --> Pipeline[📋 data/pipeline.md]
    Scanner --> Pipeline
    Batch --> Pipeline
    
    Pipeline --> Agent[🤖 AI Agent]
    Agent --> Modes[modes/*.md]
    Modes --> Evaluate[Evaluation Engine]
    
    Evaluate --> BlockA[A: Role Summary]
    Evaluate --> BlockB[B: CV Match]
    Evaluate --> BlockC[C: Level Strategy]
    Evaluate --> BlockD[D: Compensation]
    Evaluate --> BlockE[E: Personalization]
    Evaluate --> BlockF[F: Interview Prep]
    Evaluate --> BlockG[G: Legitimacy]
    
    BlockA --> Output[Output Generator]
    BlockB --> Output
    BlockC --> Output
    BlockD --> Output
    BlockE --> Output
    BlockF --> Output
    BlockG --> Output
    
    Output --> Report[📄 Report.md]
    Output --> PDF[📑 CV PDF]
    Output --> TSV[📊 Tracker Entry]
    
    TSV --> Merge[merge-tracker.mjs]
    Merge --> Tracker[🗂️ applications.md]
    
    style User fill:#e1f5ff
    style Agent fill:#fff4e1
    style Tracker fill:#e8f5e9
    style Report fill:#f3e5f5
    style PDF fill:#fff3e0
```

---

## 2. Data Flow Architecture

```mermaid
graph LR
    subgraph Input Layer
        CV[cv.md]
        Profile[config/profile.yml]
        Digest[article-digest.md]
        Portals[portals.yml]
    end
    
    subgraph Processing Layer
        Agent[AI Agent]
        Modes[modes/*.md]
        Scripts[*.mjs Scripts]
    end
    
    subgraph Output Layer
        Reports[reports/]
        PDFs[output/]
        TrackerTSV[batch/tracker-additions/]
    end
    
    subgraph State Layer
        Pipeline[data/pipeline.md]
        Tracker[data/applications.md]
        History[data/scan-history.tsv]
    end
    
    CV --> Agent
    Profile --> Agent
    Digest --> Agent
    Portals --> Scripts
    
    Agent --> Modes
    Agent --> Reports
    Agent --> PDFs
    Agent --> TrackerTSV
    
    Scripts --> Pipeline
    Scripts --> History
    
    TrackerTSV --> Scripts
    Scripts --> Tracker
    
    style Input Layer fill:#e3f2fd
    style Processing Layer fill:#fff3e0
    style Output Layer fill:#f3e5f5
    style State Layer fill:#e8f5e9
```

---

## 3. Evaluation Pipeline (Single Job)

```mermaid
sequenceDiagram
    participant User
    participant Agent as AI Agent
    participant Browser as Playwright/WebFetch
    participant Modes as modes/oferta.md
    participant PDF as generate-pdf.mjs
    participant Merge as merge-tracker.mjs
    participant Tracker as applications.md
    
    User->>Agent: Paste job URL
    Agent->>Browser: Fetch job description
    Browser-->>Agent: JD content
    
    Agent->>Modes: Evaluate (A-F blocks)
    Note over Modes: A: Role Summary<br/>B: CV Match<br/>C: Level Strategy<br/>D: Compensation<br/>E: Personalization<br/>F: Interview Prep<br/>G: Legitimacy
    
    Modes-->>Agent: Score + Analysis
    
    Agent->>Agent: Generate report
    Agent->>Agent: Write reports/{num}-{company}-{date}.md
    
    Agent->>PDF: Generate tailored CV
    PDF-->>Agent: PDF created
    
    Agent->>Agent: Write TSV to batch/tracker-additions/
    Agent->>Merge: Trigger merge
    Merge->>Tracker: Update applications.md
    
    Tracker-->>User: ✅ Job evaluated & tracked
```

---

## 4. Scanner Architecture

```mermaid
graph TD
    Start[node scan.mjs] --> Load[Load portals.yml]
    Load --> Loop{For each company}
    
    Loop --> Detect[Detect ATS Provider]
    Detect --> GH{Greenhouse?}
    Detect --> ASH{Ashby?}
    Detect --> LEV{Lever?}
    
    GH -->|Yes| GHProvider[providers/greenhouse.mjs]
    ASH -->|Yes| ASHProvider[providers/ashby.mjs]
    LEV -->|Yes| LEVProvider[providers/lever.mjs]
    
    GHProvider --> API[HTTP GET /api/v1/boards/{id}/jobs]
    ASHProvider --> API
    LEVProvider --> API
    
    API --> Filter[Filter by title/location]
    Filter --> Dedup[Check scan-history.tsv]
    Dedup --> New{New jobs?}
    
    New -->|Yes| Append[Append to pipeline.md]
    New -->|Yes| Log[Log to scan-history.tsv]
    New -->|No| Loop
    
    Append --> Loop
    Log --> Loop
    Loop --> End[Scan complete]
    
    style Start fill:#4caf50,color:#fff
    style End fill:#4caf50,color:#fff
    style API fill:#2196f3,color:#fff
    style Append fill:#ff9800,color:#fff
```

---

## 5. Batch Processing Workflow

```mermaid
graph TB
    Input[batch-input.tsv] --> Runner[batch-runner.sh]
    Runner --> State[Load batch-state.tsv]
    
    State --> Spawn{Spawn Workers}
    Spawn --> W1[Worker 1<br/>claude -p]
    Spawn --> W2[Worker 2<br/>claude -p]
    Spawn --> W3[Worker N<br/>claude -p]
    
    W1 --> Prompt1[Read batch-prompt.md]
    W2 --> Prompt2[Read batch-prompt.md]
    W3 --> Prompt3[Read batch-prompt.md]
    
    Prompt1 --> Eval1[Evaluate 1 job]
    Prompt2 --> Eval2[Evaluate 1 job]
    Prompt3 --> Eval3[Evaluate 1 job]
    
    Eval1 --> Out1[Write report + PDF + TSV]
    Eval2 --> Out2[Write report + PDF + TSV]
    Eval3 --> Out3[Write report + PDF + TSV]
    
    Out1 --> Exit1[Exit]
    Out2 --> Exit2[Exit]
    Out3 --> Exit3[Exit]
    
    Exit1 --> Check{All complete?}
    Exit2 --> Check
    Exit3 --> Check
    
    Check -->|No| State
    Check -->|Yes| Merge[merge-tracker.mjs]
    Merge --> Done[✅ Batch complete]
    
    style Runner fill:#4caf50,color:#fff
    style Done fill:#4caf50,color:#fff
    style W1 fill:#2196f3,color:#fff
    style W2 fill:#2196f3,color:#fff
    style W3 fill:#2196f3,color:#fff
```

---

## 6. File Structure & Data Flow

```mermaid
graph LR
    subgraph User Layer<br/>"Never auto-updated"
        CV[cv.md]
        Profile[config/profile.yml]
        CustomProfile[modes/_profile.md]
        Portals[portals.yml]
        Digest[article-digest.md]
    end
    
    subgraph System Layer<br/>"Auto-updatable"
        SharedModes[modes/_shared.md]
        OtherModes[modes/oferta.md<br/>modes/batch.md<br/>etc.]
        Scripts[*.mjs scripts]
        Templates[templates/*]
    end
    
    subgraph Data Layer<br/>"User data"
        Pipeline[data/pipeline.md]
        Tracker[data/applications.md]
        Reports[reports/*.md]
        PDFs[output/*.pdf]
    end
    
    CV -.->|Read by| System Layer
    Profile -.->|Read by| System Layer
    CustomProfile -.->|Read by| System Layer
    Portals -.->|Read by| System Layer
    Digest -.->|Read by| System Layer
    
    System Layer -->|Generates| Data Layer
    
    style User Layer fill:#e8f5e9
    style System Layer fill:#fff3e0
    style Data Layer fill:#e3f2fd
```

---

## 7. Evaluation Blocks (A-G)

```mermaid
graph TD
    Start[Job Description] --> A[Block A: Role Summary]
    A --> A1[Classify archetype<br/>1 of 6 types]
    A --> A2[Extract key info]
    A --> A3[Quick fit assessment]
    
    A --> B[Block B: CV Match]
    B --> B1[Compare CV vs JD]
    B --> B2[Identify gaps]
    B --> B3[Mitigation strategies]
    
    B --> C[Block C: Level Strategy]
    C --> C1[Seniority analysis]
    C --> C2[Title positioning]
    C --> C3[Negotiation angle]
    
    C --> D[Block D: Compensation]
    D --> D1[Market research]
    D --> D2[Salary bands]
    D --> D3[Total comp breakdown]
    
    D --> E[Block E: Personalization]
    E --> E1[CV tailoring plan]
    E --> E2[Keyword injection]
    E --> E3[Proof point mapping]
    
    E --> F[Block F: Interview Prep]
    F --> F1[STAR+R stories]
    F --> F2[Company research]
    F --> F3[Question bank]
    
    F --> G[Block G: Legitimacy]
    G --> G1[Posting quality check]
    G --> G2[Red flag detection]
    G --> G3[Tier classification]
    
    G --> Score[Calculate Score<br/>10 dimensions<br/>1-5 scale]
    Score --> Report[Generate Report]
    
    style A fill:#e3f2fd
    style B fill:#f3e5f5
    style C fill:#fff3e0
    style D fill:#e8f5e9
    style E fill:#fce4ec
    style F fill:#fff9c4
    style G fill:#ffccbc
    style Score fill:#4caf50,color:#fff
```

---

## 8. Integrity Scripts Flow

```mermaid
graph TD
    subgraph Maintenance Scripts
        Verify[verify-pipeline.mjs]
        Normalize[normalize-statuses.mjs]
        Dedup[dedup-tracker.mjs]
        Merge[merge-tracker.mjs]
        Doctor[doctor.mjs]
    end
    
    Tracker[data/applications.md] --> Verify
    Verify --> Check1{Format OK?}
    Check1 -->|No| Fix1[Report errors]
    Check1 -->|Yes| Pass1[✅]
    
    Tracker --> Normalize
    Normalize --> Check2{Statuses canonical?}
    Check2 -->|No| Fix2[Normalize to templates/states.yml]
    Check2 -->|Yes| Pass2[✅]
    
    Tracker --> Dedup
    Dedup --> Check3{Duplicates?}
    Check3 -->|Yes| Remove[Remove duplicates]
    Check3 -->|No| Pass3[✅]
    
    TSV[batch/tracker-additions/*.tsv] --> Merge
    Merge --> Validate[Validate TSV format]
    Validate --> Append[Append to tracker]
    Append --> Cleanup[Delete TSV files]
    
    System[System files] --> Doctor
    Doctor --> CheckNode[Node.js 18+?]
    Doctor --> CheckDeps[Dependencies OK?]
    Doctor --> CheckFiles[Required files exist?]
    CheckNode --> Health[Health report]
    CheckDeps --> Health
    CheckFiles --> Health
    
    style Pass1 fill:#4caf50,color:#fff
    style Pass2 fill:#4caf50,color:#fff
    style Pass3 fill:#4caf50,color:#fff
    style Health fill:#4caf50,color:#fff
    style Fix1 fill:#f44336,color:#fff
    style Fix2 fill:#ff9800,color:#fff
```

---

## 9. Mode Interaction Diagram

```mermaid
graph TD
    User[User Input] --> Auto{Input Type?}
    
    Auto -->|URL or JD text| AutoPipe[modes/auto-pipeline.md]
    Auto -->|"Evaluate"| Oferta[modes/oferta.md]
    Auto -->|"Compare"| Ofertas[modes/ofertas.md]
    Auto -->|"Scan"| Scan[modes/scan.md]
    Auto -->|"Apply"| Apply[modes/apply.md]
    Auto -->|"Pipeline"| Pipeline[modes/pipeline.md]
    Auto -->|"Batch"| Batch[modes/batch.md]
    
    AutoPipe --> Shared[modes/_shared.md<br/>Core logic]
    Oferta --> Shared
    Ofertas --> Shared
    
    Shared --> Custom[modes/_profile.md<br/>User customizations]
    
    Custom --> Eval[Evaluation Engine]
    
    Scan --> ScanScript[scan.mjs]
    Apply --> Browser[Playwright]
    Pipeline --> Loop[Process each URL]
    Batch --> BatchRunner[batch-runner.sh]
    
    Loop --> Oferta
    BatchRunner --> Workers[Spawn N workers]
    Workers --> Oferta
    
    style Shared fill:#ff9800,color:#fff
    style Custom fill:#4caf50,color:#fff
    style Eval fill:#2196f3,color:#fff
```

---

## 10. Corporate Network Setup

```mermaid
graph LR
    Corp[Corporate Network<br/>SSL Inspection] --> Proxy[Proxy/Firewall]
    
    Proxy --> Block1{npm install?}
    Proxy --> Block2{node scan.mjs?}
    
    Block1 -->|❌ SELF_SIGNED_CERT| Fix1[Configure npm]
    Block2 -->|❌ SELF_SIGNED_CERT| Fix2[Configure Node.js]
    
    Fix1 --> Cert1[npm config set cafile<br/>~/.certs/combined-ca-bundle.pem]
    Fix2 --> Cert2[NODE_EXTRA_CA_CERTS=<br/>~/.certs/combined-ca-bundle.pem]
    
    Cert1 --> Alias[Shell alias or export]
    Cert2 --> Alias
    
    Alias --> Success[✅ All commands work]
    
    style Corp fill:#f44336,color:#fff
    style Success fill:#4caf50,color:#fff
    style Cert1 fill:#2196f3,color:#fff
    style Cert2 fill:#2196f3,color:#fff
```

---

## Legend

**Diagram Colors:**
- 🟦 Blue - System/Processing components
- 🟩 Green - Success states / User layer
- 🟧 Orange - Important components / System layer
- 🟥 Red - Error states / Warnings
- 🟪 Purple - Output files
- 🟨 Yellow - Data/State files

**Symbols:**
- 📋 - Data file
- 🤖 - AI Agent
- 👤 - User
- 📄 - Report
- 📑 - PDF
- 📊 - Tracker/TSV
- 🗂️ - Database/State
- ✅ - Success

---

## Usage Tips

1. **GitHub/GitLab:** These diagrams render automatically in markdown preview
2. **VS Code:** Install "Markdown Preview Mermaid Support" extension
3. **Export:** Use mermaid.live to convert to PNG/SVG for presentations
4. **Edit:** Modify diagram source directly in this file

## Related Documentation

- [SYSTEM-OVERVIEW.md](./SYSTEM-OVERVIEW.md) - Text-based architecture overview
- [ARCHITECTURE.md](./ARCHITECTURE.md) - Technical implementation details
- [DATA_CONTRACT.md](../DATA_CONTRACT.md) - File ownership rules
