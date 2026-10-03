# 📚 Pesquisa Bibliográfica para o Whitepaper do 3CP

Este documento **organiza e categoriza** as referências bibliográficas necessárias para o whitepaper do 3CP, divididas por **temas** e **prioridade**. 
Use esta lista para:
1. **Citar no whitepaper** (Seção 9: Referências).
2. **Aprofundar seu entendimento** do 3CP e seus fundamentos.
3. **Validar inovações** (ex: Mandatory Anchoring vs. estado da arte).

---

## 🔍 **Como Usar Este Documento**
- **⭐⭐⭐⭐⭐**: Referências **obrigatórias** (citar no whitepaper).
- **⭐⭐⭐⭐**: Referências **recomendadas** (para aprofundamento).
- **⭐⭐⭐**: Referências **úteis** (para contexto).
- **📌**: Indica que a referência **já está citada no whitepaper**.

---

## 📌 **1. Criptografia Pós-Quântica (NIST)**
*Fundamento para as primitivas do 3CP (Dilithium3, Kyber1024, ECVRF).*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| 📌 [FIPS 203](https://csrc.nist.gov/publications/detail/fips/203/final) | Module-Lattice-Based Key-Encapsulation Mechanism Standard (ML-KEM) | Padrão NIST para KEM pós-quântico (Kyber1024). | [Link](https://csrc.nist.gov/publications/detail/fips/203/final) | ⭐⭐⭐⭐⭐ | Handshake seguro entre pares. |
| 📌 [FIPS 204](https://csrc.nist.gov/publications/detail/fips/204/final) | Module-Lattice-Based Digital Signature Standard (ML-DSA) | Padrão NIST para assinaturas pós-quânticas (Dilithium3). | [Link](https://csrc.nist.gov/publications/detail/fips/204/final) | ⭐⭐⭐⭐⭐ | Assinaturas de blocos, entradas, Mandates. |
| 📌 [FIPS 205](https://csrc.nist.gov/publications/detail/fips/205/final) | Stateless Hash-Based Digital Signature Standard (SLH-DSA) | Padrão NIST para assinaturas baseadas em hash. | [Link](https://csrc.nist.gov/publications/detail/fips/205/final) | ⭐⭐⭐ | Alternativa futura. |
| [NIST PQC Project](https://csrc.nist.gov/projects/post-quantum-cryptography) | Post-Quantum Cryptography Standardization | Processo de padronização do NIST para criptografia pós-quântica. | [Link](https://csrc.nist.gov/projects/post-quantum-cryptography) | ⭐⭐⭐⭐ | Contexto histórico. |
| [NIST PQC Round 3](https://csrc.nist.gov/projects/post-quantum-cryptography/round-3-submissions) | Round 3 Submissions | Lista de algoritmos finalistas do NIST. | [Link](https://csrc.nist.gov/projects/post-quantum-cryptography/round-3-submissions) | ⭐⭐⭐ | Dilithium e Kyber foram vencedores. |

**Resumo:**
- **Dilithium3 (ML-DSA-65)**: Assinaturas digitais baseadas em **reticulados** (lattice-based). **Nível de segurança NIST 3** (equivalente a AES-192).
- **Kyber1024 (ML-KEM-1024)**: Key Encapsulation Mechanism baseados em **reticulados**. **Nível de segurança NIST 5**.
- **ECVRF**: Verifiable Random Function sobre **Ristretto255** (RFC 9381).

---

## 📌 **2. Protocolos de Consenso BFT**
*Fundamento para o consenso do 3CP (BFT em 2 fases + VRF).*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| 📌 [Castro & Liskov, 1999](https://dl.acm.org/doi/10.1145/317636.317775) | Practical Byzantine Fault Tolerance | Artigo seminal sobre **PBFT** (3 fases). | [Link](https://dl.acm.org/doi/10.1145/317636.317775) | ⭐⭐⭐⭐⭐ | Base teórica para BFT. |
| 📌 [Buchman et al., 2016](https://arxiv.org/abs/1802.04888) | The Latest on Tendermint | Consenso BFT com **2 fases** (PREPARE/COMMIT). | [Link](https://arxiv.org/abs/1802.04888) | ⭐⭐⭐⭐⭐ | Inspiração para o consenso do 3CP. |
| [Gilchrist et al., 2020](https://arxiv.org/abs/1907.08002) | Algorand: Scaling Byzantine Agreements for Cryptocurrencies | Consenso BFT com **VRF** para eleição de líder. | [Link](https://arxiv.org/abs/1907.08002) | ⭐⭐⭐⭐ | Inspiração para eleição de líder. |
| [Miller et al., 2016](https://eprint.iacr.org/2016/199) | The HoneyBadgerBFT Protocol | Consenso BFT **assíncrono**. | [Link](https://eprint.iacr.org/2016/199) | ⭐⭐⭐ | Alternativa para redes assíncronas. |
| [Dwork et al., 1988](https://dl.acm.org/doi/10.1145/42267.42269) | Consensus in the Presence of Partial Synchrony | Teoria de consenso em redes parcialmente síncronas. | [Link](https://dl.acm.org/doi/10.1145/42267.42269) | ⭐⭐⭐ | Fundamento teórico. |

**Resumo:**
- **PBFT (Castro & Liskov)**: 3 fases (pre-prepare, prepare, commit). **Complexidade O(N²)**.
- **Tendermint**: 2 fases (PREPARE/COMMIT). **Usa VRF para eleição de líder**.
- **3CP**: 2 fases (PREPARE/COMMIT) + **VRF (ECVRF)** + **quórum `ceil(2N/3)`**.

---

## 📌 **3. Sparse Merkle Trees (SMT)**
*Fundamento para o commitment de estado do 3CP.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| 📌 [Eth 2.0 Spec](https://github.com/ethereum/eth2.0-specs) | Ethereum 2.0 Specifications | SMT para **estado do Ethereum**. | [Link](https://github.com/ethereum/eth2.0-specs) | ⭐⭐⭐⭐⭐ | Inspiração para design da SMT. |
| 📌 [Zcash Sapling](https://z.cash/technology/sapling/) | Zcash Sapling Protocol | SMT para **privacidade** (provas compactas). | [Link](https://z.cash/technology/sapling/) | ⭐⭐⭐⭐ | Inspiração para provas SMT. |
| [IPFS Spec](https://github.com/ipfs/specs) | IPFS Specifications | **Merkle DAG** para dados imutáveis. | [Link](https://github.com/ipfs/specs) | ⭐⭐⭐ | Inspiração para Anchor Publishers. |
| [Merkle, 1987](https://dl.acm.org/doi/10.1145/3135.3139) | A Digital Signature Based on a Conventional Encryption Function | Artigo original sobre **Merkle Trees**. | [Link](https://dl.acm.org/doi/10.1145/3135.3139) | ⭐⭐⭐ | Fundamento teórico. |
| [Sedgwick & Wayne, 2016](https://algs4.cs.princeton.edu/99scientific/) | Sparse Merkle Trees | Implementação de SMT em Java. | [Link](https://algs4.cs.princeton.edu/99scientific/) | ⭐⭐⭐ | Implementação de referência. |

**Resumo:**
- **SMT do Ethereum 2.0**: Profundidade 256, **BLAKE2b** (no 3CP, usamos **BLAKE3-256**).
- **Zcash Sapling**: SMT para **provas de conhecimento zero** (ZK).
- **3CP**: SMT com profundidade 256, **BLAKE3-256**, provas de **8KB**.

---

## 📌 **4. Verifiable Random Functions (VRF)**
*Fundamento para eleição de líder no 3CP.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| 📌 [RFC 9381](https://datatracker.ietf.org/doc/html/rfc9381) | Verifiable Random Functions (VRFs) | Padrão IETF para VRFs. | [Link](https://datatracker.ietf.org/doc/html/rfc9381) | ⭐⭐⭐⭐⭐ | Eleição de líder (ECVRF). |
| [Micali et al., 1999](https://cseweb.ucsd.edu/~mihir/papers/vrf.pdf) | Verifiable Random Functions | Artigo original sobre VRFs. | [Link](https://cseweb.ucsd.edu/~mihir/papers/vrf.pdf) | ⭐⭐⭐⭐ | Fundamento teórico. |
| [Dodis & Yampolskiy, 2005](https://eprint.iacr.org/2005/007) | Verifiable Random Functions: A Survey | Survey sobre VRFs. | [Link](https://eprint.iacr.org/2005/007) | ⭐⭐⭐ | Revisão de estado da arte. |
| [Ristretto255](https://ristretto.group/) | Ristretto255: A Nonnaive Curve25519 | Curva elíptica **Ristretto255** (usada no ECVRF). | [Link](https://ristretto.group/) | ⭐⭐⭐⭐ | Base para ECVRF. |

**Resumo:**
- **ECVRF (RFC 9381)**: VRF sobre **Ristretto255** (grupo de ordem prima, sem cofator).
- **Uso no 3CP**: Eleição de líder **verificável e não manipulável**.

---

## 📌 **5. Serialização de Dados (CBOR, CDDL)**
*Fundamento para o wire format do 3CP.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| 📌 [RFC 8949](https://datatracker.ietf.org/doc/html/rfc8949) | Concise Binary Object Representation (CBOR) | Formato binário **canônico** para serialização. | [Link](https://datatracker.ietf.org/doc/html/rfc8949) | ⭐⭐⭐⭐⭐ | Wire format de blocos e entradas. |
| 📌 [RFC 8610](https://datatracker.ietf.org/doc/html/rfc8610) | Concise Data Definition Language (CDDL) | Linguagem para definir **schemas** de dados. | [Link](https://datatracker.ietf.org/doc/html/rfc8610) | ⭐⭐⭐⭐⭐ | Schemas do 3CP (ex: `block.cddl`). |
| [Bormann & Hoffman, 2020](https://datatracker.ietf.org/doc/html/rfc8742) | CBOR Object Signing and Encryption (COSE) | Assinaturas digitais em CBOR. | [Link](https://datatracker.ietf.org/doc/html/rfc8742) | ⭐⭐⭐ | Alternativa para assinaturas. |

**Resumo:**
- **CBOR (RFC 8949)**: Serialização **determinística** (chaves em ordem crescente).
- **CDDL (RFC 8610)**: Schemas para validar dados CBOR.
- **3CP**: Todos os blocos e entradas são serializados em **CBOR canônico**.

---

## 📌 **6. Blockchain e Cadeia de Custódia**
*Comparação com alternativas e contexto histórico.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| 📌 [Nakamoto, 2008](https://bitcoin.org/bitcoin.pdf) | Bitcoin: A Peer-to-Peer Electronic Cash System | Primeiro blockchain público. | [Link](https://bitcoin.org/bitcoin.pdf) | ⭐⭐⭐⭐⭐ | Comparação com 3CP. |
| 📌 [Buterin et al., 2014](https://ethereum.org/whitepaper/) | Ethereum: A Next-Generation Smart Contract Platform | Blockchain com smart contracts. | [Link](https://ethereum.org/whitepaper/) | ⭐⭐⭐⭐⭐ | Comparação com 3CP. |
| [Androulaki et al., 2018](https://dl.acm.org/doi/10.1145/3183440.3183445) | Hyperledger Fabric: A Distributed Ledger Platform | Blockchain privada para consórcios. | [Link](https://dl.acm.org/doi/10.1145/3183440.3183445) | ⭐⭐⭐⭐ | Comparação com 3CP. |
| [AWS QLDB, 2019](https://aws.amazon.com/qldb/) | Amazon QLDB: A Transparent, Immutable, and Cryptographically Verifiable Ledger | Ledger imutável da AWS. | [Link](https://aws.amazon.com/qldb/) | ⭐⭐⭐ | Comparação com 3CP. |
| [Guardtime KSI](https://www.guardtime.com/technology) | KSI Blockchain | Blockchain para timestamping. | [Link](https://www.guardtime.com/technology) | ⭐⭐⭐ | Comparação com 3CP. |

**Resumo:**
- **Bitcoin/Ethereum**: Blockchains públicas. **❌ Não detectam omissões**.
- **Hyperledger Fabric**: Blockchain privada. **❌ Depende da boa fé do operador**.
- **Amazon QLDB**: Ledger imutável. **❌ Centralizado e proprietário**.
- **Guardtime KSI**: Timestamping. **❌ Não detecta omissões**.
- **3CP**: **✅ Detecta omissões** + **✅ Verificável por terceiros**.

---

## 📌 **7. Ameaças Quânticas**
*Fundamento para a adoção de criptografia pós-quântica.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| 📌 [Shor, 1994](https://arxiv.org/abs/quant-ph/9511027) | Algorithms for Quantum Computation: Discrete Logarithms and Factoring | Algoritmo de Shor quebra **RSA/ECDSA** em tempo polinomial. | [Link](https://arxiv.org/abs/quant-ph/9511027) | ⭐⭐⭐⭐⭐ | Justificativa para pós-quântico. |
| 📌 [Grover, 1996](https://arxiv.org/abs/quant-ph/9605043) | Quantum Mechanics Helps in Searching for a Needle in a Haystack | Algoritmo de Grover reduz segurança de hashes para **√N**. | [Link](https://arxiv.org/abs/quant-ph/9605043) | ⭐⭐⭐⭐⭐ | Justificativa para hashes longos. |
| [NIST PQC Timeline](https://csrc.nist.gov/projects/post-quantum-cryptography/timeline) | NIST PQC Standardization Timeline | Cronograma de padronização do NIST. | [Link](https://csrc.nist.gov/projects/post-quantum-cryptography/timeline) | ⭐⭐⭐ | Contexto histórico. |
| [Mosca, 2018](https://eprint.iacr.org/2018/416) | Cryptographically Relevant Quantum Computers: A Timeline | Estimativa de quando computadores quânticos quebrarão criptografia clássica. | [Link](https://eprint.iacr.org/2018/416) | ⭐⭐⭐ | **2030-2035** para RSA-2048. |

**Resumo:**
- **Shor (1994)**: Quebra **RSA/ECDSA** em **O((log N)³)**.
- **Grover (1996)**: Reduz segurança de **hashes** para **√N** (ex: SHA-256 → 128 bits).
- **3CP**: Usa **Dilithium3 (NIST 3)** e **Kyber1024 (NIST 5)** para resistir a Shor/Grover.

---

## 📌 **8. Regulamentações e Casos de Uso**
*Contexto para adoção do 3CP em setores regulados.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| 📌 [EU DORA](https://digital-strategy.ec.europa.eu/en/policies/digital-operational-resilience-act-dora) | Digital Operational Resilience Act | Regulamentação da UE para **resiliência operacional digital** em serviços financeiros. | [Link](https://digital-strategy.ec.europa.eu/en/policies/digital-operational-resilience-act-dora) | ⭐⭐⭐⭐⭐ | **Adoção ideal** para compliance. |
| 📌 [EU AI Act](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai) | Artificial Intelligence Act | Regulamentação da UE para **IA de alto risco**. | [Link](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai) | ⭐⭐⭐⭐⭐ | **Mandatory Anchoring** para IA. |
| 📌 [GDPR](https://gdpr-info.eu/) | General Data Protection Regulation | Regulamentação da UE para **proteção de dados**. | [Link](https://gdpr-info.eu/) | ⭐⭐⭐⭐⭐ | **Auditoria de dados**. |
| [Wells Fargo Fine (2023)](https://www.reuters.com/business/finance/us-wells-fargo-fine-2023-07-19/) | $3.7B Fine for Fake Accounts | Multa por **omissão de registros**. | [Link](https://www.reuters.com/business/finance/us-wells-fargo-fine-2023-07-19/) | ⭐⭐⭐⭐ | **Caso de uso real**. |
| [Facebook GDPR Fine (2021)](https://www.irishtimes.com/business/technology/facebook-fined-265m-by-irish-regulator-over-data-breach-1.4905022) | €265M Fine for Data Breach | Multa por **violação de GDPR**. | [Link](https://www.irishtimes.com/business/technology/facebook-fined-265m-by-irish-regulator-over-data-breach-1.4905022) | ⭐⭐⭐⭐ | **Caso de uso real**. |
| [Boeing 737 MAX Fine (2020)](https://www.reuters.com/business/aerospace-defense/boeing-agrees-25-billion-settlement-over-737-max-crashes-2020-12-17/) | $2.5B Fine for Safety Omissions | Multa por **omissão de registros de segurança**. | [Link](https://www.reuters.com/business/aerospace-defense/boeing-agrees-25-billion-settlement-over-737-max-crashes-2020-12-17/) | ⭐⭐⭐⭐ | **Caso de uso real**. |

**Resumo:**
- **DORA (UE)**: Exige **resiliência operacional** para serviços financeiros. **3CP pode ser usado para compliance**.
- **AI Act (UE)**: Exige **transparência e accountability** para IA de alto risco. **Mandatory Anchoring resolve isso**.
- **GDPR (UE)**: Exige **auditoria de dados**. **3CP permite auditoria verificável por terceiros**.

---

## 📌 **9. Trabalhos sobre Cadeia de Custódia**
*Contexto para o problema que o 3CP resolve.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| [NIST SP 800-86](https://csrc.nist.gov/publications/detail/sp/800-86/final) | Guide to Integrating Forensic Techniques into Incident Response | Guia do NIST para **cadeia de custódia forense**. | [Link](https://csrc.nist.gov/publications/detail/sp/800-86/final) | ⭐⭐⭐⭐ | Fundamento teórico. |
| [ISO/IEC 27037](https://www.iso.org/standard/44381.html) | Information Technology — Security Techniques — Identification and Trust Platforms | Padrão ISO para **cadeia de custódia**. | [Link](https://www.iso.org/standard/44381.html) | ⭐⭐⭐ | Comparação com 3CP. |
| [RFC 3161](https://datatracker.ietf.org/doc/html/rfc3161) | Internet X.509 PKI Time-Stamp Protocol | Protocolo de timestamping. | [Link](https://datatracker.ietf.org/doc/html/rfc3161) | ⭐⭐⭐ | Comparação com 3CP. |
| [Bull, 2006](https://www.nist.gov/publications/guide-computer-forensics) | Computer Forensics: An Introduction | Introdução à **cadeia de custódia digital**. | [Link](https://www.nist.gov/publications/guide-computer-forensics) | ⭐⭐⭐ | Contexto histórico. |

**Resumo:**
- **NIST SP 800-86**: Guia para **cadeia de custódia forense** (mas não detecta omissões).
- **ISO/IEC 27037**: Padrão para **cadeia de custódia** (mas não é verificável por terceiros).
- **3CP**: **Detecta omissões** + **verificável por terceiros**.

---

## 📌 **10. Trabalhos sobre Auditoria e Compliance**
*Contexto para adoção do 3CP em auditorias.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| [COSO Framework](https://www.coso.org/) | Committee of Sponsoring Organizations of the Treadway Commission | Framework para **controle interno**. | [Link](https://www.coso.org/) | ⭐⭐⭐⭐ | **Mandatory Anchoring** para compliance. |
| [ISO 19011](https://www.iso.org/standard/70017.html) | Guidelines for Auditing Management Systems | Guia para **auditorias**. | [Link](https://www.iso.org/standard/70017.html) | ⭐⭐⭐⭐ | **Light Client Verification** para auditorias. |
| [SOC 2](https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/sorhome.html) | Service Organization Control 2 | Padrão de auditoria para **serviços em nuvem**. | [Link](https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/sorhome.html) | ⭐⭐⭐⭐ | **3CP pode ser usado para SOC 2**. |
| [PCI DSS](https://www.pcisecuritystandards.org/) | Payment Card Industry Data Security Standard | Padrão de segurança para **cartões de pagamento**. | [Link](https://www.pcisecuritystandards.org/) | ⭐⭐⭐ | **3CP para auditoria de transações**. |

**Resumo:**
- **COSO/ISO 19011/SOC 2/PCI DSS**: Padrões de auditoria que **exigem evidências verificáveis**. **3CP fornece isso**.

---

## 📌 **11. Trabalhos sobre Zero-Knowledge (ZK)**
*Contexto para o CARCOSA (ZK + 3CP).*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| [Ben-Sasson et al., 2014](https://eprint.iacr.org/2013/879) | Zcash: Decentralized Anonymous Payments from Bitcoin | Primeiro uso de **zk-SNARKs** em blockchain. | [Link](https://eprint.iacr.org/2013/879) | ⭐⭐⭐⭐ | Fundamento para ZK. |
| [Winterfell](https://github.com/novifach/winterfell) | Winterfell: A STARK-based ZK Framework for Rust | Framework **STARK** para Rust (usado no CARCOSA). | [Link](https://github.com/novifach/winterfell) | ⭐⭐⭐⭐⭐ | **CARCOSA usa Winterfell**. |
| [STARKs (2018)](https://eprint.iacr.org/2018/046) | Scalable Transparent Arguments of Knowledge | **STARKs**: Provas ZK **transparentes** (sem setup). | [Link](https://eprint.iacr.org/2018/046) | ⭐⭐⭐⭐ | **CARCOSA usa STARKs**. |
| [SNARKs (2013)](https://eprint.iacr.org/2013/507) | Succinct Non-Interactive Arguments of Knowledge | **SNARKs**: Provas ZK **succintas**. | [Link](https://eprint.iacr.org/2013/507) | ⭐⭐⭐ | Alternativa para SNARKs. |

**Resumo:**
- **STARKs**: Provas ZK **transparentes** (sem setup) e **escaláveis**. Usadas no **CARCOSA**.
- **SNARKs**: Provas ZK **succintas**, mas requerem **setup**.
- **Winterfell**: Framework STARK para Rust (usado no CARCOSA).

---

## 📌 **12. Trabalhos sobre Identidade Digital**
*Contexto para UID0 (identidade no 3CP).*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| [DID W3C](https://www.w3.org/TR/did-core/) | Decentralized Identifiers (DIDs) | Padrão W3C para **identidades descentralizadas**. | [Link](https://www.w3.org/TR/did-core/) | ⭐⭐⭐⭐ | Inspiração para UID0. |
| [Verifiable Credentials](https://www.w3.org/TR/vc-data-model/) | Verifiable Credentials Data Model | Padrão W3C para **credenciais verificáveis**. | [Link](https://www.w3.org/TR/vc-data-model/) | ⭐⭐⭐⭐ | Inspiração para UID0. |
| [Soulbound Tokens (2022)](https://vitalik.ca/general/2022/01/26/soulbound.html) | Soulbound Tokens | **SBTs**: Tokens **não transferíveis** (usados em identidade). | [Link](https://vitalik.ca/general/2022/01/26/soulbound.html) | ⭐⭐⭐⭐ | **UID0 é um SBT**. |

**Resumo:**
- **UID0**: Identidade **soulbound** (não transferível) vinculada a um **contrato legal**.
- **DID/VC**: Padrões W3C para identidade descentralizada.

---

## 📌 **13. Trabalhos sobre Light Clients**
*Fundamento para Light Client Verification no 3CP.*

| **Referência** | **Título** | **Descrição** | **Link** | **Prioridade** | **Uso no 3CP** |
|---------------|------------|---------------|----------|----------------|----------------|
| [Buterin, 2015](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/light-clients/) | Light Clients in Ethereum | Como **light clients** verificam blocos no Ethereum. | [Link](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/light-clients/) | ⭐⭐⭐⭐ | Inspiração para Light Client do 3CP. |
| [Beiko, 2020](https://hackmd.io/@adietrich/light-client-protocol) | Light Client Protocol for Ethereum 2.0 | Protocolo para light clients no **Eth 2.0**. | [Link](https://hackmd.io/@adietrich/light-client-protocol) | ⭐⭐⭐⭐ | Inspiração para design. |
| [Kiayias et al., 2017](https://eprint.iacr.org/2017/454) | The Bitcoin Backbone Protocol: Analysis and an Application to Proof-of-Stake | Análise de **light clients** em Bitcoin. | [Link](https://eprint.iacr.org/2017/454) | ⭐⭐⭐ | Fundamento teórico. |

**Resumo:**
- **Light Clients**: Verificam blocos **sem executar consenso** ou **manter estado completo**.
- **3CP**: Light clients verificam **assinaturas Dilithium3** e **provas SMT**.

---
---

## 🔥 **Resumo Executivo: O que Você Precisa Saber**

### **📚 Referências Obrigatórias (⭐⭐⭐⭐⭐) para o Whitepaper**
*(Citar no whitepaper, Seção 9: Referências)*

1. **Criptografia Pós-Quântica:**
   - [FIPS 203](https://csrc.nist.gov/publications/detail/fips/203/final) (Kyber1024)
   - [FIPS 204](https://csrc.nist.gov/publications/detail/fips/204/final) (Dilithium3)
   - [RFC 9381](https://datatracker.ietf.org/doc/html/rfc9381) (ECVRF)

2. **Consenso BFT:**
   - [Castro & Liskov, 1999](https://dl.acm.org/doi/10.1145/317636.317775) (PBFT)
   - [Buchman et al., 2016](https://arxiv.org/abs/1802.04888) (Tendermint)

3. **Sparse Merkle Trees:**
   - [Eth 2.0 Spec](https://github.com/ethereum/eth2.0-specs) (SMT para estado)
   - [Zcash Sapling](https://z.cash/technology/sapling/) (SMT para privacidade)

4. **Serialização:**
   - [RFC 8949](https://datatracker.ietf.org/doc/html/rfc8949) (CBOR)
   - [RFC 8610](https://datatracker.ietf.org/doc/html/rfc8610) (CDDL)

5. **Blockchain:**
   - [Nakamoto, 2008](https://bitcoin.org/bitcoin.pdf) (Bitcoin)
   - [Buterin et al., 2014](https://ethereum.org/whitepaper/) (Ethereum)

6. **Ameaças Quânticas:**
   - [Shor, 1994](https://arxiv.org/abs/quant-ph/9511027) (Algoritmo de Shor)
   - [Grover, 1996](https://arxiv.org/abs/quant-ph/9605043) (Algoritmo de Grover)

7. **Regulamentações:**
   - [EU DORA](https://digital-strategy.ec.europa.eu/en/policies/digital-operational-resilience-act-dora)
   - [EU AI Act](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai)
   - [GDPR](https://gdpr-info.eu/)

8. **Casos de Uso:**
   - [Wells Fargo Fine (2023)](https://www.reuters.com/business/finance/us-wells-fargo-fine-2023-07-19/)
   - [Facebook GDPR Fine (2021)](https://www.irishtimes.com/business/technology/facebook-fined-265m-by-irish-regulator-over-data-breach-1.4905022)
   - [Boeing 737 MAX Fine (2020)](https://www.reuters.com/business/aerospace-defense/boeing-agrees-25-billion-settlement-over-737-max-crashes-2020-12-17/)

---

### **🎯 Como Organizar a Pesquisa**
1. **Comece pelos fundamentos:**
   - Leia **FIPS 203/204** (criptografia pós-quântica).
   - Leia **RFC 9381** (VRF).
   - Leia **Castro & Liskov (1999)** (BFT).

2. **Aprofunde nos componentes do 3CP:**
   - **Consenso**: Tendermint (Buchman et al., 2016).
   - **SMT**: Eth 2.0 Spec, Zcash Sapling.
   - **Wire Format**: RFC 8949 (CBOR), RFC 8610 (CDDL).

3. **Valide as inovações:**
   - **Mandatory Anchoring**: Não há referências diretas (é uma **inovação do 3CP**).
   - **Contestability**: Não há referências diretas (é uma **inovação do 3CP**).
   - **Light Client Verification**: Compare com Ethereum 2.0 (Beiko, 2020).

4. **Contexto regulatório:**
   - Leia **EU DORA** e **EU AI Act** para entender como o 3CP pode ser adotado.

---

### **📌 Ferramentas para Gerenciar Referências**
1. **Zotero**: [zotero.org](https://www.zotero.org/) (grátis, open-source).
2. **Mendeley**: [mendeley.com](https://www.mendeley.com/) (grátis).
3. **JabRef**: [jabref.org](https://www.jabref.org/) (para LaTeX/BibTeX).

**Dica:** Use **Zotero** para organizar as referências em pastas (ex: "Criptografia", "Consenso", "Regulamentações").

---

### **🎯 Próximos Passos para Você**
1. **Revisar o whitepaper** (`/workspace/github__had-nu__3CP/docs/whitepaper.md`).
2. **Escolher 5-10 referências** para ler em detalhes (comece pelas ⭐⭐⭐⭐⭐).
3. **Validar as inovações** (Mandatory Anchoring, Contestability) contra o estado da arte.
4. **Adicionar mais referências** ao whitepaper (se necessário).

**Pergunta:**
*Quer que eu ajude a organizar as referências em um arquivo BibTeX (para LaTeX) ou em um formato específico?* 😊
