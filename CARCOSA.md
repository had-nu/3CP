# CARCOSA — Camada de Auditoria com Prova Zero-Knowledge Acoplada ao 3CP

## Relação com o 3CP

Este repositório (`had-nu/3CP`) contém a **especificação do protocolo 3CP**.
Há dois artefatos que a implementam ou estendem:

| Artefato | Papel | O que faz |
|----------|-------|-----------|
| **Gleipnir** | Implementação de referência do 3CP | Go. Implementa o protocolo 3CP conforme a spec — consenso, SMT, gRPC, Dilithium3. A suíte de 33 testes valida conformidade na fronteira gRPC |
| **CARCOSA** | Framework de auditoria ZK **sobre** o 3CP | Rust. **Não implementa o 3CP.** Consome a camada de ancoragem do Gleipnir (via gRPC) para adicionar provas de conhecimento-zero (STARK/Winterfell) a fluxos de auditoria. É uma camada *acoplada* ao 3CP, não uma implementação alternativa |

Em suma:

- **3CP** define *como* ancorar evidências com integridade verificável por terceiros
- **Gleipnir** realiza o 3CP — é a rede que aceita submissões, forma consenso e produz blocos
- **CARCOSA** usa o Gleipnir como *camada de âncora* para selar provas ZK na chain, mantendo a privacidade dos aprovadores sem abrir mão da detecção de omissão

## 1. Objetivo

Demonstrar que **3CP + Mandato + ZK** resolve o trilema auditoria vs privacidade vs deteccao de omissao:

- O **Mandato 3CP** define *o que* deve ser provado (torna omissao detectavel)
- A **STARK (Winterfell)** prova *que* foi aprovado sem revelar *quem* aprovou (privacidade)
- O **Hash de 32B** ancorado no SMT vincula a prova a chain sem expor conteudo (escalabilidade)

---

## 2. Arquitetura

```
+----------------------------------------------------------------------------+
|                         DOMINIO AUDITADO                                   |
|                                                                            |
|  +----------+    +--------------------+     +---------------------------+  |
|  | Policy   |--->|  STARK Prover      |     |  Compliance Gaps          |  |
|  | YAML     |    |  (Winterfell/Rust) |     |  (events missing from     |  |
|  |          |    |                    |     |   chain despite mandate)  |  |
|  | min: 2/3 |    |  Inputs:           |     +---------------------------+  |
|  | CVSS>7.0 |    |  +- 3 pub keys     |              |                    |
|  +----------+    |  +- 2 signatures   |              |                    |
|       |          |  +- approval data  |              |                    |
|       v          |  +- policy hash    |              |                    |
|  +----------+    |                    |              |                    |
|  | Mandate  |    |  Prova (STARK):    |     +--------+--------+          |
|  | (assinado|    |  ">2/3 aprovadores |     |  Compliance     |          |
|  | ancorado)|    |   autorizados      |     |  Verifier       |          |
|  +----------+    |   aprovaram sem    |     |  (auditor tool) |          |
|       |          |   revelar quem"    |     |                 |          |
|       |          +---------+----------+     | 1. Le mandates  |          |
|       |                   |                 | 2. Varre chain  |          |
|       |                   | proof.bin       | 3. Verifica ZK  |          |
|       |                   v                 | 4. Report gaps  |          |
|       |          +------------------+       +-----------------+          |
|       |          |  Blob Store (FS) |              |                     |
|       |          |  + <digest>/     |              |                     |
|       |          |      proof.bin   |              |                     |
|       |          |      pub.json    |              |                     |
|       |          |      metadata.jsn|              |                     |
|       |          +------------------+              |                     |
|       |                                            |                     |
|       | digest = BLAKE3(proof) = 32B               |                     |
|       v                                            |                     |
|  +----------------------------------------+        |                     |
|  |       Gleipnir (Rede 3CP)              |        |                     |
|  |                                        |        |                     |
|  |  ProvenanceEntry {                     |        |                     |
|  |    Hash:        digest (32B) +---------+--------+-- SMT proof --------+|
|  |    Submitter:   "zk-poc-1"             |        |                     |
|  |    Timestamp:   T_unixnano             |        |                     |
|  |    Label:       "zk:2of3"              |        |                     |
|  |    MandateRef:  hash_do_mandato (32B)  |        |                     |
|  |  }                                      |        |                     |
|  |                                        |        |                     |
|  |  SMT: key=digest, value=digest         |        |                     |
|  |  Block: finalidade instantanea (3s)    |        |                     |
|  +----------------------------------------+        |                     |
|                       =============================+                     |
|                     TUDO VERIFICAVEL POR TERCEIROS                        |
+----------------------------------------------------------------------------+
```

