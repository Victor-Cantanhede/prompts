Quero que você atue como um **engenheiro DevOps/SRE sênior, especialista em GitHub Actions, Docker, CI/CD, monorepos e Portainer**, e faça uma análise completa deste projeto antes de modificar qualquer arquivo.

## Objetivo

Preciso implementar um fluxo de **CI/CD robusto, seguro, simples de manter e otimizado** para este monorepo.

A arquitetura atual possui múltiplas aplicações dentro de `apps/`, atualmente com pelo menos:

* `apps/backend`
* `apps/frontend`

Cada aplicação possui suas próprias dependências e deve ser tratada como uma unidade independente de build, teste, criação de imagem Docker e deploy.

Também existem Dockerfiles específicos para as aplicações e um `docker-compose.prod.yml` utilizado no ambiente de produção/Portainer.

O deploy de produção é feito através do **Portainer**.

O objetivo é que o GitHub Actions seja responsável pelo **CI + build + publicação das imagens**, enquanto o Portainer seja responsável pelo **deployment dos containers/Stack**.

---

# 1. PRIMEIRO: ANALISE O PROJETO

Antes de criar ou alterar qualquer arquivo, faça uma análise detalhada do repositório.

Não assuma uma estrutura genérica.

Inspecione, no mínimo:

* estrutura completa do monorepo;
* `apps/backend/package.json`;
* `apps/backend/package-lock.json`;
* `apps/frontend/package.json`;
* `apps/frontend/package-lock.json`;
* `docker/backend.Dockerfile`;
* `docker/frontend.Dockerfile`;
* `docker-compose.prod.yml`;
* `docker-compose.dev.yml`;
* `.dockerignore`;
* `.gitignore`;
* configurações de build;
* configurações de testes;
* configurações de TypeScript;
* configuração do NestJS;
* configuração do Next.js;
* variáveis de ambiente;
* dependências entre backend/frontend;
* qualquer infraestrutura ou configuração existente relacionada a CI/CD;
* diretório `.github`, caso exista.

Procure também por:

* scripts `build`;
* scripts `test`;
* scripts `lint`;
* scripts `start`;
* scripts `start:prod`;
* migrations;
* Prisma;
* processos que precisam ocorrer antes do backend iniciar;
* healthchecks;
* portas;
* volumes;
* networks;
* secrets;
* dependências externas;
* Redis/RabbitMQ/PostgreSQL/etc., caso existam;
* qualquer serviço que faça parte da Stack de produção.

**Não altere nada ainda.**

Primeiro apresente sua análise da arquitetura atual e os problemas/riscos encontrados.

---

# 2. ARQUITETURA DESEJADA

Quero chegar a uma arquitetura conceitualmente semelhante a:

```text
                         GitHub
                           │
                           │ push / merge na main
                           ▼
                   GitHub Actions
                           │
              ┌────────────┴────────────┐
              │                         │
        Backend mudou?            Frontend mudou?
              │                         │
              ▼                         ▼
       Backend CI                  Frontend CI
              │                         │
       npm ci / tests              npm ci / build
              │                         │
              ▼                         ▼
       Docker build                Docker build
              │                         │
              ▼                         ▼
       Container Registry       Container Registry
              │                         │
              └────────────┬────────────┘
                           │
                           ▼
                       Portainer
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
          Backend                    Frontend
```

O princípio fundamental é:

> **cada aplicação deve poder ser construída, versionada e publicada independentemente das outras aplicações do monorepo.**

Por exemplo:

Se um commit alterar somente:

```text
apps/backend/**
```

não quero reconstruir nem publicar desnecessariamente a imagem do frontend.

Se alterar somente:

```text
apps/frontend/**
```

não quero reconstruir nem publicar desnecessariamente a imagem do backend.

Se uma alteração afetar infraestrutura compartilhada, como:

```text
docker/backend.Dockerfile
docker/frontend.Dockerfile
docker-compose.prod.yml
```

