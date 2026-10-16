# ansible-examples

Exemplos de playbooks prontos para serem carregados como projeto no
**Ansible Automation Platform (AAP)**.

## Estrutura

```
.
├── collections/
│   └── requirements.yml          # coleções instaladas pelo AAP na sincronização
├── playbooks/
│   └── openshift/
│       ├── create_project.yml    # CreateProject
│       ├── deploy_httpd.yml      # DeployHttpd
│       ├── vars/
│       │   └── catalog.yml       # configuração de GitHub e Developer Hub
│       └── tasks/                # tarefas compartilhadas
│           ├── check_credentials.yml
│           ├── github_repo.yml   # localiza/cria o repositório do projeto
│           ├── catalog_upsert.yml  # grava o catalog-info.yaml e registra no RHDH
│           └── http_request.yml  # chamada HTTP com token sem expor no log
└── aap/
    ├── credential_types/
    │   └── github_rhdh_catalog.yml  # credencial customizada GitHub + RHDH
    └── surveys/                  # surveys de exemplo para os Job Templates
        ├── openshift_create_project.json
        └── openshift_deploy_httpd.json
```

## Configurando no AAP

1. **Projeto** – crie um Project com *Source Control Type* = Git apontando para
   `https://github.com/victorfim/ansible-examples.git`. Na sincronização o AAP
   instala as coleções de `collections/requirements.yml`.
2. **Credencial** – crie as credenciais exigidas pelo playbook (veja abaixo).
3. **Job Template** – escolha o projeto, o playbook, um inventário (pode ser um
   inventário vazio; os playbooks rodam em `localhost`) e a credencial.
4. **Survey** (opcional) – use o JSON correspondente em `aap/surveys/` como
   referência para cadastrar as perguntas.

### Credencial e Execution Environment (OpenShift)

**Credencial:** tipo *OpenShift or Kubernetes API Bearer Token*, com a URL da
API do cluster (ex.: `https://api.cluster.exemplo.com:6443`) e o token de um
usuário ou service account com permissão para criar projetos
(`self-provisioner` ou superior). A mesma credencial serve para os dois
playbooks.