---

## 3. Componentes

### 3.1 Mandato 3CP (ancora da obrigacao)

```yaml
mandate:
  id: "M-RELEASE-GATE-ZK-v1"
  authority: "root:security-team"
  version: 1
  valid_from: "2026-01-01T00:00:00Z"
  valid_until: "2026-12-31T23:59:59Z"
  policy_hash: "a1b2c3..."   # BLAKE3 da policy YAML
  rules:
    - event_class: "release_gate"
      description: "Toda release com CVSS >= 7.0 exige prova ZK de aprovacao 2-de-3"
      severity_min: 7.0
      severity_max: 10.0
      mandatory: true
      required_fields: ["Signature"]
      label_prefix: "zk:2of3"
      max_deferral_sec: 30
```

Ancorado via `SubmitMandate` -> `ProvenanceEntry{Label: "3cp:mandate:v1"}`.

### 3.2 STARK Prover (Winterfell / Rust)

**O que prova**: Conhecimento de 2 assinaturas validas de 3 aprovadores autorizados, sem revelar quais.

**AIR design (trace de 3 linhas + colunas de transicao)**:

| Coluna | Descricao |
|--------|-----------|
| `pubkey_hash_i` | BLAKE3(pubKey_i) --- 32B |
| `commitment_i` | BLAKE3(approvalData || pubKey_i) |
| `is_authorized_i` | 1 se pubKey_i esta na lista do mandato |
| `has_valid_sig_i` | 1 se assinatura i e valida para approvalData |
| `cumulative` | Soma de (is_authorized AND has_valid_sig) |

**Restricoes de transicao**:
- `cumulative[t+1] in {cumulative[t], cumulative[t]+1}` (so incrementa)
- `cumulative[t+1] == cumulative[t] + (is_authorized[t] AND has_valid_sig[t])`
- `cumulative[final] >= threshold` (boundary constraint)

**Parametros Winterfell (estimados)**:
| Parametro | Valor | Razao |
|-----------|-------|-------|
| Security level | 100 bits | Conjectural pos-quantico |
| Extension field | QuadraticExtension | Para constraint divisor |
| FRI folding factor | 4 | Balanco proof size / proving time |
| Proof size | ~70 KB | Sem setup, STARK puro |
| Proving time | ~5-15s | Laptop single-thread |
| Verification | ~20-50ms | Constraint evaluation |

**Binario CLI**:

```
zkp prove \
  --approver-keys  keys/pk1.bin,keys/pk2.bin,keys/pk3.bin \
  --signatures     sigs/sig1.bin,sigs/sig2.bin \
  --approval-data  approval.json \
  --policy-hash    a1b2c3... \
  --threshold      2 \
  --output         blobs/
```

### 3.3 Blob Store

```
carcosa/output/blobs/
+-- index.toml
+-- <blake3_hex_do_proof>/
    +-- proof.bin          # STARK proof serializado (~70KB)
    +-- pub_inputs.json    # { policy_hash, threshold, approval_data_hash, commitments[] }
    +-- metadata.json      # { release_id, timestamp, approver_count, mandate_ref }
    +-- commitments.json   # commitments publicos (para verificacao independente)
```

Nomeado por `BLAKE3(proof.bin)`. O mesmo valor e o `Hash` no `ProvenanceEntry`.