analise cuidadosamente quais aplicações realmente precisam ser reconstruídas/redeployadas.

Não implemente regras cegamente: determine isso a partir da arquitetura real do projeto.

---

# 3. GITHUB ACTIONS

Projete o melhor workflow possível para este cenário.

Você pode utilizar:

* workflows separados por aplicação;
* workflow único com jobs independentes;
* `paths`;
* `dorny/paths-filter`;
* matrix;
* reusable workflows;
* composite actions;

ou uma combinação dessas técnicas.

Escolha a arquitetura que considerar mais adequada e explique o motivo.

Priorize:

1. simplicidade;
2. confiabilidade;
3. segurança;
4. velocidade;
5. cache eficiente;
6. facilidade de manutenção;
7. independência entre aplicações;
8. facilidade de rollback;
9. baixo acoplamento;
10. observabilidade do pipeline.

---

# 4. CI DE CADA APLICAÇÃO

Cada aplicação deve possuir seu próprio ciclo de CI.

Para o backend, considere algo semelhante a:

```text
checkout
→ setup Node
→ npm ci
→ lint
→ unit tests
→ build
→ eventualmente e2e
→ docker build
→ docker push
```

Para o frontend:

```text
checkout
→ setup Node
→ npm ci
→ lint
→ testes, se existentes
→ build
→ docker build
→ docker push
```

Mas **não copie cegamente essa sequência**.

Analise os `package.json` reais e determine quais scripts existem e qual é a sequência correta para este projeto.

Use `npm ci` quando apropriado, respeitando os lockfiles existentes.

Não introduza pnpm, yarn, Turborepo, Nx ou qualquer outro gerenciador/framework de monorepo sem necessidade.

---

# 5. DOCKER

Analise profundamente os Dockerfiles existentes antes de modificá-los.

Quero aproveitar ao máximo a estrutura atual.

Avalie:

* multi-stage builds;
* tamanho final da imagem;
* dependências de runtime;
* `npm ci`;
* `npm prune`;
* cache de layers;
* BuildKit;
* Docker Buildx;
* `.dockerignore`;
* segurança;
* usuário não-root;
* healthcheck;
* variáveis de ambiente;
* arquivos necessários em runtime.

Não reescreva os Dockerfiles apenas por preferência pessoal.

Somente altere-os se houver um ganho técnico claro e justifique a alteração.

---

# 6. CONTAINER REGISTRY

Escolha o registry mais adequado para este projeto.

Por padrão, considere **GitHub Container Registry (GHCR)**, caso não exista uma razão concreta para utilizar outro registry.

As imagens devem ser independentes.

Por exemplo:

```text
ghcr.io/<owner>/<repository>-backend
ghcr.io/<owner>/<repository>-frontend
```

ou outra convenção que você considere melhor.

Defina uma estratégia de tags adequada.

Quero evitar depender exclusivamente de:

```text
latest
```

Prefira tags imutáveis baseadas no commit/versão, por exemplo:

```text
sha-<commit>
```

e, se fizer sentido:

```text
latest
```

como alias da versão atual de produção.

Analise também se tags como:

```text
main
production
v1.2.3
```

fazem sentido.

O objetivo é permitir rollback seguro.

---

# 7. PORTAINER

Este é um dos pontos mais importantes.

O ambiente de produção utiliza **Portainer**.

Quero que você projete a integração GitHub Actions → Registry → Portainer da forma mais correta possível.

Avalie as opções disponíveis, principalmente:

### Opção preferencial

```text
GitHub Actions
      ↓
build image
      ↓
push image
      ↓
Portainer webhook
      ↓
Portainer atualiza a Stack
      ↓
docker compose pull
      ↓
containers recriados
```

Analise se o **Webhook do Portainer** é a melhor solução neste cenário.

Caso exista uma alternativa tecnicamente superior para este projeto, explique-a antes de implementar.

Não quero que o GitHub Actions faça SSH diretamente no servidor se isso puder ser evitado.

