# FleetCheck – Build Systems Lab

This project is intentionally incomplete. Follow the worksheet in the order given.

Expected final application output:

```
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

Do not copy the solution POM. The objective is to observe how each build change alters the result.

## Relatório — FleetCheck (Maven)

### Passo 1 — Build inicial (falha esperada)
Erro de compilação obtido:

package com.fasterxml.jackson.core.type does not exist

Causado pelo import `com.fasterxml.jackson.core.type.TypeReference` (linha 3 do `App.java`). 
O segundo erro relacionado vem de `import com.fasterxml.jackson.databind.ObjectMapper` (linha 4). 
Os restantes erros ("cannot find symbol") são consequência destes dois imports em falta, já que o Jackson não estava declarado no `pom.xml`.

### Passo 2 — Dependência do Jackson
Após adicionar `jackson-databind:2.22.2`, a compilação passou a ter sucesso.
Esta é uma falha melhor do que a do passo 1 porque o build avançou mais: deixou de ser um erro de configuração (dependência em falta) e passou a estar na fase de testes, mais próxima de detetar defeitos de comportamento da aplicação.

**Nota:** o projeto inicial não continha nenhuma classe em `src/test/java`, pelo que não foi possível observar a falha de testes esperada neste passo. O defeito de comportamento foi observado mais tarde, através da saída da aplicação (ver Passo 4).

### Passo 3 — Árvore de dependências
`jackson-databind:2.22.2` aparece como dependência **direta**.
`jackson-core` e `jackson-annotations` aparecem como dependências **transitivas**, trazidas automaticamente pelo `jackson-databind`.

### Passo 4 — JAR executável
- JAR normal (`fleetcheck-1.0.0.jar`): falha ao executar com `no main manifest attribute`, porque não tem `Main-Class` definida nem as dependências incluídas.
- Após adicionar o `maven-shade-plugin`, o JAR resultante (`fleetcheck-1.0.0-all.jar`) passou a ser executável, porque o Shade:
  - define `Main-Class: pt.upt.fleetcheck.App` no manifesto;
  - incorpora as classes de todas as dependências dentro do próprio JAR ("fat JAR").

Saída obtida:

FleetCheck 1.0 | Vehicles loaded: 4 | Vehicles requiring service: 1 | Average mileage: 37000 km

**Defeito identificado:** o valor de "Vehicles requiring service" é 1, mas o esperado é 2 — um defeito de comportamento na lógica da aplicação, não no build.

### Passo 5 — Maven Wrapper
Gerado com `mvn wrapper:wrapper`, criando `mvnw`, `mvnw.cmd` e `.mvn/wrapper/`.
**Pergunta:** o wrapper remove a suposição de que quem for correr o build (colega ou CI) já tem o Maven instalado, e com a versão correta. Com o wrapper, a versão certa do Maven é descarregada automaticamente, tornando o build reprodutível em qualquer máquina.

### Passo 6 — GitHub Actions
Workflow criado em `.github/workflows/build.yml`, correndo `./mvnw -B clean verify`.
Primeira execução falhou com `Permission denied` ao correr `./mvnw`, porque o Windows não preserva a permissão de execução do ficheiro ao fazer commit. Resolvido com:

git update-index --chmod=+x mvnw

Execução final com sucesso: <URL_DO_RUN>

**Nota:** a linha `target/site/jacoco/**` foi removida da lista de artefactos a enviar, porque o plugin JaCoCo não gera relatório sem testes (o projeto inicial não tinha testes em `src/test/java`).

### Passo 7 — SBOM (CycloneDX)
Gerado `target/bom.json` através do `cyclonedx-maven-plugin`.
**Pergunta:** o SBOM contém componentes (como `jackson-core` e `jackson-annotations`) que não foram escritos explicitamente nas dependências porque são dependências **transitivas**, resolvidas automaticamente pelo Maven a partir do `jackson-databind`. O SBOM reflete todo o grafo de dependências efetivamente usado em runtime, não só o que foi declarado diretamente.