### 3.4 Integracao Gleipnir (gRPC)

O Gleipnir nao precisa de modificacoes. Usa API existente:

```
grpc_cli call localhost:50051 SubmitHash "
  hash: $(cat blobs/<digest>/proof.bin | blake3sum)
  submitter: \"zk-poc-1\"
  timestamp: $(date +%s%N)
  label: \"zk:2of3\"
  mandate_ref: $(cat mandate_hash.hex)
"
```

### 3.5 Compliance Verifier (auditor)

```
carcosa verify \
  --block-idx      42 \
  --proof-digest   a1b2c3... \
  --blob-store     output/blobs/ \
  --gleipnir-addr  localhost:50051

Output esperado:
+-- 3CP SMT inclusion:     [OK] VERIFIED (StateRoot match)
+-- STARK proof validity:  [OK] VERIFIED (cumulative >= 2)
+-- Mandate compliance:    [OK] VERIFIED (entry exists, fields ok)
+-- Overall:               COMPLIANT
```

---

## 4. Fluxo Completo

```
 T0 --- POLICY DEFINITION
        policy.yaml -> BLAKE3 -> policy_hash
        mandato.yaml -> SubmitMandate -> ancorado na chain

 T1 --- EVENT TRIGGER
        release com CVSS 8.5 -> matching rule (severity_min: 7.0)

 T2 --- APPROVAL FLOW
        3 aprovadores listados na politica
        2 assinam (Dilithium3) o approval data
        3o pode estar ausente (nao bloqueante)

 T3 --- STARK PROVING
carcosa prove \
          --approver-keys  pk1,pk2,pk3 \
          --signatures     sig1,sig2 \
          --approval-data  release-42.json \
          --policy-hash    a1b2c3 \
          --threshold      2 \
          --output         blobs/

        Output: blobs/<digest>/proof.bin + pub_inputs.json

 T4 --- 3CP ANCHORING
        digest = BLAKE3(proof.bin)
        grpc SubmitHash {
          hash:        digest,
          label:       "zk:2of3",
          mandate_ref: mandate_hash
        }
        -> Ticket -> WaitForAnchor -> SMT proof

 T5 --- AUDIT VERIFICATION (a qualquer momento)
        zkp verify --block-idx 42 --proof-digest <digest>

        +-- Fetch block do Gleipnir
        +-- Extrair ProvenanceEntry pelo digest
        +-- Load proof.bin do blob store
        +-- Verificar STARK (Winterfell verifier)
        +-- Verificar SMT inclusion (BLAKE3 leaf -> root -> StateRoot)
        +-- Report: [OK] COMPLIANT

 T6 --- CENARIO DE GAP (OMISSAO)
        Se a organizacao NAO ancorou a prova:

        carcosa audit --window T0..T1 --mandate M-001

        +-- GetActiveMandates(T0..T1) -> [M-001]
        +-- Varre chain por entries com Label: "zk:2of3" e Timestamp in [T0,T1]
        +-- Detecta: release_gate CVSS 8.5 @ T1.5 -> nenhuma entry
        +-- Report:
            [NO] COMPLIANCE GAP
            Mandato: M-001 (release_gate, CVSS >= 7.0)
            Tipo:    missing
            Evento:  release-42 @ 2026-03-15T14:30:00Z
            Status:  NAO ANCORADO -- omissao detectada
```

---

## 5. O que a PoC Demonstra (Proposicoes de Valor)

| # | Proposicao | Mecanismo |
|---|-----------|-----------|
| P1 | **Privacidade dos aprovadores** | STARK nao revela quais 2-de-3 assinaram. A chain so tem o hash da prova |
| P2 | **Deteccao de omissao** | Mandato ancorado. Se a prova nao existe, o gap e detectavel por qualquer terceiro |
| P3 | **Verificacao independente** | Auditor nao precisa de acesso interno -- chain publica + blob store bastam |
| P4 | **Pos-quantico** | Dilithium3 (chain) + STARK+BLAKE3 (ZK) = stack pos-quantica |
| P5 | **Non-repudiation legal** | Assinaturas Dilithium3 off-chain para disputas; STARK prova suficiencia |
| P6 | **Baixo custo de chain** | Apenas 32 bytes por evento no SMT (~3.2M eventos por GB de chain) |
| P7 | **Mandate lifecycle versionado** | Politica evolui com audit trail completo (PrevVersion, Supersedes) |