O Portainer deve continuar sendo o ponto responsável pelo deployment.

---

# 8. PORTAINER STACK

Analise o `docker-compose.prod.yml` existente e determine como ele deve funcionar como Stack no Portainer.

Quero uma configuração em que as imagens sejam referenciadas pelo registry, por exemplo:

```yaml
services:
  backend:
    image: ghcr.io/.../backend:<tag>

  frontend:
    image: ghcr.io/.../frontend:<tag>
```

Mas determine a melhor estratégia de tags considerando a forma como o Portainer atualiza a Stack.

Avalie cuidadosamente a diferença entre:

```text
latest
```

e:

```text
sha-<commit>
```

em relação ao mecanismo de atualização do Portainer.

Se o uso de tag imutável exigir uma estratégia adicional para o Portainer saber qual versão deve utilizar, projete essa estratégia.

**Não crie uma solução que publique uma imagem nova mas deixe o Portainer executando uma imagem antiga.**

O fluxo deve ser determinístico.

---

# 9. DEPLOY INDEPENDENTE

Quero deployment independente.

Exemplo:

### Alteração apenas no backend

```text
commit
  ↓
detecta backend alterado
  ↓
CI backend
  ↓
build backend
  ↓
push backend
  ↓
Portainer deploy backend
```

Frontend não deve ser reconstruído desnecessariamente.

### Alteração apenas no frontend

```text
commit
  ↓
detecta frontend alterado
  ↓
CI frontend
  ↓
build frontend
  ↓
push frontend
  ↓
Portainer deploy frontend
```

Backend não deve ser reconstruído desnecessariamente.

### Alteração nos dois

```text
commit
  ↓
backend CI ──────┐
                 ├──→ registry → Portainer
frontend CI ─────┘
```

Determine a melhor forma de lidar com deploys simultâneos para evitar condições de corrida.

---

# 10. BANCO DE DADOS E PRISMA

O backend possui Prisma.

Analise cuidadosamente:

```text
apps/backend/prisma/
```

e as migrations existentes.

Determine se migrations devem ser executadas:

* durante o build;
* durante o startup do container;
* em um job separado;
* em um container/job temporário;
* através de outro mecanismo.

**Não execute migrations automaticamente durante o `docker build`.**

Analise os riscos de executar:

```text
prisma migrate deploy
```

em múltiplas réplicas ou múltiplos containers simultaneamente.

Proponha a estratégia mais segura para produção.

Não altere o comportamento das migrations sem explicar.

---

# 11. SECRETS E VARIÁVEIS DE AMBIENTE

Faça uma separação clara entre:

### GitHub Secrets

Credenciais necessárias para:

* autenticar no registry;
* chamar o Portainer;
* qualquer outro recurso que realmente precise ser acessado pelo GitHub Actions.

### Portainer / servidor

Secrets e variáveis necessárias pelos containers de produção.

**Não coloque secrets de produção dentro das imagens Docker.**

**Não copie `.env.prod` para dentro da imagem.**

**Não commite secrets.**

Analise também se alguma variável do Next.js precisa estar disponível em build time, pois algumas variáveis podem ser incorporadas ao bundle durante o build.

Diferencie corretamente:

```text
build-time environment
```

de:

```text
runtime environment
```

principalmente no frontend Next.js.

---

# 12. SEGURANÇA

Faça uma revisão de segurança do pipeline.

Avalie:

* permissões do `GITHUB_TOKEN`;
* `contents: read`;
* permissões para publicar no GHCR;
* secrets;
* webhook do Portainer;
* exposição do Portainer à internet;
* tokens;
* credenciais;
* supply chain;
* pinning de actions;
* dependências;
* execução como root;
* imagens Docker;
* vazamento de variáveis;
* logs contendo secrets.

Use o princípio do menor privilégio.

Se recomendar alguma proteção adicional, explique.

---

# 13. CACHE E PERFORMANCE

