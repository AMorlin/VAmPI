# Infra local — DefectDojo + Dependency-Track

Este diretório contém os `docker-compose.yml` que a pipeline (`.github/workflows/devsecops.yml`)
usa para subir o DefectDojo e o Dependency-Track como infraestrutura efêmera dentro de cada job do
GitHub Actions. Este README é pra quem quiser subir a mesma infra **manualmente, na própria
máquina**, depois de clonar o repositório — seja pra inspecionar os achados pela interface web,
seja pra depurar a pipeline localmente antes de dar push.

Todo comando abaixo tem duas versões — **Bash** (Linux/macOS/Git Bash) e **PowerShell**
(Windows). Use a que corresponde ao seu terminal.

## Pré-requisitos

- Docker + Docker Compose (`docker compose version`)
- `curl` — no Windows, os exemplos chamam `curl.exe` explicitamente pra não cair no alias
  `Invoke-WebRequest` que o PowerShell registra por padrão pro nome `curl`
- Pra gerar o SBOM manualmente: Node.js (`npx`)
- Pra rodar o Semgrep manualmente: o próprio Docker já resolve (`docker run returntocorp/semgrep`)

## Clonar

```bash
git clone https://github.com/AMorlin/VAmPI.git
cd VAmPI
```

## DefectDojo

```bash
docker compose -f infra/defectdojo/docker-compose.yml up -d
```

Sobe em **http://localhost:9090**. Leva de 1 a 3 minutos pra ficar pronto (migrações do Django +
criação do usuário admin). Acompanhe com:

**Bash**
```bash
until curl -sf -o /dev/null http://localhost:9090/login; do sleep 5; done && echo "pronto"
```

**PowerShell**
```powershell
do {
  Start-Sleep -Seconds 5
  curl.exe -sf -o NUL http://localhost:9090/login
} while ($LASTEXITCODE -ne 0)
Write-Host "pronto"
```

Login: usuário `admin`, senha `ci-automation-only` (fixa no `infra/defectdojo/docker-compose.yml`
— ambiente local/efêmero, não é um segredo real).

### Importar um scan do Semgrep manualmente

**Bash**
```bash
# gera o JSON do Semgrep
docker run --rm -v "$PWD:/src" returntocorp/semgrep:latest \
  semgrep scan --config auto --json --output /src/semgrep-results.json /src

# pega um token de API
TOKEN=$(curl -s -X POST http://localhost:9090/api/v2/api-token-auth/ \
  -d "username=admin" -d "password=ci-automation-only" | jq -r .token)

# importa os achados (cria produto + engagement automaticamente)
curl -X POST http://localhost:9090/api/v2/import-scan/ \
  -H "Authorization: Token $TOKEN" \
  -F "scan_type=Semgrep JSON Report" \
  -F "file=@semgrep-results.json" \
  -F "product_type_name=DevSecOps Trabalho Final" \
  -F "product_name=VAmPI" \
  -F "engagement_name=local" \
  -F "auto_create_context=true"
```

**PowerShell**
```powershell
# gera o JSON do Semgrep — ${PWD} com chaves, senão o ":" que vem depois quebra o parser
docker run --rm -v "${PWD}:/src" returntocorp/semgrep:latest `
  semgrep scan --config auto --json --output /src/semgrep-results.json /src

# pega um token de API
$response = curl.exe -s -X POST http://localhost:9090/api/v2/api-token-auth/ `
  -d "username=admin" -d "password=ci-automation-only" | ConvertFrom-Json
$TOKEN = $response.token

# importa os achados (cria produto + engagement automaticamente)
curl.exe -X POST http://localhost:9090/api/v2/import-scan/ `
  -H "Authorization: Token $TOKEN" `
  -F "scan_type=Semgrep JSON Report" `
  -F "file=@semgrep-results.json" `
  -F "product_type_name=DevSecOps Trabalho Final" `
  -F "product_name=VAmPI" `
  -F "engagement_name=local" `
  -F "auto_create_context=true"
