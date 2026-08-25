# Plano de Migração de Diretórios — v1.0 → v2.0

> Extraído de `SPEC-3CP-V2.md` (era-of-authoring artefact — plano de tarefas para quem
> aplicou a migração v1→v2 nos diretórios do repositório). Não é normativo do protocolo;
> é registo de processo editorial, preservado aqui por rastreabilidade histórica.

# MODIFICAÇÕES NECESSÁRIAS NOS DIRETÓRIOS

## `spec/schemas/`

### Arquivos a MODIFICAR:

| Arquivo | Mudança |
|---------|---------|
| **`block.cddl`** | Adicionar campos v2.0 (keys 12-17): `ProtocolVersion`, `PrepareSigsBitmap`, `PrepareSigs`, `CommitSig`, `ExternalAnchors`, `KeyRotationEpoch`. Atualizar semântica de `Sigs` (key 8) para indicar que contém PREPARE signatures. |
| **`mandate.cddl`** | Adicionar `rule` para `EventClass: "3cp:consensus-config"` e `EventClass: "3cp:anchor-config"` como exemplos normativos. |

### Arquivos a CRIAR:

| Arquivo | Conteúdo |
|---------|----------|
| **`network-state.cddl`** | `network-state`, `node-state`, `validator-info` conforme Apêndice A.2. |
| **`key-rotation.cddl`** | `key-rotation-entry` conforme Apêndice A.3. |
| **`genesis.cddl`** | `genesis-block`, `genesis-validator-set`, `genesis-validator-info` conforme Apêndice A.4. |
| **`light-client.cddl`** | `light-client-request`, `light-client-response` conforme Apêndice A.5. |
| **`zk-bridge.cddl`** | `zk-bridge-request`, `hash-function` enum conforme Apêndice A.6. |

## `spec/notes/`

### Arquivos a MANTER (sem alteração):

| Arquivo | Motivo |
|---------|--------|
| **`mandatory-anchoring.md`** | Conceito de Mandates permanece válido na v2.0. |

### Arquivos a CRIAR:

| Arquivo | Conteúdo |
|---------|----------|
| **`bft-consensus.md`** | Nota explicativa sobre por que o consenso em duas fases com quórum `ceil(2N/3)` garante `f < N/3` tolerância a falhas bizantinas. Incluir diagrama de máquina de estados PREPARE/COMMIT. |
| **`key-rotation.md`** | Nota sobre o problema de rotação de chaves em sistemas de longa duração e como a v2.0 resolve via entradas de protocolo. |
| **`light-client-verification.md`** | Nota sobre o modelo de verificabilidade por terceiros e como um light client verifica blocos sem executar consenso. |
| **`migration-v1-v2.md`** | Guia para implementadores: procedimento de hard fork, exportação de estado v1.0, criação de genesis v2.0, `LegacyAnchor`. |

## `spec/examples/`

### Arquivos a MODIFICAR:

| Arquivo | Mudança |
|---------|---------|
| **`test-vectors.md`** | Atualizar para v2.0: novo formato de bloco com campos 12-17, vetores de `PrepareSigsBitmap`, `CommitSig`, exemplo de `key-rotation-entry`, exemplo de `genesis-block` com `GenesisValidatorSet`, vetores de verificação de quórum batch. |

### Arquivos a CRIAR:

| Arquivo | Conteúdo |
|---------|----------|
| **`test-vectors-key-rotation.md`** | Vetores de teste completos para uma rotação de chaves válida, incluindo `SignatureOld` e `SignatureNew` verificáveis. |
| **`test-vectors-light-client.md`** | Vetores de teste para um light client verificando um bloco v2.0: `GetValidatorSet`, verificação de `PrepareSigs`, `CommitSig`, e prova SMT. |

## `spec/` (raiz)

### Arquivos a MODIFICAR:

| Arquivo | Mudança |
|---------|---------|
| **`glossary.md`** | Adicionar termos: `PrepareSigsBitmap`, `CommitSig`, `KeyRotationEpoch`, `GenesisValidatorSet`, `Anchor Publisher`, `Light Client`, `ZKBridge`, `NetworkID`, `Mode Degraded`, `GraceCycles`. |
| **`3CP.md`** | Manter como referência histórica da v1.0. Adicionar nota no topo indicando que a versão normativa atual é a v2.0 (`SPEC-3CP-V2.md`). |

### Arquivos a CRIAR:

| Arquivo | Conteúdo |
|---------|----------|
| **`SPEC-3CP-V2.md`** | **Este documento.** A especificação normativa v2.0 completa. |

## `README.md` (raiz do repositório)

### Mudanças:

- Atualizar para indicar que a especificação normativa atual é a **v2.0**.
- Link direto para `spec/SPEC-3CP-V2.md`.
- Seção "Implementações de Referência" com links para os repositórios externos (`had-nu/gleipnir`, `had-nu/carcosa`), explicitando que não fazem parte deste repositório.
- Seção "Conformidade" indicando a suite de testes v2.0.

---

**Nota final:** Este documento é uma especificação de protocolo. Nenhum código de implementação, script de build, ferramenta CLI, ou artefato executável deve ser adicionado a este repositório. Implementações são desenvolvidas em repositórios separados sob suas próprias governanças.