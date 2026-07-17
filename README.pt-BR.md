<p align="center">
  <h1>3CP</h1>
</p>

<p align="center">
  <em>Protocolo Criptográfico de Cadeia de Custódia</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-draft-yellow" alt="Status: Rascunho">
  <img src="https://img.shields.io/badge/protocol--version-v1-blue" alt="Versão do Protocolo: v1">
  <img src="https://img.shields.io/badge/conformance-33%2F33%20passing-brightgreen" alt="Conformidade: 33/33">
  <img src="https://img.shields.io/badge/pre--publication-private-red" alt="Pré-publicação: Privado">
</p>

---

Um **protocolo de camada de aplicação** para produzir, preservar e verificar
evidências criptográficas de cadeia de custódia em sistemas distribuídos. O 3CP
permite que qualquer parte — regulador, auditor, consumidor, verificador
terceiro — verifique de forma independente que um determinado artefato digital
existiu em um determinado momento, que sua proveniência foi atestada por
identidades conhecidas e que nenhuma reconstrução retroativa de eventos é
possível sem detecção.

## Tese Central

> A responsabilização não pode depender da boa-fé da entidade auditada.

O 3CP estende este princípio com **Mandatory Event Anchoring** (§13):
declarações assinadas, versionadas e nativas do protocolo que definem quais
eventos DEVEM ser registrados. A conformidade é verificável de forma
independente ao comparar a cadeia contra o conjunto de mandatos ativo. A
omissão torna-se detectável.

Estruturas de conformidade existentes (ISO 27001, SOC 2, NIST CSF, DORA, CRA)
presumem que as organizações produzem evidências honestamente e que os
auditores podem avaliá-las. Quando a responsabilidade legal e o interesse
econômico entram em conflito, existe um incentivo objetivo para alterar, suprimir,
fragmentar ou reinterpretar essas evidências. O 3CP inverte esse modelo:
torna a integridade da evidência de cadeia de custódia verificável por
terceiros independentes, independentemente da cooperação do operador.

## O Que o 3CP Não É

O 3CP não é uma blockchain, não é um token, não é uma plataforma de
smart-contracts e não é um livro-razão de propósito geral. É um protocolo —
um conjunto de formatos de transmissão, regras de consenso e algoritmos de
verificação — que qualquer sistema pode implementar para produzir evidências
contestáveis de proveniência de decisões.

## Propriedades Principais

| Propriedade | O que significa |
|---|---|
| **Contestabilidade** | A evidência pode ser desafiada, mas o desafio ocorre sobre a cadeia intacta, não sobre uma cadeia reconstruída após o incidente |
| **Verificabilidade por terceiros** | Qualquer parte com a cadeia pública pode verificar de forma independente provas SMT, assinaturas e hashes de bloco sem contatar a rede produtora |
| **Não-repúdio** | Cada entrada é criptograficamente vinculada à identidade submissora via assinaturas Dilithium3; uma vez validada pelo quórum, nenhuma parte pode negar a submissão |
| **Preservação epistêmica** | O sistema registra quem sabia o quê, quando, quem aprovou, quem assinou, quem alterou — a epistemologia do incidente, não apenas hashes |
| **Segurança pós-quântica** | Assinaturas Dilithium3, KEM Kyber1024, ECVRF (Ristretto255) — criptografia pós-quântica padronizada pelo NIST em toda a pilha |
| **Ancoragem obrigatória** | Declarações de Mandato nativas do protocolo (§13) tornam as obrigações de ancoragem verificáveis independentemente; lacunas de conformidade (entradas ausentes, campos ausentes) são detectáveis criptograficamente por qualquer terceiro |

## Pilha de Protocolo

| Camada | Especificação |
|---|---|
| Formato de transmissão | CBOR canônico (chaveado por inteiro, determinístico) — veja [`spec/3CP.md`](spec/3CP.md) §4 |
| Consenso | Eleição de líder ECVRF (RFC 9381) + quórum M-de-N Dilithium3 — §6 |
| Estado | Árvore Merkle Esparsa (Blake3, profundidade 256) — §5 |
| Transporte (canônico) | gRPC sobre Protocol Buffers — §7 |
| Sub-cadeias | SMT por serviço + âncoras entre cadeias — §9 |
| Identidade | Tokens soulbound UID0 com vinculação derivada de contrato — §8 |
| Mandatos | Declarações assinadas, versionadas e ancoradas de obrigações de ancoragem — §13 |

## Estrutura do Repositório

```
3CP/
├── README.md            ← este arquivo
├── README.pt-BR.md      ← versão em português
├── CONTRIBUTING.md      ← como propor alterações
├── LICENSE              ← Todos os Direitos Reservados (pré-publicação)
├── spec/
│   ├── 3CP.md           ← especificação normativa do protocolo (RFC 2119)
│   ├── glossary.md       ← terminologia
│   ├── schemas/          ← definições de formato CDDL
│   └── examples/         ← vetores de teste
```

## Status

- **Rascunho** — a especificação está completa e validada por uma suíte de
  conformidade de 33 testes contra a implementação de referência.
- **Pré-publicação** — este repositório é privado. A especificação será
  aberta junto com o artigo acadêmico correspondente.
- **Versão do protocolo**: v1 — formato de transmissão estável. Versões
  futuras serão compatíveis com versões anteriores ou explicitamente
  versionadas.

## Implementação de Referência

**Gleipnir** — uma implementação de referência em Go do protocolo 3CP.
Disponível em [github.com/had-nu/gleipnir](https://github.com/had-nu/gleipnir)
sob AGPL-3.0. A suíte de testes de conformidade do Gleipnir (33 testes)
valida o protocolo na fronteira gRPC.

## Citação (pré-publicação)

```bibtex
@techreport{3cp-protocol,
  title        = {{3CP}: Protocolo Criptográfico de Cadeia de Custódia},
  author       = {André Ataíde},
  year         = {2026},
  note         = {Rascunho de pré-publicação}
}
```

---

<p align="center">
  <strong>3CP</strong> — Protocolo Criptográfico de Cadeia de Custódia v1<br>
  Todos os Direitos Reservados © 2026
</p>