Otimize o pipeline.

Avalie:

* `actions/setup-node` com cache;
* npm cache;
* Docker Buildx;
* GitHub Actions cache;
* cache de layers Docker;
* `npm ci`;
* builds paralelos.

Quero evitar reinstalar/rebuildar coisas desnecessariamente.

Mas não sacrifique confiabilidade em troca de alguns segundos.

---

# 14. CONCORRÊNCIA

Projete uma estratégia para evitar deploys conflitantes.

Por exemplo:

```yaml
concurrency:
```

ou mecanismo equivalente.

Considere este cenário:

```text
commit A → deploy
commit B → deploy
commit C → deploy
```

e determine o comportamento desejado.

Não quero que uma versão antiga consiga terminar o deployment depois de uma versão mais nova e sobrescrevê-la.

Projete isso cuidadosamente.

---

# 15. ROLLBACK

Quero que seja possível fazer rollback.

Defina como voltar de:

```text
backend:sha-ABC
```

para:

```text
backend:sha-XYZ
```

sem precisar reconstruir a imagem.

A imagem publicada deve ser reutilizável.

Explique como o rollback será realizado no Portainer.

---

# 16. BRANCHES E AMBIENTES

Analise a estratégia atual de branches.

Se não existir uma estratégia clara, proponha uma simples.

Considere pelo menos:

```text
pull request
    ↓
CI
    ↓
merge main
    ↓
production
```

Não implemente staging automaticamente sem necessidade.

Se sugerir:

```text
develop → staging
main → production
```

justifique.

---

# 17. PULL REQUESTS

Quero separar:

### CI de Pull Request

Executa:

```text
lint
test
build
```

mas **não faz deploy de produção**.

### CI/CD da main

Executa:

```text
lint
test
build
docker build
docker push
Portainer deploy
```

quando aplicável.

Analise se isso é adequado para o projeto.

---

# 18. PATH FILTERING

Implemente detecção inteligente de mudanças.

Exemplo conceitual:

```text
apps/backend/**       → backend
apps/frontend/**      → frontend
docker/backend.*      → backend
docker/frontend.*     → frontend
```

Mas analise também arquivos compartilhados.

Se existir algo como:

```text
package-lock.json
docker-compose.prod.yml
.github/**
```

determine corretamente o impacto.

Não quero uma regra simplista do tipo:

```text
qualquer mudança no repositório → rebuild de tudo
```

a menos que a arquitetura realmente exija isso.

---

# 19. WORKFLOW FINAL

Depois da análise, implemente os arquivos necessários.

Pode ser algo como:

```text
.github/
└── workflows/
    ├── backend.yml
    ├── frontend.yml
    └── ...
```

ou uma arquitetura diferente se você concluir que é melhor.

Crie somente os arquivos realmente necessários.

Evite complexidade prematura.

---

# 20. NÃO FAÇA ISSO

Não:

* invente arquivos que não existem;
* assuma scripts sem verificar `package.json`;
* assuma que o Portainer está configurado de determinada forma;
* coloque secrets no Git;
* coloque secrets nas imagens;
* use SSH se Portainer Webhook resolver adequadamente;
* faça deploy se os testes falharem;
* publique imagem se o build falhar;
* use `latest` como única estratégia de versionamento;
* execute migrations no Docker build;
* reconstrua todas as aplicações em qualquer alteração;
* introduza Nx/Turborepo/pnpm/yarn sem necessidade;
* reescreva toda a infraestrutura existente sem necessidade.

---

# 21. ANTES DE IMPLEMENTAR

Apresente primeiro:

## A. Diagnóstico

O que existe atualmente.

## B. Problemas encontrados

Liste problemas/riscos do CI/CD atual, caso existam.

## C. Arquitetura proposta

Mostre o fluxo:

```text
GitHub
 ↓
GitHub Actions
 ↓
Docker
 ↓
GHCR
 ↓
Portainer
 ↓
Production
```

