# Pipeline de Entrega (Docker + Orca) — pacote de exemplo

Este pacote contém os 3 artefatos que você pediu, organizados em duas pastas
que representam dois repositórios diferentes:

```
repo-central/                          <- vai para o REPOSITÓRIO PRINCIPAL de workflows
  .github/workflows/
    reusable-delivery-pipeline.yml     <- o workflow reutilizável em si

repo-aplicacao-exemplo/                <- representa um repositório de APLICAÇÃO qualquer
  .github/workflows/
    caller-delivery.yml                <- workflow que CHAMA o reusable acima
  artifacts/
    app.jar                            <- artefato dummy (2KB) só para teste
  dockerfiles-teste/
    Dockerfile.valido                  <- cenário que deve PASSAR
    Dockerfile.invalido-mvn            <- cenário que deve ser BLOQUEADO (mvn)
    Dockerfile.invalido-npm            <- cenário que deve ser BLOQUEADO (npm)
```

## 1. Onde colocar cada arquivo

- `repo-central/.github/workflows/reusable-delivery-pipeline.yml` → copie para o
  repositório principal onde já ficam os workflows do dia a dia (o mesmo do
  `b3-reusable-workflows`).
- `repo-central/.github/workflows/reusable-delivery-pipeline-teste.yml` →
  **versão enxuta**, sem Orca e sem publicação/deploy (Release ou Nexus).
  Use essa primeiro para validar só o bloqueio do Dockerfile — ver seção 5.
- `repo-aplicacao-exemplo/.github/workflows/caller-delivery.yml` → caller da
  versão **completa** (com Orca + Release). Copie para o repositório de uma
  aplicação real. **Ajuste a linha `uses:`** trocando
  `B3/repo-central-workflows` pelo nome real do repositório principal e a
  branch/tag (`@main`, ou uma tag fixa tipo `@v1` se preferirem versionar).
- `repo-aplicacao-exemplo/.github/workflows/caller-delivery-teste.yml` →
  caller da **versão de teste** (aponta para `reusable-delivery-pipeline-teste.yml`).
  Use só um dos dois callers de cada vez no mesmo repositório, pra não rodar
  os dois pipelines em paralelo no mesmo push/PR.

## 2. Como testar cada critério de aceitação

### Teste A — Dockerfile válido (deve passar)
```bash
cp dockerfiles-teste/Dockerfile.valido Dockerfile
git add Dockerfile artifacts/app.jar
git commit -m "teste: cenario valido"
git push
```
Esperado: job `validate-dockerfile` passa e o pipeline segue até
`build-scan-release`.

### Teste B — Dockerfile com `mvn` (deve bloquear)
```bash
cp dockerfiles-teste/Dockerfile.invalido-mvn Dockerfile
git add Dockerfile
git commit -m "teste: cenario invalido mvn"
git push
```
Esperado: job `validate-dockerfile` falha com a mensagem:
> Você está fazendo build de uma aplicação. Utilize outro fluxo, esse fluxo é
> somente para entrega de arquivos / imagens de terceiro.

### Teste C — Dockerfile com `npm` (deve bloquear)
Mesma lógica do Teste B, usando `Dockerfile.invalido-npm`. Serve para provar
que a regra não é hardcoded só pra Maven — cobre outras stacks também.

### Teste D — Artefato maior que 50MB (deve bloquear a origem "repo")
```bash
# gera um arquivo dummy de 60MB só para o teste
head -c 60000000 /dev/urandom > artifacts/app.jar
git add artifacts/app.jar
git commit -m "teste: artefato maior que 50MB"
git push
```
Esperado: job `resolve-artifact` falha porque o artefato está no repo mas
excede `max-size-mb: 50`.

### Teste E — Artefato vindo do Nexus (source=nexus)
```bash
git rm artifacts/app.jar
git commit -m "teste: forcar busca no Nexus"
git push
```
Esperado: job `resolve-artifact` não encontra o arquivo local, define
`source=nexus` e baixa via `curl` usando a `nexus-artifact-url` configurada
no `caller-delivery.yml`. Se as credenciais `NEXUS_USER`/`NEXUS_PASS` não
estiverem configuradas como secrets do repositório, esse passo falha —
é esperado até você configurar os secrets reais.

## 3. Secrets necessários no repositório da aplicação

Configure em **Settings → Secrets and variables → Actions**:
- `NEXUS_USER` / `NEXUS_PASS` (só necessários se for usar artefato do Nexus)
- `ORCA_API_TOKEN` (obrigatório, usado no scan da imagem)

## 4. Versão de teste (sem Orca, sem publicação/deploy)

Enquanto você só quer confirmar que o **bloqueio do Dockerfile** funciona,
use o par:

- `repo-central/.github/workflows/reusable-delivery-pipeline-teste.yml`
- `repo-aplicacao-exemplo/.github/workflows/caller-delivery-teste.yml`

Essa versão mantém os jobs `validate-dockerfile` (o bloqueio em si) e
`resolve-artifact` (regra dos 50MB / origem repo x Nexus, já que faz parte
da mesma validação de entrada), mas o job final só faz `docker build` para
confirmar que a imagem sobe — **sem Orca, sem `docker save`, sem Release e
sem qualquer push para Nexus ou outro registry**.

Rode os Testes A, B e C da seção 2 normalmente com esse caller — o
comportamento de bloqueio é idêntico ao da versão completa, só sem o custo
de rodar o scan e publicar artefato a cada tentativa.

### Como migrar para a versão completa depois

Quando o bloqueio estiver validado, no repositório da aplicação:

1. Troque, no arquivo caller, `reusable-delivery-pipeline-teste.yml` por
   `reusable-delivery-pipeline.yml` (ou simplesmente passe a usar o
   `caller-delivery.yml` no lugar do `caller-delivery-teste.yml`).
2. Adicione o secret `ORCA_API_TOKEN` no repositório.
3. Pronto — não precisa alterar mais nada, os inputs (`artifact-path`,
   `nexus-artifact-url`, `max-size-mb` etc.) são os mesmos nas duas versões.

## 5. Observação sobre o Orca

O comando `orca-cli image scan ...` usado no workflow reutilizável é um
exemplo da sintaxe mais comum. Confirme com o time de segurança de vocês o
comando exato / flags configuradas na conta Orca da organização antes de
colocar isso em produção — a lógica (escanear a imagem recém-buildada e
falhar em vulnerabilidades críticas/altas) está correta, só a sintaxe pode
precisar de ajuste fino.