### 5.1 Matriz de Comparacao

| Capacidade | Log tradicional | ZK puro | 3CP puro | 3CP + ZK (PoC) |
|------------|----------------|---------|----------|-----------------|
| Integridade | [NO] Mutavel | [OK] | [OK] | [OK] |
| Privacidade do aprovador | [NO] Expoe | [OK] | [NO] Hash publico | [OK] |
| Deteccao de omissao | [NO] Indistinguivel | [NO] Provador escolhe | [OK] (Mandato) | [OK] (Mandato) |
| Verificacao por terceiros | [NO] Requer acesso | [OK] | [OK] | [OK] |
| Non-repudiation | [NO] | [OK] (prova) | [OK] (assinatura) | [OK] (ambos) |
| Pos-quantico | [NO] | [OK] (STARK) | [OK] (Dilithium3) | [OK] (stack) |
| Custo por evidencia | Alto (armazenar tudo) | Medio (prova ~70KB) | Minimo (32B hash) | Minimo (hash) + blob |

---

## 6. Estrutura de Diretorios

```
carcosa/
+-- air/
|   +-- Cargo.toml                     # workspace member: winterfell + deps
|   +-- src/
|   |   +-- lib.rs                     # AIR definition + prover + verifier
|   |   +-- approval_air.rs            # ConstraintDefinition + PublicInputs
|   |   +-- trace.rs                   # Trace generation (3 rows x N cols)
|   +-- tests/
|       +-- integration.rs             # Prove + verify roundtrip
|
+-- cli/
|   +-- Cargo.toml
|   +-- src/
|       +-- main.rs                    # CLI dispatcher
|       +-- commands/
|       |   +-- mod.rs
|       |   +-- prove.rs               # zkp prove
|       |   +-- verify.rs              # zkp verify (STARK + 3CP)
|       |   +-- anchor.rs              # gRPC -> Gleipnir
|       |   +-- audit.rs               # compliance gap detection
|       |   +-- setup.rs               # gerar Winterfell params
|       +-- gleipnir/
|           +-- mod.rs                 # gRPC client stubs
|           +-- client.rs              # SubmitHash, WaitForAnchor, GetBlock
|
+-- proto/
|   +-- gleipnir.proto                 # copiado do Gleipnir (para geracao de stub)
|
+-- scripts/
|   +-- demo-full.sh                   # fluxo completo automatizado
|   +-- demo-gap.sh                    # cenario de omissao (gap detection)
|   +-- generate-test-keys.sh          # gera 3 pares Dilithium3 + VRF
|
+-- test-data/
|   +-- policy.yaml                    # politica externa (severity, approvers)
|   +-- mandato.yaml                   # Mandato 3CP correspondente
|   +-- approver-keys/
|   |   +-- pk1.bin, pk2.bin, pk3.bin  # chaves publicas (1952B cada)
|   |   +-- sk1.bin, sk2.bin, sk3.bin  # chaves privadas (armazenamento seguro)
|   |   +-- vrf-pk1.bin, ...           # chaves VRF para identidade 3CP
|   +-- approval-data/
|   |   +-- release-42.json            # approval legitimo (2 de 3 assinam)
|   |   +-- release-43-omitted.json    # release que deveria ser ancorada mas nao foi
|   +-- signatures/
|       +-- sig1.bin, sig2.bin         # assinaturas Dilithium3 (2700B cada)
|       +-- sig3-missing.bin           # placeholder (assinatura ausente)
|
+-- output/
|   +-- blobs/                         # gerado pelo prove
|       +-- index.toml                 # indice de provas ancoradas
|       +-- <digest>/
|           +-- proof.bin
|           +-- pub_inputs.json
|           +-- metadata.json
|           +-- commitments.json
|
+-- docker/
|   +-- docker-compose.yml             # Gleipnir 5 nos + PoC container
|   +-- Dockerfile                     # builder para o PoC
|
+-- docs/
|   +-- ARCHITECTURE.md                # este plano
|   +-- AIR-SPEC.md                    # especificacao detalhada do AIR
|   +-- INTEGRATION.md                 # como conectar ao Gleipnir
|   +-- TEST-VECTORS.md                # test vectors do fluxo
|
+-- Makefile
+-- Cargo.toml                         # workspace root
+-- README.md
+-- .gitignore
```