## D. Estratégia de deploy

Explique exatamente como:

```text
backend
```

e:

```text
frontend
```

serão publicados e atualizados independentemente.

## E. Estratégia de tags

Explique como funcionará:

```text
sha-xxxxx
latest
```

ou a estratégia que você escolher.

## F. Estratégia do Portainer

Explique exatamente como o GitHub Actions irá conversar com o Portainer.

## G. Estratégia de migrations

Explique como Prisma migrations serão executadas.

## H. Secrets

Liste quais secrets precisam existir no GitHub e quais ficam exclusivamente no Portainer.

## I. Rollback

Explique como realizar rollback.

Somente depois dessa análise e justificativa faça as alterações.

---

# 22. IMPLEMENTAÇÃO

Depois da aprovação lógica da arquitetura, implemente o CI/CD diretamente no projeto.

Ao final:

1. mostre todos os arquivos criados;
2. mostre todos os arquivos modificados;
3. explique cada alteração;
4. valide a sintaxe dos workflows;
5. valide os Dockerfiles;
6. valide o `docker-compose.prod.yml`;
7. verifique se os paths estão corretos;
8. verifique se os nomes das imagens estão consistentes;
9. verifique se os secrets utilizados realmente correspondem aos necessários;
10. verifique se o workflow não faz deploy quando os testes falham;
11. verifique se backend e frontend podem ser publicados independentemente;
12. verifique se o Portainer receberá o trigger somente depois que a imagem tiver sido publicada com sucesso;
13. verifique se rollback é possível;
14. documente qualquer configuração manual necessária no GitHub ou Portainer.

---

# 23. DOCUMENTAÇÃO

Crie ou atualize uma documentação específica, por exemplo:

```text
docs/deployment.md
```

ou outro local apropriado à estrutura existente.

Documente:

* arquitetura;
* GitHub Actions;
* registry;
* imagens;
* tags;
* secrets;
* Portainer;
* webhook;
* Stack;
* deploy;
* rollback;
* migrations;
* troubleshooting.

A documentação deve permitir que outro desenvolvedor configure o ambiente sem precisar descobrir informações implícitas.

---

# 24. CRITÉRIO FINAL DE QUALIDADE

Considere a implementação concluída somente se este fluxo for verdadeiro:

### Backend

```text
Alteração em apps/backend
        ↓
PR
        ↓
CI backend
        ↓
merge main
        ↓
build backend
        ↓
test backend
        ↓
Docker build backend
        ↓
push GHCR
        ↓
Portainer
        ↓
backend atualizado
```

### Frontend

```text
Alteração em apps/frontend
        ↓
PR
        ↓
CI frontend
        ↓
merge main
        ↓
build frontend
        ↓
test/build frontend
        ↓
Docker build frontend
        ↓
push GHCR
        ↓
Portainer
        ↓
frontend atualizado
```

### Regra fundamental

Se somente o backend mudou:

```text
backend = deploy
frontend = NÃO deploy
```

Se somente o frontend mudou:

```text
frontend = deploy
backend = NÃO deploy
```

Se ambos mudaram:

```text
backend = deploy
frontend = deploy
```

Se os testes falharem:

```text
NÃO publicar imagem
NÃO fazer deploy
```

Se o Docker build falhar:

```text
NÃO fazer deploy
```

Se o push da imagem falhar:

```text
NÃO fazer deploy
```

Se o Portainer falhar:

```text
pipeline deve indicar claramente que o build/push foi concluído,
mas o deployment falhou.
```

---

## Resultado esperado

Quero uma solução **production-ready**, mas sem overengineering.

Priorize:

**confiabilidade > segurança > simplicidade > performance.**

Não quero apenas um exemplo de GitHub Actions.

Quero que você **analise o projeto real, adapte a solução à arquitetura existente e implemente o CI/CD completo GitHub → Registry → Portainer**, mantendo backend e frontend independentes e garantindo deploy seguro e rollback.