```

Os achados aparecem em **Products → VAmPI → local** na interface.

Derrubar (apaga todos os dados, é um ambiente descartável):

```bash
docker compose -f infra/defectdojo/docker-compose.yml down -v
```

## Dependency-Track

```bash
docker compose -f infra/dependency-track/docker-compose.yml up -d
```

Sobe em **http://localhost:8080** (API) e **http://localhost:8081** (interface web). Espere ficar
de pé:

**Bash**
```bash
until curl -sf http://localhost:8080/api/version >/dev/null; do sleep 5; done && echo "pronto"
```

**PowerShell**
```powershell
do {
  Start-Sleep -Seconds 5
  curl.exe -sf -o NUL http://localhost:8080/api/version
} while ($LASTEXITCODE -ne 0)
Write-Host "pronto"
```

### Por que precisa de um passo extra aqui

Por padrão essa imagem (linha clássica 4.14.4, escolhida de propósito — ver nota no topo do
`infra/dependency-track/docker-compose.yml`) tenta espelhar o NVD, que só casa vulnerabilidade por
CPE e nunca vai achar nada nos componentes Python que o `cdxgen` gera (só tem `purl`). Então, antes
de subir o SBOM, é preciso trocar a fonte de vulnerabilidades pra uma que case por `purl`:

**Bash**
```bash
# troca a senha inicial (obrigatório no primeiro login)
curl -X POST http://localhost:8080/api/v1/user/forceChangePassword \
  --data-urlencode "username=admin" --data-urlencode "password=admin" \
  --data-urlencode "newPassword=ci-automation-only" --data-urlencode "confirmPassword=ci-automation-only"

TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/user/login \
  --data-urlencode "username=admin" --data-urlencode "password=ci-automation-only")

# desliga o NVD (inútil aqui) e liga o OSV pro ecossistema PyPI (casa por purl, sem credencial)
curl -X POST http://localhost:8080/api/v1/configProperty/aggregate \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '[
    {"groupName":"vuln-source","propertyName":"nvd.enabled","propertyValue":"false"},
    {"groupName":"vuln-source","propertyName":"google.osv.enabled","propertyValue":"PyPI"}
  ]'

# (opcional) liga também o OSS Index, se você tiver conta em ossindex.sonatype.org
curl -X POST http://localhost:8080/api/v1/configProperty/aggregate \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '[
    {"groupName":"scanner","propertyName":"ossindex.api.username","propertyValue":"SEU_USERNAME"},
    {"groupName":"scanner","propertyName":"ossindex.api.token","propertyValue":"SEU_TOKEN"}
  ]'

# a config só é lida no próximo boot do apiserver — reinicia pra aplicar
docker compose -f infra/dependency-track/docker-compose.yml restart apiserver
until curl -sf http://localhost:8080/api/version >/dev/null; do sleep 5; done
```

**PowerShell**
```powershell
# troca a senha inicial (obrigatório no primeiro login)
curl.exe -X POST http://localhost:8080/api/v1/user/forceChangePassword `
  --data-urlencode "username=admin" --data-urlencode "password=admin" `
  --data-urlencode "newPassword=ci-automation-only" --data-urlencode "confirmPassword=ci-automation-only"

$TOKEN = curl.exe -s -X POST http://localhost:8080/api/v1/user/login `
  --data-urlencode "username=admin" --data-urlencode "password=ci-automation-only"

# desliga o NVD (inútil aqui) e liga o OSV pro ecossistema PyPI (casa por purl, sem credencial)
$body = @'
[
  {"groupName":"vuln-source","propertyName":"nvd.enabled","propertyValue":"false"},
  {"groupName":"vuln-source","propertyName":"google.osv.enabled","propertyValue":"PyPI"}
]
'@
curl.exe -X POST http://localhost:8080/api/v1/configProperty/aggregate `
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d $body

# (opcional) liga também o OSS Index, se você tiver conta em ossindex.sonatype.org
$ossIndexBody = @'
[
  {"groupName":"scanner","propertyName":"ossindex.api.username","propertyValue":"SEU_USERNAME"},
  {"groupName":"scanner","propertyName":"ossindex.api.token","propertyValue":"SEU_TOKEN"}
]
'@
curl.exe -X POST http://localhost:8080/api/v1/configProperty/aggregate `
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d $ossIndexBody

# a config só é lida no próximo boot do apiserver — reinicia pra aplicar
docker compose -f infra/dependency-track/docker-compose.yml restart apiserver
do {
  Start-Sleep -Seconds 5
  curl.exe -sf -o NUL http://localhost:8080/api/version
} while ($LASTEXITCODE -ne 0)
```

> No `@'...'@` (here-string do PowerShell), a linha de fechamento `'@` **tem que** começar na
> coluna 0 — sem espaços/indentação antes dela, senão dá erro de parsing.

