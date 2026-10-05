# SPEC-3CP-V2.md

**Título:** 3CP Protocol — Especificação Normativa v2.0  
**Subtítulo:** Cryptographic Chain-of-Custody Protocol com Consenso BFT, Governança de Longo Prazo e Verificabilidade por Terceiros  
**Status:** NORMATIVA  
**Versão:** 2.0.0  
**Data:** 2026-07-28  
**Base:** 3CP Protocol v1.0 (spec/3CP.md)  
**Escopo:** Especificação de protocolo abstrato. Agnóstica a linguagem de implementação, sistema operacional, biblioteca criptográfica ou framework de execução.  
**Relação com Implementações:** O protocolo 3CP é implementável por qualquer parte. As implementações de referência conhecidas — Gleipnir (Go) e CARCOSA (Rust) — residem em repositórios separados e **não fazem parte desta especificação**. Esta norma deve ser suficiente para que uma implementação independente, sem acesso ao código-fonte de referência, alcance interoperabilidade completa.

---

## 1. Escopo, Princípios e Linguagem Normativa

As palavras-chave **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, e **OPTIONAL** neste documento devem ser interpretadas conforme [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

**Princípios diretores do 3CP v2.0:**
1. **Responsabilização sem boa-fé:** A conformidade não pode depender da boa-fé da entidade auditada.
2. **Verificabilidade independente:** Qualquer parte com acesso à cadeia pública deve poder verificar integridade, quórum e genealogia sem confiar no operador da rede.
3. **Finalidade por ciclo:** Um ciclo de consenso produz zero ou um bloco final. Blocos finais são imutáveis e irreversíveis.
4. **Governança on-chain:** Regras de operação da rede (quórum, latência, publicação) são declaradas em Mandates ancorados na própria cadeia.

---

## 2. Changelog: Diferenças da v1.0 para a v2.0

| # | Mudança | v1.0 | v2.0 |
|---|---------|------|------|
| 1 | **Consenso** | Eleição VRF + co-assinatura M-de-N sem fases definidas | Duas fases atômicas (PREPARE/COMMIT) com quórum fixo `ceil(2N/3)` |
| 2 | **ValidatorSet** | Estado de rede continha apenas `UID` e `Status` | Estado de rede obrigatoriamente contém `Dilithium3PK` e `VRFPK` de todos os validadores |
| 3 | **Rotação de chaves** | Não especificada (out-of-band) | Operação de protocolo via `3cp:key-rotation:v1` |
| 4 | **Formato de bloco** | Campos v1.0 | Campos v2.0 adicionais: `ProtocolVersion`, `PrepareSigsBitmap`, `PrepareSigs`, `CommitSig`, `ExternalAnchors`, `KeyRotationEpoch`, `LegacyAnchor`, `Metadata` |
| 5 | **Ciclo** | Fixo em 3s, entradas descartadas em timeout | Adaptativo (`BaseInterval + EWMA(RTT)`), entradas retidas em aborto |
| 6 | **Laplaciano λ₁** | Recomputação completa a cada `LambdaInterval` | Atualização incremental (rank-one update) quando topologia não muda |
| 7 | **Verificabilidade** | Teórica ("cadeia pública") | Protocolo de Light Client e Anchor Publishers obrigatórios |
| 8 | **ZK** | Menção conceitual ao CARCOSA | Interface `ZKBridge` versionada (v1.0.0) especificada como contrato de protocolo |
| 9 | **Identidade** | `FEntropy` de 32 bytes, derivação não normatizada | Derivação via `HKDF-SHA256(salt=NetworkID, ...)` com entropia mínima de 128 bits |
| 10 | **Conformidade** | 33 testes operacionais | Suite estendida com testes adversariais BFT (12 casos mínimos) |

---

## 3. Modelo de Sistema e Entidades

### 3.1 Entidades

- **Submitter:** Entidade que submete uma `ProvenanceEntry` para ancoragem. Não precisa ser um validador.
- **Validator:** Entidade que participa do consenso, produzindo e co-assinando blocos. Cada validador possui um par de chaves Dilithium3 e um par de chaves VRF.
- **Anchor Publisher:** Entidade (validador ou serviço externo) que publica blocos finais em storage de leitura pública.
- **Light Client:** Entidade que verifica blocos e provas SMT sem participar do consenso e sem manter estado completo.
- **Auditor:** Entidade que verifica conformidade de Mandates contra a cadeia de entradas ancoradas.

### 3.2 Modelo de Ameaças

- **A1 — Adversário de rede:** Observa, atrasa, descarta ou reordena mensagens. Não controla chaves de validadores.
- **A2 — Operador comprometido:** Controla infraestrutura de um ou mais nós, mas não a maioria do quórum (`< ceil(2N/3)`).
- **A3 — Submissor comprometido:** Controla credenciais de submissão. Pode submeter entradas falsas, mas não pode forjar assinaturas de validadores.
- **A4 — Adversário externo:** Sem acesso a chaves. Pode tentar DoS ou replay.
- **A5 — Adversário quântico:** Possui computador quântico capaz de quebrar criptografia clássica. O 3CP v2.0 usa primitivas pós-quânticas (Dilithium3, ML-KEM) para resistir a A5.

---

## 4. Primitivas Criptográficas

### 4.1 Funções de Hash

| Função | Saída | Uso |
|--------|-------|-----|
| `BLAKE3-256` | 32 bytes | Hash de entradas, folhas SMT, nós SMT, identificadores de bloco |
| `SHA-256` | 32 bytes | `BlockHash` (identificador de bloco) |
| `HKDF-SHA256` | Variável | Derivação de chaves, seeds UID0 |

### 4.2 Assinaturas

**Algoritmo:** ML-DSA-65 (Dilithium3), conforme FIPS 204.

- Tamanho da chave pública: 1.952 bytes
- Tamanho da chave privada: 4.032 bytes
- Tamanho da assinatura: 3.309 bytes
- Nível de segurança NIST: 3 (equivalente a AES-192)

**[MUST]** Toda assinatura no protocolo 3CP usa Dilithium3, salvo indicação explícita em contrário.

### 4.3 VRF (Verifiable Random Function)

**Algoritmo:** ECVRF-EDWARDS25519-SHA512-Elligator2, conforme RFC 9381.

- Curva: Ristretto255 (grupo de ordem prima, sem cofator)
- Hash interna: SHA-512
- Tamanho da chave pública: 32 bytes
- Tamanho da chave privada: 32 bytes
- Tamanho da prova: 96 bytes (`Gamma || C || S`)
- Tamanho da saída `Gamma`: 32 bytes

**[MUST]** A derivação do nonce na prova Schnorr deve ser determinística via HMAC-SHA512 com a chave privada VRF como chave HMAC.

### 4.4 KEM (Key Encapsulation Mechanism)

**Algoritmo:** ML-KEM-1024 (Kyber1024), conforme FIPS 203.

- Tamanho da chave pública: 1.568 bytes
- Tamanho do ciphertext: 1.568 bytes
- Tamanho do shared secret: 32 bytes

**[MUST]** Usado exclusivamente para handshake de transporte seguro entre pares (Kyber encapsulate/decapsulate + AEAD).

### 4.5 AEAD

**Algoritmo:** ChaCha20-Poly1305 (RFC 8439).

**[MUST]** Usado para criptografia de transporte após handshake Kyber. A chave de 32 bytes é derivada via HKDF-SHA256 a partir do shared secret Kyber.

---

## 5. Formato de Bloco v2.0

### 5.1 Codificação

**[MUST]** Todos os blocos e entradas são serializados em CBOR (RFC 8949) canônico, conforme seção 4.2.1 da especificação CBOR.

### 5.2 Estrutura do Bloco

```
Block {
    ;; Campos v1.0 (mantidos, semântica inalterada)
    0  => uint64,                    ; Index
    1  => bytes .size 32,            ; PrevHash (BLAKE3-256)
    2  => bytes .size 32,            ; StateRoot (SMT root, BLAKE3-256)
    3  => bytes .size 16,            ; Proposer (RootID do líder)
    4  => [* bytes],                 ; Triad (reservado)
    5  => [* provenance-entry],       ; Anchored entries
    6  => float64,                    ; Lambda1 (autovalor de Fiedler)
    7  => int64,                     ; Timestamp (UnixNano)
    8  => null,                      ; RESERVED em ProtocolVersion == 2 (era Sigs na v1.0)
    9  => [* validator-info],        ; Validators (conjunto canônico; ver §7.1)
    10 => quorum-config,              ; Quorum
    11 => bytes .size 32,            ; BlockHash (SHA-256)

    ;; Campos v2.0 (novos, MUST em redes v2)
    12 => uint16,                     ; ProtocolVersion (default: 2)
    13 => bytes,                      ; PrepareSigsBitmap (bitfield, N bits)
    14 => [* bytes .size 3309],       ; PrepareSigs (apenas signatários ativos)
    15 => bytes .size 3309,           ; CommitSig (assinatura do líder sobre B_final)
    16 => [* tstr],                   ; ExternalAnchors (URIs/CIDs de publicação)
    17 => uint64,                     ; KeyRotationEpoch (ciclo de referência para chaves)
}
```

### 5.3 Semântica dos Campos de Assinatura v2.0

**[MUST]** A key 8 é `RESERVED` em `ProtocolVersion == 2` e MUST NOT carregar assinaturas. Na v1.0 ela era `Sigs`, com assinaturas de validadores sem distinção de fase; na v2.0 a distinção de fase passa a ser obrigatória e as assinaturas PREPARE têm endereço próprio (key 14). A posição 8 é preservada para que a numeração de chaves permaneça estável entre as duas versões.

**[MUST]** `PrepareSigsPayload` (key 14) é o **único** campo de assinaturas PREPARE em `ProtocolVersion == 2`. Um verificador MUST NOT aceitar assinaturas PREPARE de nenhuma outra chave.

**[MUST]** `PrepareSigsBitmap` é um bitfield onde o bit `i` é `1` se e somente se o validador de índice `i` no array `Validators` assinou PREPARE.

**[MUST]** `PrepareSigsPayload` (key 14) contém apenas as assinaturas dos validadores cujo bit em `PrepareSigsBitmap` é `1`, na ordem crescente de índice.

**[MUST]** `CommitSig` (key 15) é a assinatura Dilithium3 do líder sobre o hash do bloco final `B_final` (incluindo `PrepareSigsBitmap` e `PrepareSigsPayload`).

### 5.4 Cálculo do BlockHash

**[MUST]** O `BlockHash` é computado como:

```
BlockHash = SHA-256(
    LE64(Index)
    || PrevHash
    || StateRoot
    || Proposer
    || HashOfAnchoredEntries   ;; BLAKE3-256 do array canônico CBOR de entradas
    || LE64(Timestamp)
    || QuorumConfigCanonical
)
```

**[MUST]** `HashOfAnchoredEntries` é o hash BLAKE3-256 da serialização canônica CBOR do array em key 5 (`Anchored`). Isso permite verificar a integridade das entradas sem reter o bloco completo.

---

## 6. Consenso BFT em Duas Fases

### 6.1 Ciclo de Consenso

Um **ciclo** é a unidade atômica de tempo do protocolo. Cada ciclo `c` produz zero ou um bloco final de índice `c`.

**[MUST]** O ciclo é dividido em duas fases atômicas: PREPARE e COMMIT.

### 6.2 Fase PREPARE

**Entrada:** Estado de rede no início do ciclo `c`, conjunto de entradas pendentes `E`, e o conjunto de validadores `V`.

**Passo 1 — Eleição de líder:**
```
alpha_c = c || StateRoot_at_cycle_start
```

Para cada validador `v_i ∈ V`:
```
proof_i = VRF_Sign(sk_i^VRF, alpha_c)
gamma_i = VRF_ProofToHash(proof_i)
```

O líder é o validador com menor `gamma_i` (comparação lexicográfica de 32 bytes). Em caso de empate (extremamente improvável com Ristretto255), o empate é quebrado pelo menor `ValidatorID` lexicográfico.

**[MUST]** O `StateRoot_at_cycle_start` é a raiz da SMT **antes** da inserção de quaisquer entradas do ciclo `c`. A `alpha` é fixa no início do ciclo e não muda durante as fases.

**Passo 2 — Proposta:**
O líder propõe um bloco candidato `B` contendo:
- `Index = c`
- `PrevHash = H(B_{c-1})`
- `StateRoot = Root(SMT_after_inserting_E)`
- `Proposer = leader.ValidatorID`
- `Anchored = E`
- `Lambda1 = λ₁` (se computado neste ciclo)
- `Timestamp = now()`
- `Validators = [v_0.pk, v_1.pk, ..., v_{N-1}.pk]`
- `Quorum = {TotalValidators: N, RequiredSigs: ceil(2N/3)}`

**Passo 3 — Votação PREPARE:**
Cada validador `v_j` recebe `B` e verifica:
1. `B.PrevHash == H(B_{c-1})` aceito localmente.
2. `B.StateRoot == Root(SMT_after_inserting_E)` computado localmente.
3. Cada `e ∈ E` passa em `validateEntry(e)`.
4. `B.Lambda1 >= MinLambda1` (se aplicável).
5. `B.ProtocolVersion == 2` (em redes v2).

Se todas as verificações passam, `v_j` assina `H(B)` e difunde `PREPARE-SIG_j`.

### 6.3 Fase COMMIT

**Passo 4 — Coleta de quórum:**
O líder coleta `PREPARE-SIG_j` até atingir `Q = ceil(2N/3)`.

**[MUST]** Se o líder não coletar `Q` assinaturas dentro de `CycleTimeout`, o ciclo é **abortado**. As entradas `E` permanecem pendentes. O ciclo `c` não produz bloco.

**Passo 5 — Construção do bloco final:**
```
B_final = B || PrepareSigsBitmap || PrepareSigsPayload || CommitSig_leader
```

Onde `CommitSig_leader = Dilithium3_Sign(sk_leader, H(B_final))`.

**Passo 6 — Difusão e validação:**
O líder difunde `B_final`. Cada validador `v_j` verifica:
1. `PrepareSigsBitmap` contém pelo menos `Q` bits `1`.
2. Cada assinatura em `PrepareSigsPayload` verifica contra a chave pública do validador correspondente em `Validators`.
3. `CommitSig_leader` verifica contra `B.Proposer`.

Se todas as verificações passam, `v_j` aceita `B_final` como bloco final do ciclo `c`, apenda à cadeia local, e transita o estado de rede.

### 6.4 Ciclo Abortado

**[MUST]** Se um ciclo é abortado (timeout de PREPARE, quórum insuficiente, ou `λ₁ < MinLambda1`):
- Nenhum bloco é apendido.
- As entradas `E` permanecem na fila de pendentes.
- O ciclo `c+1` inicia com nova eleição VRF usando `alpha_{c+1} = (c+1) || H(B_{c-1})`.

### 6.5 Modo Degraded

**[MUST]** Se `N < 4`, o protocolo opera em modo `degraded`:
- `Q = 1` (qualquer assinatura única é suficiente).
- Cada bloco deve conter `Metadata["3cp:degraded-block"] = true`.
- A saída do modo `degraded` exige `N >= 4` e `GraceCycles` consecutivos (default: 10) com quórum normal.

---

## 7. Estado de Rede e ValidatorSet

### 7.1 GenesisValidatorSet

**[MUST]** O bloco genesis (índice 0) contém um `GenesisValidatorSet`:

```
GenesisValidatorSet = [* validator-info]

validator-info = {
    0 => bytes .size 16,    ; ValidatorID (RootID)
    1 => bytes .size 1952,   ; Dilithium3PK
    2 => bytes .size 32,    ; VRFPK
    3 => bytes .size 32,    ; ContractHash (opcional)
}
```

**[MUST]** O `GenesisValidatorSet` é imutável. Alterações no conjunto de validadores (adição, remoção, rotação de chaves) ocorrem via entradas de protocolo subsequentes.

### 7.2 NodeState

**[MUST]** O estado de rede (`NetworkState`) mantém, para cada nó conhecido:

```
NodeState = {
    0 => bytes .size 16,    ; UID (RootID)
    1 => float64,           ; Status
    2 => uint64,            ; Consecutive
    3 => bytes .size 1952,  ; Dilithium3PK (NOVO v2.0)
    4 => bytes .size 32,    ; VRFPK (NOVO v2.0)
}
```

**[MUST]** A verificação de `VRFProof` de um peer deve usar a `VRFPK` obtida do `NodeState` de rede, nunca da identidade local do verificador.

---

## 8. Rotação de Chaves

### 8.1 EventClass `3cp:key-rotation:v1`

**[MUST]** O protocolo define uma entrada de rotação de chaves:

```
key-rotation-entry = {
    ;; Herda estrutura base de provenance-entry
    0 => bytes .size 32,    ; Hash (BLAKE3-256 do payload)
    1 => bytes .size 16,    ; Submitter (ValidatorID)
    2 => int64,              ; Timestamp
    3 => tstr,               ; Label: "3cp:key-rotation:v1"

    ;; Campos específicos
    20 => bytes .size 1952,  ; NewPublicKey (Dilithium3)
    21 => bytes .size 32,    ; NewVRFPublicKey
    22 => uint64,             ; EffectiveCycle
    23 => uint64,             ; ExpiryCycle
    24 => bytes .size 3309,  ; SignatureOld (Dilithium3 com chave antiga)
    25 => bytes .size 3309,  ; SignatureNew (Dilithium3 com chave nova)
}
```

### 8.2 Regras de Validação

**[MUST]** Uma `key-rotation-entry` é válida se e somente se:
1. `SignatureOld` verifica contra a `Dilithium3PK` ativa do `Submitter` no estado do ciclo corrente.
2. `SignatureNew` verifica contra `NewPublicKey`.
3. `EffectiveCycle >= currentCycle + KeyRotationLeadTime` (default: 10).
4. `ExpiryCycle >= EffectiveCycle + MinKeyOverlap` (default: 10).
5. `EffectiveCycle > lastRotationCycle` do mesmo validador (proíbe rotações sobrepostas).

### 8.3 Período de Overlap

Durante `[EffectiveCycle, ExpiryCycle]`:
- Ambas as chaves (antiga e nova) são aceitas para verificação de blocos.
- Após `ExpiryCycle`, apenas a nova chave é aceita para novas assinaturas.

**[MUST]** Blocos anteriores a `EffectiveCycle` continuam verificáveis com a chave antiga indefinidamente.

---

## 9. Sparse Merkle Tree (SMT)

### 9.1 Parâmetros

- **Profundidade:** 256
- **Hash de folha:** `BLAKE3("leaf" || key || value)`
- **Hash de nó:** `BLAKE3(left || right)`
- **Chave:** `entry.Hash` (32 bytes)
- **Valor:** `entry.Hash` (32 bytes)

### 9.2 Prova de Inclusão

**[MUST]** Uma prova SMT é um array de 256 hashes de 32 bytes cada (8.192 bytes total), representando os irmãos no caminho da folha até a raiz.

### 9.3 Verificação

```
function VerifySMTProof(root, key, value, proof):
    current = BLAKE3("leaf" || key || value)
    for depth = 0 to 255:
        sibling = proof[depth]
        bit = (key[depth / 8] >> (depth % 8)) & 1
        if bit == 0:
            current = BLAKE3(current || sibling)
        else:
            current = BLAKE3(sibling || current)
    return current == root
```

---

## 10. Ciclo Adaptativo e Retenção de Entradas

### 10.1 Duração do Ciclo

**[MUST]** A duração do ciclo é adaptativa:

```
CycleDuration = BaseInterval + NetworkLatencyEstimate
NetworkLatencyEstimate = EWMA(RTT) * SafetyFactor
```

Onde:
- `BaseInterval`: default 3.000ms (configurável por Mandate)
- `SafetyFactor`: default 1.5 (configurável por Mandate)
- `MaxCycleDuration`: 10.000ms (hard cap do protocolo)
- `EWMA(RTT)`: média móvel exponencial do tempo de ida e volta entre pares

### 10.2 Retenção de Entradas Pendentes

**[MUST]** Entradas pendentes **nunca são descartadas** por expiração de ciclo.

**[MUST]** Entradas pendentes por mais de `MaxPendingTTL` ciclos (default: 100) são rejeitadas com erro `ErrPendingExpired`.

### 10.3 Ciclo Vazio

**[MAY]** Se `SkipEmptyCycles == true` (configurável por Mandate) e não há entradas pendentes, o ciclo pode ser pulado sem produção de bloco.

---

## 11. Cálculo do Laplaciano λ₁

### 11.1 Definição

O grafo de rede é representado por uma matriz de adjacência `A`, onde `A[i][j] = Status_j` se o nó `i` conhece o nó `j`. O Laplaciano `L = D - A`, onde `D` é a matriz diagonal de graus.

`λ₁` é o menor autovalor não nulo de `L` (autovalor de Fiedler).

### 11.2 Atualização Incremental

**[MUST]** Se entre ciclos consecutivos nenhum nó foi adicionado ou removido, e apenas os `Status` dos nós existentes mudaram, a implementação deve usar **rank-one update** no Laplaciano em vez de reconstruí-lo.

**[MUST]** O cálculo completo do Laplaciano só é permitido quando:
- Um nó é adicionado ou removido.
- `LambdaInterval` ciclos se passaram sem recomputação completa.
- O estado foi marcado como `dirty` por mudança estrutural.

### 11.3 Algoritmo para N > 100

**[MUST]** Para `N > 100`, o protocolo permite aproximação via **método de Lanczos** com número de iterações:

```
k = min(50, max(30, floor(N / 10)))
```

**[MUST]** O critério de parada do Lanczos deve incluir verificação de convergência do Ritz value. Se convergir antes de `k` iterações, a computação pode parar antecipadamente.

### 11.4 Fragmentação

**[MUST]** Se `λ₁ < MinLambda1` e `N >= 2`, o ciclo corrente é abortado e a rede entra em estado `fragmented` até que `λ₁` se recupere ou nós sejam removidos.

---

## 12. Verificabilidade por Terceiros

### 12.1 Anchor Publishers

**[MUST]** Um Anchor Publisher publica blocos finais em storage de leitura pública:

```
AnchorPublisherConfig = {
    0 => tstr,               ; Mode: "all" / "designated" / "external"
    1 => [* bytes .size 16], ; DesignatedPublishers (apenas se Mode == "designated")
    2 => uint,                ; MinRedundancy (default: 2)
}
```

**[MUST]** Backends obrigatórios:
- Filesystem local
- IPFS (CIDv1, codec `raw`, hash `blake3-256` ou `sha2-256`)
- Blob store S3-compatível (com checksum SHA-256)

### 12.2 Light Client Protocol

**[MUST]** O protocolo define operações de leitura para light clients:

```
GetBlock(index: uint64) -> Block
StreamBlocks(startIndex: uint64) -> stream Block
GetValidatorSet(cycle: uint64) -> [* validator-info]
GetMerkleProof(key: bytes .size 32, blockIndex: uint64) -> SMTProof
```

**[MUST]** Um light client verifica um bloco `B` sem executar consenso:
1. Obtém `ValidatorSet` do ciclo `B.Index`.
2. Verifica que `B.PrepareSigsPayload` contém `Q = ceil(2N/3)` assinaturas válidas contra as chaves do `ValidatorSet`, mapeando cada entrada para o validador pelo bit correspondente em `B.PrepareSigsBitmap`.
3. Verifica `B.CommitSig` contra `B.Proposer`.
4. Verifica a cadeia de `PrevHash`.

---

## 13. ZK Bridge Protocol

### 13.1 Interface ZKBridge v1.0.0

**[MUST]** O protocolo define uma interface estável para consumidores de provas de conhecimento zero:

```
ZKBridge v1.0.0 {
    GetBlockRange(start: uint64, end: uint64) -> [* Block]
    GetMerkleProof(key: bytes .size 32, blockIndex: uint64) -> SMTProof
    GetValidatorSet(cycle: uint64) -> [* validator-info]
}
```

**[MUST]** Alterações breaking exigem bump de versão major. Backward compatibility deve ser mantida por pelo menos 2 versões major.

### 13.2 Hash Functions para Circuitos ZK

**[SHOULD]** Circuitos ZK que consumam hashes 3CP devem usar funções de hash amigáveis a STARKs (e.g., Poseidon2, Rescue-Prime) para hashing interno do circuito, mantendo BLAKE3 apenas para hashing externo.

---

## 14. Derivação de Identidade UID0

### 14.1 NetworkID

**[MUST]** `NetworkID` é o hash BLAKE3-256 do bloco genesis:

```
NetworkID = BLAKE3-256(GenesisBlock)
```

**[MUST]** `NetworkID` é imutável para toda a vida da cadeia.

### 14.2 Derivação de Seed

**[MUST]** A seed de 32 bytes para derivação de identidade é:

```
seed = HKDF-SHA256(
    salt = NetworkID,
    info = "3cp-uid0-v2",
    ikm = entropySource
)
```

**[MUST]** `entropySource` deve conter no mínimo 128 bits de entropia efetiva. Implementações devem rejeitar `entropySource` com entropia estimada < 80 bits.

---

## 15. Mandatos

### 15.1 Estrutura

Ver `spec/schemas/mandate.cddl` para a definição CDDL completa.

**[MUST]** Um mandato é uma declaração assinada, versionada e ancorada que define obrigações de ancoragem. O `Authority` é o RootID que o emitiu, `Version` e `PrevVersion` formam a linhagem de revisões, e `Rules` declara as classes de eventos abrangidas.

**[MUST]** O `ID` de um mandato é BLAKE3-256 do CBOR canônico do mandato com `ID` e `Signature` excluídos. Uma implementação MUST recomputá-lo e MUST NOT confiar no valor armazenado: um corpo cujas regras não correspondem ao identificador que apresenta MUST ser rejeitado.

### 15.2 Autenticidade

**[MUST]** O campo `Signature` (chave 10) é uma assinatura Dilithium3 do `Authority` sobre o mesmo payload canônico, com `ID` e `Signature` excluídos.

**[MUST]** Uma implementação MUST verificar esta assinatura antes de aceitar um mandato como política vinculativa, e MUST resolvê-la contra uma chave ML-DSA-65 registada — o ValidatorSet ou o estado de rede. Um mandato cujo `Authority` não resolve para uma chave conhecida MUST ser rejeitado, e uma assinatura ausente ou com tamanho incorreto MUST ser rejeitada antes de qualquer verificação dispendiosa.

Sem esta verificação, `Authority` é uma afirmação: qualquer submissor poderia instalar um mandato em nome de outro RootID, e a rede o trataria como política autêntica. A ausência da assinatura MUST ser tratada como falha de autorização, não como ausência de exigência.

**[MUST]** A vigência (`ValidFrom`, `ValidUntil`) MUST NOT ser imposta no momento da admissão. Um auditor precisa de carregar um mandato expirado para avaliar uma janela que ele cobriu; recusá-lo na entrada apagaria o registo que a auditoria procura.

### 15.3 Fiscalização na Submissão

**[MUST]** Uma entrada que defina `MandateRef` MUST ser verificada contra o mandato que referencia: o mandato MUST existir, estar em vigor no timestamp da entrada, e ter todos os campos que as suas regras exigem.

Uma entrada sem `MandateRef` não faz qualquer afirmação e MUST ser aceite. A maioria das entradas de uma cadeia de proveniência não é governada por nenhum mandato, e rejeitá-las tornaria o mecanismo inutilizável.

Esta meia-fiscalização é estrutural: impede afirmações de conformidade que falham os requisitos mais básicos. NÃO pode deteção de eventos omitidos, porque o protocolo nunca vê um evento que não foi submetido.

### 15.4 Verificação e Fiscalização

Ver `spec/schemas/mandate.cddl` para as estruturas `compliance-verification` e `compliance-gap`.

**[MUST]** Uma implementação MUST oferecer uma operação que verifique a cadeia contra um mandato numa janela de tempo, comparando as entradas ancoradas com as obrigações declaradas.

**[MUST]** A omissão MUST ser criptograficamente detetável: um evento que um mandato exige e que nunca foi ancorado MUST produzir um gap, distinguível de uma entrada presente mas incompleta.

**[MUST]** A verificação MUST NOT derivar o conjunto de validadores do bloco que está a verificar. O conjunto tem de vir de uma âncora de confiança externa — o bloco genesis, um Anchor Publisher, ou um nó completo sob garantia do operador. Um bloco que nomeie os seus próprios validadores é assinável inteiramente por quem o escreveu.

### 15.5 Modo Degraded

**[MUST]** Um mandato é aplicável independentemente do modo de operação. A regra de quorum de §6.5 rege a finalização de blocos, não as obrigações de ancoragem: um mandato MUST NOT ser considerado cumprido com menos do que as suas próprias regras exigem, mesmo que a cadeia esteja em modo degraded.

---

## 16. Testes de Conformidade v2.0

**[MUST]** Implementações que declarem conformidade com 3CP v2.0 devem passar:

| ID | Categoria | Descrição |
|----|-----------|-----------|
| TC-BFT-01 | Consenso | Líder propõe dois blocos distintos → detectado e slashed |
| TC-BFT-02 | Consenso | Assinatura PREPARE inválida → validador marcado como suspect |
| TC-BFT-03 | Consenso | `f < N/3` falhas → consenso continua |
| TC-BFT-04 | Consenso | `f >= N/3` falhas → liveness failure (não safety) |
| TC-NET-01 | Rede | Partição 5 ciclos → reconciliação sem forks |
| TC-NET-02 | Rede | Latência > MaxCycleDuration → aborto com retenção |
| TC-ROT-01 | Chaves | Rotação válida → ambas as chaves aceitas no overlap |
| TC-ROT-02 | Chaves | Rotação com EffectiveCycle muito cedo → rejeitada |
| TC-ZK-01 | ZK | Prova SMT verificável por light client |
| TC-PUB-01 | Publicação | Bloco publicado → recuperável via ExternalAnchors |
| TC-SCA-01 | Escalabilidade | 10.000 entradas/min → throughput sustentado |
| TC-MEM-01 | Durabilidade | 1h contínua → estabilidade de estado |

---

## 17. Migração v1 → v2

**[MUST]** A migração da v1.0 para a v2.0 é um **hard fork**.

**Procedimento:**
1. Parar todos os nós da rede v1.0 no mesmo bloco final `B_final`.
2. Exportar: validator set, mandates ativos, SMT root, `H(B_final)`.
3. Criar bloco genesis v2.0 com:
   - `InitialValidators` = validator set exportado
   - `GenesisMandate` = mandates convertidos para formato v2
   - `LegacyAnchor` = `H(B_final)` (prova de continuidade)
4. Iniciar rede v2.0 a partir do novo genesis.

---

## 18. Referências Normativas

- FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard (ML-KEM)
- FIPS 204: Module-Lattice-Based Digital Signature Standard (ML-DSA)
- FIPS 205: Stateless Hash-Based Digital Signature Standard (SLH-DSA) — não usado no 3CP, mas referência pós-quântica
- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels
- RFC 8439: ChaCha20 and Poly1305 for IETF Protocols
- RFC 8610: Concise Data Definition Language (CDDL)
- RFC 8949: Concise Binary Object Representation (CBOR)
- RFC 9381: Verifiable Random Functions (VRFs)

---

## 19. Apêndice A: CDDLs Completos v2.0

### A.1 Bloco (block-v2.cddl)

Ver `spec/schemas/block.cddl` (atualizado conforme seção 5).

### A.2 Estado de Rede (network-state.cddl)

```
network-state = {
    0 => [* node-state],           ; Nodes
    1 => float64,                   ; Lambda1
    2 => uint64,                    ; Cycle
    3 => [* mandate-entry],         ; ActiveMandates
    4 => bytes .size 32,            ; SupervisionRoot
    5 => [* validator-info],        ; ValidatorSet (NOVO v2.0)
}

node-state = {
    0 => bytes .size 16,    ; UID
    1 => float64,            ; Status
    2 => uint64,            ; Consecutive
    3 => bytes .size 1952,  ; Dilithium3PK
    4 => bytes .size 32,    ; VRFPK
}

validator-info = {
    0 => bytes .size 16,    ; ValidatorID
    1 => bytes .size 1952,  ; Dilithium3PK
    2 => bytes .size 32,    ; VRFPK
    3 => bytes .size 32,    ; ContractHash
}
```

### A.3 Rotação de Chaves (key-rotation.cddl)

```
key-rotation-entry = {
    0 => bytes .size 32,    ; Hash
    1 => bytes .size 16,    ; Submitter
    2 => int64,              ; Timestamp
    3 => tstr,               ; Label: "3cp:key-rotation:v1"
    20 => bytes .size 1952,  ; NewPublicKey
    21 => bytes .size 32,    ; NewVRFPublicKey
    22 => uint64,             ; EffectiveCycle
    23 => uint64,             ; ExpiryCycle
    24 => bytes .size 3309,  ; SignatureOld
    25 => bytes .size 3309,  ; SignatureNew
}
```

### A.4 Genesis (genesis.cddl)

```
genesis-block = {
    0 => uint64,                    ; Index: 0
    1 => bytes .size 32,            ; PrevHash: zeros
    2 => bytes .size 32,            ; StateRoot
    3 => bytes .size 16,            ; Proposer
    5 => [* provenance-entry],       ; Anchored (GenesisMandate)
    6 => float64,                    ; Lambda1
    7 => int64,                      ; Timestamp
    9 => [* genesis-validator-info],  ; Validators (conjunto canônico; ver §7.1)
    10 => quorum-config,              ; Quorum
    11 => bytes .size 32,            ; BlockHash
    12 => uint16,                     ; ProtocolVersion: 2
    18 => bytes .size 32,            ; LegacyAnchor (H(B_final v1.0), se aplicável)
}

genesis-validator-set = [* genesis-validator-info]

genesis-validator-info = {
    0 => bytes .size 16,    ; ValidatorID
    1 => bytes .size 1952,  ; Dilithium3PK
    2 => bytes .size 32,    ; VRFPK
    3 => bytes .size 32,    ; ContractHash
}
```

### A.5 Light Client (light-client.cddl)

```
light-client-request = {
    0 => tstr,               ; Method: "GetBlock" / "StreamBlocks" / "GetValidatorSet" / "GetMerkleProof"
    1 => uint64,             ; StartIndex (para StreamBlocks)
    2 => uint64,             ; BlockIndex (para GetBlock, GetMerkleProof)
    3 => bytes .size 32,     ; Key (para GetMerkleProof)
}

light-client-response = {
    0 => bool,               ; Success
    1 => block / [* block] / [* validator-info] / smt-proof / null,
    2 => tstr,               ; Error message (se Success == false)
}
```

### A.6 ZK Bridge (zk-bridge.cddl)

```
zk-bridge-request = {
    0 => tstr,               ; Method: "GetBlockRange" / "GetMerkleProof" / "GetValidatorSet"
    1 => uint64,             ; Start
    2 => uint64,             ; End
    3 => bytes .size 32,     ; Key
    4 => uint64,             ; BlockIndex
    5 => hash-function,       ; HashFunction (para circuitos ZK)
}

hash-function = &(
    BLAKE3: 1,
    POSEIDON2: 2,            ; reservado para v2.1+
    RESCUE_PRIME: 3,         ; reservado para v2.1+
)
```

---

## 20. Apêndice B: Algoritmos Formais

### B.1 Seleção de Proposer

```
function SelectProposer(proofs, alpha):
    best_gamma = 0xFF...FF  ; 32 bytes, valor máximo
    best_proposer = null
    
    for each (signer_id, proof) in proofs:
        pk = LookupVRFPK(signer_id)  ; do NetworkState
        if pk == null: continue
        
        gamma, err = VRF_Verify(pk, alpha, proof)
        if err != null: continue  ; proof inválido
        
        if gamma < best_gamma:
            best_gamma = gamma
            best_proposer = signer_id
    
    return best_proposer
```

### B.2 Verificação de Quórum (Batch)

```
function VerifyQuorumBatch(blockHash, sigs, validators, required):
    if len(sigs) < required: return ErrQuorumNotMet
    
    valid = 0
    used_validators = set()
    
    for sig in sigs:
        for i, pk_bytes in validators:
            if i in used_validators: continue
            pk = Dilithium3_PKFromBytes(pk_bytes)
            if pk.Verify(blockHash, sig):
                valid++
                used_validators.add(i)
                break
    
    if valid < required: return ErrQuorumNotMet
    return nil
```

### B.3 Rank-One Update do Laplaciano

```
function IncrementalLaplacianUpdate(L_old, node_status_changes):
    L_new = L_old
    for (i, j, delta) in node_status_changes:
        ; delta = novo_status - antigo_status
        L_new[i][i] += delta
        L_new[i][j] -= delta
        L_new[j][i] -= delta
        L_new[j][j] += delta
    return L_new
```

---

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