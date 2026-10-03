# Diagramas do 3CP

Este diretório contém **diagramas técnicos** do protocolo 3CP em formato **Mermaid.js**. 
Copie o código de cada diagrama e cole em [Mermaid Live Editor](https://mermaid.live/) para visualizar.

---

## 📌 1. Arquitetura Geral do 3CP
**Descrição:** Visão macro de como o 3CP funciona, mostrando entidades e componentes.

```mermaid
flowchart TD
    subgraph Entidades
        A[Submitter] -->|1. Submeter ProvenanceEntry| B(Validator Node)
        B -->|2. Consenso BFT| C[Block Proposal]
        C -->|3. Ancorar Bloco| D[Sparse Merkle Tree]
        D -->|4. Publicar Bloco| E[Anchor Publisher]
        E -->|5. Verificar| F[Light Client]
        F -->|6. Auditar| G[Auditor]
    end
    
    subgraph Componentes
        H[Mandate] -->|Define Regras| D
        I[UID0 Identity] -->|Autentica| A
        J[ECVRF] -->|Eleição de Líder| B
        K[Dilithium3] -->|Assinaturas| C
        L[Kyber1024] -->|Handshake Seguro| B
        M[BLAKE3-256] -->|Hashing| D
    end
    
    style A fill:#f9f,stroke:#333
    style B fill:#bbf,stroke:#333
    style D fill:#9f9,stroke:#333
    style E fill:#ff9,stroke:#333
    style F fill:#9ff,stroke:#333
```

**Legenda:**
- **Submitter**: Quem submete dados (ex: um banco registrando uma transação).
- **Validator Node**: Nós que validam e assinam blocos (ex: Gleipnir).
- **Sparse Merkle Tree (SMT)**: Estrutura que armazena o estado da chain.
- **Anchor Publisher**: Publica blocos em IPFS/S3/filesystem.
- **Light Client**: Verifica blocos sem executar consenso.
- **Mandate**: Define quais eventos **devem** ser registrados.

---

## 📌 2. Fluxo de Consenso BFT (2 Fases)
**Descrição:** Como um bloco é produzido no 3CP v2.0, com eleição de líder via VRF e fases PREPARE/COMMIT.

```mermaid
sequenceDiagram
    participant Leader
    participant Validator1
    participant Validator2
    participant Validator3
    participant SMT
    participant Blockchain

    Note over Leader: Ciclo c
    Leader->>Validator1: 1. VRF Proof (gamma)
    Leader->>Validator2: 1. VRF Proof (gamma)
    Leader->>Validator3: 1. VRF Proof (gamma)
    Note over Leader: Líder = menor gamma

    Leader->>Validator1: 2. Proposta de Bloco B
    Leader->>Validator2: 2. Proposta de Bloco B
    Leader->>Validator3: 2. Proposta de Bloco B

    Validator1->>SMT: 3. Verifica StateRoot
    Validator2->>SMT: 3. Verifica StateRoot
    Validator3->>SMT: 3. Verifica StateRoot

    Validator1->>Leader: 4. PREPARE-SIG (H(B))
    Validator2->>Leader: 4. PREPARE-SIG (H(B))
    Validator3->>Leader: 4. PREPARE-SIG (H(B))

    Note over Leader: Quórum = ceil(2N/3)
    Leader->>Leader: 5. Coleta PREPARE-SIGs
    Leader->>Leader: 6. Constrói B_final (PrepareSigsBitmap + CommitSig)
    Leader->>Validator1: 7. Difunde B_final
    Leader->>Validator2: 7. Difunde B_final
    Leader->>Validator3: 7. Difunde B_final

    Validator1->>Blockchain: 8. Aceita B_final
    Validator2->>Blockchain: 8. Aceita B_final
    Validator3->>Blockchain: 8. Aceita B_final
```

**Detalhes:**
1. **Eleição de Líder**: Cada validador envia uma prova VRF (`gamma`). O líder é quem tem o menor `gamma`.
2. **Proposta**: Líder propõe um bloco `B` com entradas pendentes.
3. **Verificação**: Validadores verificam o `StateRoot` (raiz da SMT).
4. **PREPARE**: Validadores assinam `H(B)` e enviam para o líder.
5. **Quórum**: Líder espera `ceil(2N/3)` assinaturas.
6. **COMMIT**: Líder constrói `B_final` com `PrepareSigsBitmap` e `CommitSig`.
7. **Finalidade**: Bloco é **final e imutável**.

---

## 📌 3. Mandatory Event Anchoring (Inovação Principal)
**Descrição:** Como o 3CP detecta omissões de eventos obrigatórios.

```mermaid
flowchart TD
    subgraph Submissão
        A[Submitter] -->|1. Submeter Mandate| B[Validator Node]
        B -->|2. Ancorar Mandate| C[Bloco 10]
        C -->|3. Mandate Ativo| D[State: ActiveMandates]
    end
    
    subgraph Eventos
        E[Submitter] -->|4. Submeter Evento| F[Validator Node]
        F -->|5. Validar vs. Mandate| G{Evento é<br>obrigatório?}
        G -->|Sim| H[Ancorar no Bloco 11]
        G -->|Não| I[Descartar ou<br>Armazenar Temporariamente]
    end
    
    subgraph Auditoria
        J[Auditor] -->|6. Baixar Chain| K[Blocos 10-11]
        J -->|7. Baixar Mandates| L[Mandates Ativos]
        K -->|8. Comparar| M{Evento 11<br>existe?}
        M -->|Sim| N[✅ OK]
        M -->|Não| O[❌ Omissão Detectada!]
    end
    
    style O fill:#f99,stroke:#333
```

**Explicação:**
1. **Mandate**: Um `ProvenanceEntry` com `Label: "3cp:mandate:v1"` define que eventos do tipo `"release_gate"` **devem** ser registrados.
2. **Evento**: Um submissor tenta registrar um evento (ex: uma aprovação de release).
3. **Validação**: O validador verifica se o evento **deve** ser ancorado (segundo os Mandates ativos).
4. **Ancoragem**: Se for obrigatório, o evento é ancorado no bloco.
5. **Auditoria**: O auditor compara a chain com os Mandates ativos. Se um evento obrigatório **não estiver** na chain, **a omissão é detectada**.

---

## 📌 4. Sparse Merkle Tree (SMT)
**Descrição:** Como o estado do 3CP é comprometido em uma SMT com profundidade 256.

```mermaid
flowchart TD
    subgraph Folhas
        A[Folha: key=hash1] -->|Hash| B[hash1_data]
        C[Folha: key=hash2] -->|Hash| D[hash2_data]
        E[Folha: key=hash3] -->|Hash| F[hash3_data]
    end
    
    subgraph Nós Intermediários
        direction TB
        B -->|BLAKE3| G[Nó 1: H\\hash1_data\\]
        D -->|BLAKE3| H[Nó 2: H\\hash2_data\\]
        F -->|BLAKE3| I[Nó 3: H\\hash3_data\\]
        G -->|BLAKE3| J[Nó 4: H\\G + H\\]
        I -->|BLAKE3| J
    end
    
    subgraph Raiz
        J -->|BLAKE3| K[StateRoot: H(J + ...)]
    end
    
    style K fill:#9f9,stroke:#333
```

**Detalhes:**
- **Folhas**: Cada folha é um `key=hash` do evento + `value=hash` do dado.
  - `key = entry.Hash` (32 bytes, BLAKE3-256)
  - `value = entry.Hash` (32 bytes, BLAKE3-256)
- **Nós Intermediários**: Hashes de 32 bytes (BLAKE3-256) dos filhos.
- **Raiz (StateRoot)**: Hash final da SMT, **incluído em cada bloco**.
- **Prova SMT**: Um array de **256 hashes** (8KB) para verificar inclusão.

---

## 📌 5. Light Client Verification
**Descrição:** Como um auditor verifica um bloco sem executar consenso.

```mermaid
sequenceDiagram
    participant LightClient
    participant AnchorPublisher
    participant ValidatorSet

    LightClient->>AnchorPublisher: 1. GetBlock(index=10)
    AnchorPublisher-->>LightClient: 2. Retorna Bloco 10
    
    LightClient->>ValidatorSet: 3. GetValidatorSet(cycle=10)
    ValidatorSet-->>LightClient: 4. Retorna ValidatorSet (N=4)
    
    LightClient->>LightClient: 5. Verifica:
    LightClient->>LightClient: - PrepareSigsBitmap tem ≥ ceil(2N/3) bits
    LightClient->>LightClient: - PrepareSigs são válidas (Dilithium3)
    LightClient->>LightClient: - CommitSig é válida (Líder)
    LightClient->>LightClient: - PrevHash aponta para Bloco 9
    
    alt Verificação OK
        LightClient->>LightClient: 6. ✅ Bloco é válido
    else Verificação Falha
        LightClient->>LightClient: 6. ❌ Bloco é inválido
    end
```

**Passos:**
1. Obter o bloco do **Anchor Publisher** (IPFS, S3, filesystem).
2. Obter o **ValidatorSet** do ciclo do bloco.
3. Verificar **assinaturas** (quórum, líder, cadeia).

---

## 📌 6. Fluxo de Rotação de Chaves
**Descrição:** Como validadores trocam chaves sem downtime.

```mermaid
flowchart TD
    subgraph Ciclo Atual
        A[Validator] -->|1. Gera Nova Chave| B[NewDilithium3PK + NewVRFPK]
        B -->|2. Cria key-rotation-entry| C[key-rotation-entry]
        C -->|3. Assina com Chave Antiga| D[SignatureOld]
        C -->|4. Assina com Chave Nova| E[SignatureNew]
        D -->|5. Submeter Entrada| F[Bloco 10]
    end
    
    subgraph Overlap Period
        F -->|6. EffectiveCycle=15| G[Blocos 10-14: Chave Antiga]
        F -->|7. EffectiveCycle=15| H[Blocos 15-24: Ambas as Chaves]
        H -->|8. ExpiryCycle=25| I[Blocos 25+: Chave Nova]
    end
    
    style G fill:#f99,stroke:#333
    style H fill:#ff9,stroke:#333
    style I fill:#9f9,stroke:#333
```

**Estrutura da `key-rotation-entry`:**
- `NewPublicKey` (Dilithium3)
- `NewVRFPublicKey`
- `EffectiveCycle` (ciclo em que a nova chave passa a valer)
- `ExpiryCycle` (ciclo em que a antiga chave expira)
- `SignatureOld` (assinada com a chave antiga)
- `SignatureNew` (assinada com a nova chave)

---

## 📌 7. Exemplo de Mandate (YAML)
```yaml
mandate:
  id: "M-RELEASE-GATE-v1"  # BLAKE3-256 do payload
  authority: "root:security-team"  # RootID do autor
  version: 1
  valid_from: "2026-01-01T00:00:00Z"
  valid_until: "2026-12-31T23:59:59Z"  # 0 = nunca expira
  rules:
    - event_class: "release_gate"
      description: "Toda release com CVSS >= 7.0 exige aprovação"
      severity_min: 7.0
      severity_max: 10.0
      mandatory: true  # DEVE ser registrado
      required_fields: ["Approver", "Signature", "CVSS"]
      max_deferral_sec: 30  # Janela máxima de tolerância
```

---

## 📌 8. Exemplo de ProvenanceEntry (JSON)
```json
{
  "Hash": "62e3391cf9506246869a9a2828517c2dff1cf60c5c3d41798e693905cd4db509",
  "Submitter": "4e4f44453030312d2d2d2d2d2d2d2d2d",
  "Timestamp": 1784030400000000000,
  "Label": "release:gate",
  "Approver": "5e4f44453030322d2d2d2d2d2d2d2d2d",
  "Reference": "a1b2c3d4e5f6...",
  "Signature": "d1e2f3a4b5c6...",
  "MandateRef": "a1b2c3d4e5f678901234567890123456789012345678901234567890"
}
```

---

## 🔗 **Como Usar Estes Diagramas**
1. **Visualização Online:**
   - Acesse [Mermaid Live Editor](https://mermaid.live/).
   - Copie e cole o código Mermaid.
   - O diagrama será renderizado automaticamente.

2. **Integração em Documentos:**
   - **Markdown (GitHub, GitLab):** Use bloco de código ```mermaid.
   - **LaTeX:** Use o pacote `mermaid` ou converta para TikZ.
   - **HTML:** Inclua a biblioteca Mermaid.js e use a tag `<div class="mermaid">`.

3. **Ferramentas Locais:**
   - **VS Code:** Instale a extensão [Mermaid](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid).
   - **Obsidian:** Habilite o plugin Mermaid.

---

**Dica:** Para diagramas complexos, use o [Mermaid Live Editor](https://mermaid.live/) para ajustar o layout antes de incluir no whitepaper.