**Credencial do catálogo:** crie o Credential Type descrito em
`aap/credential_types/github_rhdh_catalog.yml` e uma credencial desse tipo, e
associe-a aos dois Job Templates junto com a do OpenShift. Veja
[Catálogo do Developer Hub](#catálogo-do-developer-hub).

**Execution Environment:** precisa da coleção `kubernetes.core` e da biblioteca
Python `kubernetes`. O EE suportado padrão do AAP (`ee-supported-rhel9`) já
atende.

## Playbooks

### CreateProject – `playbooks/openshift/create_project.yml`

Cria um projeto vazio no OpenShift (equivalente a `oc new-project`). Se o
projeto já existir, nada é alterado. Em seguida cria o repositório GitHub
`<github_owner>/<project_name>` com um `catalog-info.yaml` que descreve o
projeto como um **System** e registra esse arquivo no Developer Hub.

| Variável               | Obrigatória | Padrão | Descrição                              |
|------------------------|-------------|--------|----------------------------------------|
| `project_name`         | sim         |        | Nome do projeto (padrão DNS, até 63 caracteres) |
| `project_display_name` | não         | `""`   | Nome de exibição no console            |
| `project_description`  | não         | `""`   | Descrição do projeto                   |
| `catalog_owner`        | não         | `group:default/plataforma` | Dono do System no catálogo |
| `github_repo_visibility` | não       | `private` | `private` ou `public`               |

Publica `project_name`, `github_repo_url` e `catalog_entity_ref` com
`set_stats`.

### DeployHttpd – `playbooks/openshift/deploy_httpd.yml`

Implanta uma aplicação httpd de exemplo num projeto **que já existe**:

- **Deployment** com a imagem `httpd` do registry interno do cluster
  (`image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest`)
- **Service** na porta 8080
- **Route** HTTPS (TLS edge, com redirecionamento de HTTP para HTTPS)

O playbook espera o Deployment ficar disponível e mostra a URL no final. Depois
adiciona um **Component** `<project_name>-<app_name>` ao `catalog-info.yaml`
do repositório do projeto (ligado ao System do projeto) e pede ao Developer Hub
para relê-lo. Os recursos recebem o label `backstage.io/kubernetes-id`, que
liga o Component às abas Kubernetes e Topology. Tudo é idempotente: rodar de
novo não duplica nada.

O repositório do projeto precisa existir (é criado pelo CreateProject); se não
existir, o playbook falha antes de mexer no cluster.

| Variável       | Obrigatória | Padrão  | Descrição                              |
|----------------|-------------|---------|----------------------------------------|
| `project_name` | sim         |         | Projeto de destino (precisa existir)   |
| `app_name`     | não         | `httpd` | Nome do Deployment, Service e Route    |
| `app_image`    | não         | imagem `httpd` do registry interno | Imagem do container |
| `catalog_owner` | não        | `group:default/plataforma` | Dono do Component no catálogo |

Publica `project_name`, `app_url` e `catalog_entity_ref` com `set_stats`.

A imagem padrão vem da ImageStream `httpd` do namespace `openshift`, criada
pelo Samples Operator. Se ela não existir no seu cluster, informe outra imagem
em `app_image`.

### Encadeando os dois num Workflow

Crie um Workflow Template com o nó **CreateProject** seguido de
**DeployHttpd** (na saída *On Success*). Como o CreateProject publica
`project_name` com `set_stats`, o DeployHttpd recebe o valor automaticamente.
Coloque o survey no Workflow (com `project_name` e, se quiser, `app_name`) em
vez de nos Job Templates individuais.

## Catálogo do Developer Hub

Depois dos dois playbooks, o repositório `<github_owner>/<project_name>` tem
um `catalog-info.yaml` assim:

```yaml
# Gerado pelo Ansible Automation Platform (ansible-examples).
# Entidades com o mesmo kind e nome são atualizadas a cada execução.
---
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: meu-projeto
  # ...
spec:
  owner: group:default/plataforma
---
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: meu-projeto-httpd
  annotations:
    backstage.io/kubernetes-id: meu-projeto-httpd
    backstage.io/kubernetes-namespace: meu-projeto
    # ...
spec:
  type: service
  owner: group:default/plataforma
  system: meu-projeto
```

Cada execução atualiza a entidade de mesmo kind e nome e preserva as outras.
Chaves que só existem no arquivo (por exemplo, uma anotação adicionada à mão)
são mantidas; listas geradas pelo playbook, como `tags` e `links`, são
substituídas. O CreateProject registra o arquivo como location
(`POST /api/catalog/locations`; 409 = já registrado). O DeployHttpd grava o
Component e chama `POST /api/catalog/refresh` no System, o que faz o RHDH
reler o arquivo sem esperar o próximo ciclo.

### Variáveis (`playbooks/openshift/vars/catalog.yml`)

| Variável | Padrão | Descrição |
|---|---|---|
| `catalog_enabled` | `true` | `false` desliga toda a parte de GitHub e catálogo |
| `github_api_url` | env `GITHUB_API_URL` ou `https://api.github.com` | API do GitHub (GHES: `https://<host>/api/v3`) |
| `github_owner` | env `GITHUB_OWNER` | Organização (ou usuário dono do token) |
| `github_token` | env `GITHUB_TOKEN` | Token do GitHub |
| `github_repo_visibility` | `private` | Visibilidade do repositório criado |
| `rhdh_url` | env `RHDH_URL` | URL do Developer Hub; vazio = não registra via API |
| `rhdh_token` | env `RHDH_TOKEN` | Token de acesso externo do RHDH |
| `rhdh_validate_certs` | `true` | Validar o certificado do RHDH |
| `catalog_owner` | `group:default/plataforma` | Dono das entidades |
| `catalog_lifecycle` | `experimental` | `spec.lifecycle` do Component |

### Token do GitHub

Fine-grained PAT (ou GitHub App) na organização com **Administration:
read/write** (criar repositórios) e **Contents: read/write** (gravar o
`catalog-info.yaml`). Se `github_owner` for um usuário e não uma organização,
o token precisa ser desse usuário.

### Configuração do Developer Hub

```yaml
# app-config (ConfigMap do RHDH)
backend:
  auth:
    externalAccess:
      - type: static
        options:
          token: ${AAP_CATALOG_TOKEN}   # o mesmo valor de rhdh_token
          subject: aap-catalog
        accessRestrictions:
          - plugin: catalog

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}   # precisa ler os repositórios privados criados

catalog:
  rules:
    - allow: [Component, System, API, Resource, Location, Template, Group, User]
```

- **`catalog.rules`** precisa permitir `System` e `Component`; sem isso as
  entidades são descartadas em silêncio.
- **`catalog_owner`** precisa ser um usuário ou grupo existente no catálogo. Com
  RHBK, use um grupo importado pelo provider do Keycloak.
- **RBAC:** se o registro ou o refresh retornar 403, verifique se a política
  permite `catalog.location.create` e `catalog.entity.refresh` para o principal
  do token.
- **Sem acesso de rede do AAP ao RHDH:** deixe `rhdh_url` vazio e configure o
  discovery provider do GitHub (`catalog.providers.github`, com
  `catalogPath: /catalog-info.yaml`) para a organização. O RHDH encontra os
  repositórios novos no próximo ciclo do `schedule`.

### Link de volta no template do RHDH

O job publica `catalog_entity_ref` com `set_stats`. Se o template do RHDH lê
os artifacts do job, use esse valor em `output.links` (com `entityRef`) para
mostrar o botão "Abrir no catálogo" ao final da execução.

## Execução local para teste

```bash
ansible-galaxy collection install -r collections/requirements.yml
export K8S_AUTH_HOST=https://api.cluster.exemplo.com:6443
export K8S_AUTH_API_KEY=$(oc whoami -t)
export GITHUB_OWNER=minha-org GITHUB_TOKEN=github_pat_...
export RHDH_URL=https://developer-hub.apps.cluster.exemplo.com RHDH_TOKEN=...
ansible-playbook playbooks/openshift/create_project.yml -e project_name=meu-projeto
ansible-playbook playbooks/openshift/deploy_httpd.yml   -e project_name=meu-projeto
```