Depois do restart, o mirror do OSV começa sozinho. Acompanhe o progresso (leva uns 4-5 minutos
pra espelhar o ecossistema PyPI inteiro, ~25 mil advisories):

**Bash**
```bash
docker compose -f infra/dependency-track/docker-compose.yml logs -f apiserver | grep -i osv
# espera aparecer a linha: "Google OSV mirroring complete"
```

**PowerShell**
```powershell
docker compose -f infra/dependency-track/docker-compose.yml logs -f apiserver | Select-String -Pattern "osv" -CaseSensitive:$false
# espera aparecer a linha: "Google OSV mirroring complete"
```

### Enviar um SBOM

**Bash**
```bash
npx --yes @cyclonedx/cdxgen@latest -o bom.json -t python .

curl -X POST http://localhost:8080/api/v1/bom \
  -H "Authorization: Bearer $TOKEN" \
  -F "autoCreate=true" -F "projectName=VAmPI" -F "projectVersion=local" \
  -F "bom=@bom.json;type=application/json"
```

**PowerShell**
```powershell
npx --yes '@cyclonedx/cdxgen@latest' -o bom.json -t python .

curl.exe -X POST http://localhost:8080/api/v1/bom `
  -H "Authorization: Bearer $TOKEN" `
  -F "autoCreate=true" -F "projectName=VAmPI" -F "projectVersion=local" `
  -F "bom=@bom.json;type=application/json"
```

> **Se o seu caminho local tiver espaço** (ex.: repositório dentro do OneDrive, como
> `...\14. Desenvolvimento de Software Seguro com DevSecOps\VAmPI`), o `npx cdxgen` pode falhar
> tentando resolver as dependências Python — o `pip`, chamado por baixo dos panos, interpreta um
> pedaço do caminho como se fosse um nome de pacote (erro tipo `Invalid requirement: 'DevOps\14.'`).
> Rode o `cdxgen` dentro de um container em vez de via `npx` — daí o caminho vira só `/app` dentro
> do Linux do container, sem espaço nenhum, e o bug some:
>
> ```powershell
> docker run --rm -v "${PWD}:/app" ghcr.io/cyclonedx/cdxgen:master -o /app/bom.json -t python /app
> ```
>
> (Bash: `docker run --rm -v "$PWD:/app" ghcr.io/cyclonedx/cdxgen:master -o /app/bom.json -t python /app`)

Os achados aparecem em **http://localhost:8081** → Projects → VAmPI → local → Vulnerabilities, ou
via API:

**Bash**
```bash
PROJECT_UUID=$(curl -s "http://localhost:8080/api/v1/project/lookup?name=VAmPI&version=local" \
  -H "Authorization: Bearer $TOKEN" | jq -r .uuid)
curl -s "http://localhost:8080/api/v1/finding/project/$PROJECT_UUID" \
  -H "Authorization: Bearer $TOKEN" | jq '[.[] | {component: .component.name, version: .component.version, vulnerability: .vulnerability.vulnId, severity: .vulnerability.severity}]'
```

**PowerShell**
```powershell
$lookup = curl.exe -s "http://localhost:8080/api/v1/project/lookup?name=VAmPI&version=local" `
  -H "Authorization: Bearer $TOKEN" | ConvertFrom-Json
$PROJECT_UUID = $lookup.uuid

$findings = curl.exe -s "http://localhost:8080/api/v1/finding/project/$PROJECT_UUID" `
  -H "Authorization: Bearer $TOKEN" | ConvertFrom-Json
$findings | ForEach-Object {
  [PSCustomObject]@{
    component     = $_.component.name
    version       = $_.component.version
    vulnerability = $_.vulnerability.vulnId
    severity      = $_.vulnerability.severity
  }
} | Format-Table
```

Derrubar (apaga todos os dados, é um ambiente descartável):

```bash
docker compose -f infra/dependency-track/docker-compose.yml down -v
```

## Rodar as duas infras ao mesmo tempo

As portas não colidem (DefectDojo em 9090, Dependency-Track em 8080/8081), então dá pra subir as
duas juntas sem problema — é só rodar os dois `docker compose up -d` acima.
