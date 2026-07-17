# Playbook 3CP — Guia Prático para Estagiários

> Este playbook ensina um estagiário com 1 semana de empresa a rodar o Gleipnir
> (implementação de referência do protocolo 3CP), executar a suíte de testes,
> usar um cliente gRPC e contribuir com o projeto.
>
> **Pré-requisito**: Linux ou macOS com terminal. Windows pode funcionar via WSL2.

---

## Índice

1. [Setup do Ambiente](#1-setup-do-ambiente)
2. [Rodar Gleipnir — Track A: Docker Compose](#2-rodar-gleipnir--track-a-docker-compose)
3. [Rodar Gleipnir — Track B: Terminal Multiprocesso](#3-rodar-gleipnir--track-b-terminal-multiprocesso)
4. [Suíte de Conformidade (33 testes)](#4-suíte-de-conformidade-33-testes)
5. [Cliente gRPC](#5-cliente-grpc)
6. [Guia de Contribuição](#6-guia-de-contribuição)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Setup do Ambiente

### 1.1 Instalar Go

O Gleipnir é escrito em Go. A versão mínima está no `go.mod` do repositório.
Para instalar:

```bash
# Baixar Go 1.22+ (ajuste a versão se necessário)
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.22.5.linux-amd64.tar.gz

# Adicionar ao PATH (coloque no ~/.bashrc ou ~/.zshrc para persistir)
export PATH=$PATH:/usr/local/go/bin:~/go/bin

# Verificar
go version
# Esperado: go version go1.22.5 linux/amd64
```

### 1.2 Instalar protoc e plugins

```bash
# protoc
sudo apt install -y protobuf-compiler  # Debian/Ubuntu
# ou: brew install protobuf            # macOS

# plugins Go para protoc
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Verificar
protoc --version
# Esperado: libprotoc 3.x
```

### 1.3 Clonar repositórios

```bash
mkdir -p ~/workspace
cd ~/workspace

# Repositório da especificação (3CP)
git clone git@github.com:had-nu/3CP.git

# Implementação de referência (Gleipnir)
git clone git@github.com:had-nu/gleipnir.git
```

### 1.4 Verificar tudo

```bash
echo "Go: $(go version)"
echo "protoc: $(protoc --version)"
echo "protoc-gen-go: $(which protoc-gen-go)"
echo "3CP:  $(ls ~/workspace/3CP/README.md)"
echo "Gleipnir: $(ls ~/workspace/gleipnir/go.mod)"
```

Saída esperada: nenhum "No such file or directory" e versões mostradas.

---

## 2. Rodar Gleipnir — Track A: Docker Compose

> Mais rápido e isolado. Ideal para começar.

### 2.1 Verificar Docker

```bash
docker --version
docker compose version  # notação v2
```

### 2.2 Subir a rede

```bash
cd ~/workspace/gleipnir

# Subir 5 nós mais dependências
docker compose up -d

# Aguardar ~10s para os nós formarem quórum
sleep 10
```

### 2.3 Verificar

```bash
docker compose ps
# Esperado: 5 containers com "Up" (ex: gleipnir-node-1 ... gleipnir-node-5)

docker compose logs --tail=20 gleipnir-node-1
# Esperado: logs de consenso, leader election, "block finalized"
```

### 2.4 Testar com gRPC

```bash
# Instalar grpc_cli (opcional, útil para testes manuais)
sudo apt install -y grpc-cli  # ou brew install grpc

# Verificar health
grpc_cli call localhost:50051 GetHealth ""
# Esperado: status: "running", active_peers: 5
```

### 2.5 Parar a rede

```bash
cd ~/workspace/gleipnir
docker compose down
```

---

## 3. Rodar Gleipnir — Track B: Terminal Multiprocesso

> Mais didático — você vê exatamente o que cada nó faz.

### 3.1 Build do binário

```bash
cd ~/workspace/gleipnir
go build -o bin/gleipnir ./cmd/gleipnir
# Esperado: bin/gleipnir criado
```

### 3.2 Inicializar a rede

```bash
./bin/gleipnir init --nodes=5 --output-dir=./data
# Esperado: diretório ./data/ com subpastas node-1 ... node-5 criadas
# Cada subpasta contém: identity.pem (ou .json), config.yaml, keys/
```

### 3.3 Abrir 5 terminais

Para cada terminal, execute:

**Terminal 1 — Nó 1 (bootstrap)**:
```bash
cd ~/workspace/gleipnir
./bin/gleipnir start \
  --node-dir=./data/node-1 \
  --port=50051 \
  --bootstrap
```
> Log esperado: "leader elected for cycle 1", "waiting for peers..."

**Terminal 2 — Nó 2**:
```bash
cd ~/workspace/gleipnir
./bin/gleipnir start \
  --node-dir=./data/node-2 \
  --port=50052 \
  --join=localhost:50051
```
> Log esperado: "connected to bootstrap", "syncing state", "quorum formed"

**Terminal 3 — Nó 3**:
```bash
cd ~/workspace/gleipnir
./bin/gleipnir start \
  --node-dir=./data/node-3 \
  --port=50053 \
  --join=localhost:50051
```

**Terminal 4 — Nó 4**:
```bash
cd ~/workspace/gleipnir
./bin/gleipnir start \
  --node-dir=./data/node-4 \
  --port=50054 \
  --join=localhost:50051
```

**Terminal 5 — Nó 5**:
```bash
cd ~/workspace/gleipnir
./bin/gleipnir start \
  --node-dir=./data/node-5 \
  --port=50055 \
  --join=localhost:50051
```

### 3.4 Verificar quorum

Após todos os 5 nós estarem rodando, o Terminal 1 deve mostrar:
```
[INFO] Quorum formed: 5/5 peers active
[INFO] Block #1 finalized (state root: <hash>)
[INFO] Block #2 finalized (state root: <hash>)
```

### 3.5 Testar manual com grpc_cli

```bash
# Health check
grpc_cli call localhost:50051 GetHealth ""
# Esperado: active_peers: 5

# Submeter um hash
grpc_cli call localhost:50051 SubmitHash "hash: 'abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890', submitter: 'test-device', label: 'playbook:test'"
# Esperado: ticket { id: "..." }
```

### 3.6 Parar

```bash
# Ctrl+C em cada terminal, ou:
pkill gleipnir
```

---

## 4. Suíte de Conformidade (33 testes)

### 4.1 Executar

```bash
cd ~/workspace/gleipnir

# Todos os testes (não precisa de rede rodando — usa in-memory)
go test ./tests/conformance/... -v -count=1 2>&1 | tee conformance.log
```

### 4.2 Interpretar resultado

```bash
# Ver resumo no final
tail -5 conformance.log
```

Saída esperada:
```
--- PASS: TestConformance/33_<nome> (0.01s)
PASS
ok      github.com/had-nu/gleipnir/tests/conformance  12.345s
```

### 4.3 O que cada teste cobre

A execução com `-v` mostra o nome de cada teste. Os 33 testes
cobrem (agrupados):

| Grupo | Qtde | O que valida |
|-------|------|-------------|
| Submissão | 5 | SubmitHash com hash válido, zero hash, submitter vazio, label longo, assinatura inválida |
| SMT | 6 | Inclusão, não-inclusão, prova contra root errado, leaf vazio, atualização, batch insert |
| Consenso | 8 | Leader election VRF, quorum mínimo, bloco vazio, bloco com entradas, fork recovery, rede particionada, rejoining |
| Mandates | 8 | SubmitMandate, GetMandate, validação de autoridade, versão, expiração, active set, herança sub-chain |
| gRPC | 4 | GetHealth, GetBlock, StreamBlocks, WaitForAnchor timeout |
| CBOR | 2 | Codificação canônica, decodificação de bloco |

### 4.4 Depuração

Se algum teste falhar:

```bash
# Rodar apenas o teste que falhou (copie o nome do log)
go test ./tests/conformance/... -run "TestConformance/23_" -v
```

Se falhar por pico de CPU ou timeout:
```bash
go test ./tests/conformance/... -count=1 -timeout 120s
```

---

## 5. Cliente gRPC

### 5.1 Localizar o exemplo

O Gleipnir tem um cliente de exemplo em:

```bash
ls ~/workspace/gleipnir/examples/client/
# Esperado: main.go, go.mod, README.md (ou similar)
```

Se não existir `examples/client/`, procure:

```bash
find ~/workspace/gleipnir/examples -name "*.go"
find ~/workspace/gleipnir -name "*client*" -o -name "*example*"
```

### 5.2 Entender o fluxo

O cliente faz exatamente o que a spec define na §11.1:

```
1. SubmitHash(hash, submitter, label) → ticket
2. WaitForAnchor(ticket) → AnchorProof
3. GetBlock(proof.BlockIndex) → Block
4. Verify SMT proof contra Block.StateRoot
```

**Arquivos importantes no Gleipnir**:

| Arquivo | O que contém |
|---------|-------------|
| `proto/gleipnir.proto` | Definição do serviço gRPC e mensagens |
| `pkg/client/` | Implementação do cliente gRPC (se existir) |
| `pkg/engine/` | Motor de consenso (SubmitHash → Ticket → WaitForAnchor) |
| `pkg/smt/` | Implementação da Sparse Merkle Tree |

### 5.3 Rodar o exemplo

**Pré-requisito**: Tenha a rede Gleipnir rodando (Track A ou B).

```bash
cd ~/workspace/gleipnir/examples/client

# Baixar dependências
go mod tidy

# Executar
go run main.go \
  --addr=localhost:50051 \
  --hash="abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890" \
  --submitter="playbook-test" \
  --label="demo:playbook"
```

### 5.4 Output esperado

```
[1/4] Submitting hash abcdef12... → OK (ticket: t-001)
[2/4] Waiting for anchor... → OK (block: 42)
[3/4] Fetching block 42... → OK (entries: 5)
[4/4] Verifying SMT proof... → OK (verified against StateRoot)

=== RESUMO ===
Hash:        abcdef12...
Block index: 42
StateRoot:   fedcba98...
SMT proof:   verified
Status:      ANCHORED
```

### 5.5 Verificar por um terceiro (sem contatar a rede)

Um terceiro que só tem o block pode verificar a prova SMT:

```go
// Pseudocódigo do que o cliente faz:
block := gleipnirClient.GetBlock(42)
proof := block.ExtractProof("abcdef12...")
ok := smt.VerifyProof(
    proof,           // SMT proof (path + siblings)
    block.StateRoot, // root do SMT naquele bloco
    "abcdef12...",   // hash que foi ancorado
)
// ok == true → hash existia naquele bloco
```

Essa verificação **não precisa de conexão com a rede** — só precisa do block.
Isso é o que torna o 3CP verificável por terceiros.

### 5.6 Brincar

Tente modificar algum parâmetro e ver o erro:

```bash
# Hash vazio → erro ZERO_HASH
go run main.go --hash="0000000000000000000000000000000000000000000000000000000000000000"

# Submeter sem rede rodando → erro "connection refused"
```

---

## 6. Guia de Contribuição

### 6.1 Navegar a especificação

A especificação normativa está em `~/workspace/3CP/spec/3CP.md` (891 linhas).

| Seção | Título | O que tem |
|-------|--------|-----------|
| §1 | Overview | Visão geral do protocolo |
| §2 | Transport | gRPC como transporte canônico |
| §3 | Cryptographic Primitives | BLAKE3, Dilithium3, Kyber1024, ECVRF |
| §4 | Block Structure | Formato CBOR de blocos, entradas, mandates |
| §5 | Sparse Merkle Tree | SMT com Blake3 profundidade 256 |
| §6 | Consensus | Leader election VRF + quórum M-de-N |
| §7 | gRPC API | Todos os endpoints, validação, erros |
| §8 | UID0 Identity | Tokens soulbound de identidade |
| §9 | Sub-Chains | SMT por serviço + âncoras cross-chain |
| §10 | Conformance | Obrigações de uma implementação conforme |
| §11 | Implementation Guidance | Recomendações para consumidores |
| §12 | Threat Model | Adversário, ataques, mitigações |
| §13 | Mandatory Event Anchoring | Mandatos como declarações normativas |

Toda dúvida de terminologia: `~/workspace/3CP/spec/glossary.md`.

### 6.2 Fluxo de contribuição

Conforme `CONTRIBUTING.md`:

```
1. Abrir issue descrevendo a mudança
2. Discutir com mantenedores
3. Submeter PR com as alterações
```

**Tipos de mudança**:

| Tipo | Exemplo | Precisa de consenso? |
|------|---------|---------------------|
| Editorial | Corrigir typos, formatar tabela | Não (revisão apenas) |
| Normativo | Mudar MUST/SHOULD, novo campo | Sim, com rationale |
| Segurança | Alterar primitiva criptográfica | Sim + revisão independente |

### 6.3 Checklist pré-PR

Antes de abrir um Pull Request no Gleipnir:

```bash
# 1. Formatou?
go fmt ./...

# 2. Vai passar no vet?
go vet ./...

# 3. Testes passam?
go test ./... -count=1

# 4. Conformance passa?
go test ./tests/conformance/... -count=1

# 5. Compila?
go build ./...
```

Se for mudança na **spec** (`3CP.md`):

- [ ] Usa RFC 2119 (MUST/SHOULD/MAY) corretamente?
- [ ] Inclui rationale para cada exigência normativa?
- [ ] Atualizou CDDL em `spec/schemas/` se formato mudou?
- [ ] Atualizou test vectors em `spec/examples/` se formato mudou?
- [ ] Tem Security Considerations se afeta threat model?

### 6.4 Dicas de sobrevivência

- **Leia o glossary primeiro** — evita confundir "Anchor" com "Block"
- **Teste mude uma linha por vez** — o conformance suite pega regressão rápido
- **Dúvida na spec?** `grep` é seu amigo: `grep -r "termo" ~/workspace/3CP/spec/`
- **Não entendeu um teste?** Olhe o nome: `TestConformance/07_SMT_NonInclusion` — o número é a seção da spec

---

## 7. Troubleshooting

### Erro: `go: command not found`

```bash
# Go não está no PATH
export PATH=$PATH:/usr/local/go/bin
# Adicione ao ~/.bashrc para persistir
```

### Erro: `protoc-gen-go: program not found`

```bash
# Plugins Go não estão no PATH
export PATH=$PATH:~/go/bin
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

### Erro: `connection refused` ao chamar gRPC

```bash
# 1. A rede está rodando?
docker compose ps  # Track A
pgrep gleipnir     # Track B

# 2. A porta está certa?
# Track A: localhost:50051 (nó 1)
# Track B: localhost:50051 (nó 1), localhost:50052 (nó 2), etc.

# 3. O nó já formou quorum?
docker compose logs gleipnir-node-1 | grep "Quorum"
```

### Erro: `docker compose` não encontrado

```bash
# Instalar Docker Compose v2
sudo apt install -y docker-compose-v2
# Ou usar a notação antiga: docker-compose (com hífen)
```

### Erro: `port already in use`

```bash
# Descobrir o que está na porta
sudo lsof -i :50051
# Matar o processo
kill <PID>
# Ou mudar de porta
```

### Erro: Docker container cai logo após subir

```bash
# Ver logs
docker compose logs gleipnir-node-1
# Causas comuns:
# - Porta 50051 já ocupada → mate o processo ou mude a porta
# - Falta de memória → docker compose down, docker compose up -d
# - Config incorreta → verifique docker-compose.yml
```

### Teste falhou: `Timeout`

```bash
go test ./tests/conformance/... -count=1 -timeout 120s -v
```
Se continuar falhando, rode isolado:
```bash
go test ./tests/conformance/... -run "TestConformance/15_" -v -timeout 120s
```

### Teste falhou: `SMT proof mismatch`

Causa provável: o SMT foi modificado e quebrou compatibilidade retroativa com
test vectors existentes. Verifique:

```bash
git diff HEAD~1 -- pkg/smt/
```

---

> **Pronto.** Você já sabe rodar o Gleipnir, validar conformidade, conectar um
> cliente gRPC e contribuir com o projeto. Qualquer dúvida: pergunte no canal
> do time ou abra uma issue.