---

## 7. Roteiro de Implementacao

| Fase | Passo | Dependencia | Esforco (estimado) |
|------|-------|-------------|-------------------|
| **F1** | AIR constraint definition (Winterfell) | Nenhuma | 3-5 dias |
| **F2** | Trace generation + prover roundtrip | F1 | 2-3 dias |
| **F3** | CLI: `prove`, `verify` (STARK only) | F2 | 2 dias |
| **F4** | gRPC client -> Gleipnir (SubmitHash) | Gleipnir rodando | 1 dia |
| **F5** | CLI: `anchor` (prove -> blob -> gRPC) | F3 + F4 | 1 dia |
| **F6** | CLI: `audit` (gap detection) | F5 | 2 dias |
| **F7** | Docker Compose (5 nos + PoC) | Gleipnir Docker | 1 dia |
| **F8** | Scripts de demo (full + gap) | F7 | 1 dia |
| **F9** | Test vectors + integracao | F8 | 1 dia |
| **F10** | Documentacao final | F9 | 1 dia |

**Total estimado**: ~15-19 dias uteis (~3-4 semanas) para um desenvolvedor Rust familiarizado com Winterfell.

---

## 8. Riscos e Mitigacoes

| Risco | Impacto | Mitigacao |
|-------|---------|-----------|
| **Winterfell AIR complexo para iniciantes** | Atraso F1-F2 | Usar template de AIR multi-column; circuitos auxiliares de BLAKE3 ja existem no Winterfell |
| **BLAKE3 dentro do AIR e caro (muitos ciclos)** | Proving time alto (~minutos) | Usar primitive hash mais simples para o AIR (Poseidon ou Rescue) e BLAKE3 apenas para o digest externo. Ou usar a `DefaultHash` do Winterfell (Blake3 ja otimizado) |
| **Gleipnir gRPC sem StreamBlocks** | `audit` precisa varredura manual | Usar `GetBlock` sequencial -- lento mas funcional para PoC |
| **Gleipnir sem blob store** | Prova armazenada em FS local | PoC usa FS. Em producao: IPFS / S3 / URL no `PolicyURI` |
| **Chaves de teste Dilithium3** | Gleipnir pode nao aceitar chaves externas | Usar o `provectl init` do Gleipnir para gerar UIDs compativeis |
| **Prova STARK ~70KB** | Payload razoavel para blob, nao para chain | Apenas hash de 32B vai para o SMT -- proof fica no blob store |

---

## 9. Entregaveis da PoC

| Entregavel | Formato | Descricao |
|------------|---------|-----------|
| Documento de arquitetura | `docs/ARCHITECTURE.md` | Este plano consolidado |
| AIR spec | `docs/AIR-SPEC.md` | Definicao das constraints, colunas do trace, boundary conditions |
| Codigo fonte | Rust crate `air/` + `cli/` | Prover + Verifier + CLI |
| Docker Compose | `docker/docker-compose.yml` | Gleipnir 5 nos + PoC |
| Scripts de demonstracao | `scripts/demo-*.sh` | `demo-full.sh` (fluxo completo), `demo-gap.sh` (omissao) |
| Test vectors | `docs/TEST-VECTORS.md` | Inputs, outputs, hashes esperados do fluxo |
| README | `README.md` | Instrucoes de setup, execucao, verificacao |
