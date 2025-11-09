# Implantação para DigitalOcean (staging e production)

Este documento descreve os passos necessários para preparar o ambiente no DigitalOcean e os secrets no GitHub, e como funciona o workflow de CI/CD que está em `.github/workflows/ci-cd.yml`.

Resumo do fluxo
- Build da imagem Docker da aplicação localizada em `02-encontros-tech`.
- Push da imagem para o DigitalOcean Container Registry (DOCR).
- Deploy nos clusters Kubernetes de homologação (branch `staging`) e produção (branch `main` ou `workflow_dispatch`).

O que você precisa criar no DigitalOcean
1. Container Registry
   - Crie um registry (por exemplo: `meu-registry`) em DigitalOcean > Container Registry.
   - O valor `meu-registry` deve ser salvo em: `Settings > Secrets` do GitHub como `DOCR_NAME`.

2. Clusters Kubernetes
   - Crie dois clusters (ou use um com namespaces separados):
     - um para homologação (nome: por exemplo `encontros-staging`) 
     - um para produção (nome: por exemplo `encontros-production`)
   - Salve os nomes dos clusters nos secrets do GitHub: `STAGING_CLUSTER_NAME` e `PRODUCTION_CLUSTER_NAME`.

3. Banco Postgres
   - Recomendo usar DigitalOcean Managed Databases (Postgres) para homologação e produção.
   - Crie dois bancos/cluster ou um por ambiente.
   - Anote a connection string (ex.: `postgres://user:pass@host:port/dbname`).

4. Criar secret Kubernetes com DATABASE_URL
   - No cluster (ou namespace) execute:
     ```bash
     kubectl create secret generic encontros-tech-secrets \
       --from-literal=DATABASE_URL='postgres://user:pass@host:port/dbname' -n default
     ```
   - Ajuste o namespace (`-n`) conforme seu ambiente.

Configurar GitHub Secrets
 - `DIGITALOCEAN_ACCESS_TOKEN` — token com permissão para gerenciar registry e clusters (crie em DO: API > Tokens).
 - `DOCR_NAME` — o nome do seu registry criado no DO (apenas o sufixo; ex.: `meu-registry`).
 - `STAGING_CLUSTER_NAME` — nome do cluster de homologação.
 - `PRODUCTION_CLUSTER_NAME` — nome do cluster de produção.

Observações sobre o workflow
- O job `build` cria e envia a imagem para `registry.digitalocean.com/${DOCR_NAME}/encontros-tech:${{ github.sha }}`.
- O job `deploy-staging` é executado quando você der push na branch `staging`.
- O job `deploy-production` roda em `main` ou quando acionado manualmente e está ligado ao `environment: production` — configure proteções de ambiente no GitHub para exigir aprovação humana, se desejar.

Como testar localmente antes de usar o workflow
1. Build localmente e teste a imagem:
   ```bash
   # dentro da pasta do projeto
   docker build -t encontros-tech:local ./02-encontros-tech
   docker run -e DATABASE_URL='postgres://...' -p 8000:8000 encontros-tech:local
   ```

2. Simular deploy (substituir image no manifest):
   ```bash
   IMAGE=meu-registry/encontros-tech:local envsubst < 02-encontros-tech/k8s/manifests.yaml > rendered.yaml
   kubectl apply -f rendered.yaml
   ```

Passos adicionais recomendados
- Configure o namespace e quotas de recursos no Kubernetes.
- Proteja o `environment: production` no GitHub (Settings → Environments) e adicione revisores.
- Configure readiness/liveness probes no manifest para produção.

Se quiser, eu posso:
- adaptar os manifests para Helm ou Kustomize (bom para variáveis por ambiente),
- adicionar health/readiness probes,
- criar um job que atualize automaticamente um Secret Kubernetes com a connection string do Managed DB (via `doctl`),
- ou criar um template de `values.yaml` para Helm.
