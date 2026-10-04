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

## Evidência 1

Linha de erro:
[ERROR] App.java:[4,38] package com.fasterxml.jackson.databind does not exist

Import que causou o erro: import com.fasterxml.jackson.databind.ObjectMapper; (linha 4 do App.java).
O mesmo acontece com import com.fasterxml.jackson.core.type.TypeReference; (linha 3).
O App.java usa a biblioteca Jackson, mas o pom.xml não a declara como dependência, por isso o Maven não a descarrega e o compilador não encontra as classes ObjectMapper e TypeReference.
## Passo 2

Depois de adicionar a dependência jackson-databind ao pom.xml, o projeto passou a compilar (BUILD SUCCESS).

Nota: o projeto fornecido não tinha pasta src/test, por isso o Maven não tinha testes para executar e a falha na fase de testes descrita na ficha não aconteceu. O defeito existia no FleetService.java: a condição usava ">" em vez de ">=". Um veículo com exatamente 10000 km desde a última revisão (V3) não era contado como precisando de revisão. Corrigi para ">=".

Pergunta: Porque é que esta falha é melhor do que a do passo 1?
No passo 1 o build parava na compilação por um problema de configuração (dependência em falta), por isso nem era possível verificar se o programa funcionava. Uma falha nos testes significa que o build já avançou mais: o código compila e as dependências estão resolvidas, e o que falha agora é o comportamento do programa. É uma falha mais útil, porque aponta para um defeito real na lógica da aplicação e não para um problema de configuração.
## Passo 3

Resultado de mvn dependency:tree:
- Dependência direta: com.fasterxml.jackson.core:jackson-databind:2.22.2 (compile)
- Dependências transitivas (trazidas pelo jackson-databind): jackson-core:2.22.2 e jackson-annotations:2.22 (compile)
- junit-jupiter:5.14.4 (test) e as suas dependências transitivas, usadas apenas nos testes.

## Evidência 4

O JAR por defeito (target/fleetcheck-1.0.0.jar) não era executável. Ao correr java -jar apareceu o erro "no main manifest attribute", porque o ficheiro MANIFEST.MF não indicava a classe principal. Além disso, esse JAR só continha as classes do projeto, sem o Jackson, por isso mesmo com a classe principal definida o programa falharia ao precisar das bibliotecas.

O plugin Shade criou um segundo JAR, target/fleetcheck-1.0.0-all.jar (um "uber JAR"), que:
- tem no MANIFEST.MF a entrada Main-Class: pt.upt.fleetcheck.App (definida pelo ManifestResourceTransformer);
- inclui dentro dele as classes das dependências de runtime (jackson-databind, jackson-core e jackson-annotations).

Assim o JAR passa a ser autossuficiente e pode ser executado com java -jar target/fleetcheck-1.0.0-all.jar, produzindo o resultado esperado:
FleetCheck 1.0 | Vehicles loaded: 4 | Vehicles requiring service: 2 | Average mileage: 37000 km

## Passo 5

Foi gerado o Maven Wrapper com mvn wrapper:wrapper. O projeto passou a conter mvnw, mvnw.cmd e a pasta .mvn/wrapper/, configurada para usar o Maven 3.10.0. A partir daqui o build é feito com mvnw.cmd clean verify.

Foi também adicionada a propriedade project.build.outputTimestamp (2026-09-27T00:00:00Z), para que os plugins compatíveis usem sempre a mesma data nos ficheiros gerados e o mesmo código produza sempre o mesmo artefacto.

Pergunta: Que pressuposto escondido sobre o ambiente foi eliminado pelo wrapper?
O pressuposto de que quem compila o projeto já tem o Maven instalado e configurado no PATH, e na versão certa. Sem o wrapper, o build dependia da máquina do programador: noutra máquina o Maven podia não existir ou ser de outra versão, e o resultado podia ser diferente. Com o wrapper, a versão do Maven fica definida dentro do próprio projeto e é descarregada automaticamente se for necessário.

## Passo 6

O projeto foi enviado para o GitHub (https://github.com/ines111006/fleetcheck-maven) e foi criado o workflow .github/workflows/build.yml. Em cada push ou pull request, o GitHub Actions usa uma máquina Ubuntu com Java 21, executa ./mvnw -B clean verify e guarda o JAR gerado como artefacto (fleetcheck-build).

Execução com sucesso: https://github.com/ines111006/fleetcheck-maven/actions/runs/37235774588

O ficheiro mvnw foi marcado como executável antes do primeiro commit (git update-index --chmod=+x mvnw), para evitar o erro de permissões no Linux.

## Evidência 7

Foi adicionado o plugin cyclonedx-maven-plugin (versão 2.9.3), executado na fase verify. O comando mvnw.cmd clean verify gerou o ficheiro target/bom.json, um SBOM no formato CycloneDX 1.6 com 3 componentes: jackson-databind 2.22.2, jackson-core 2.22.2 e jackson-annotations 2.22.

Porque é que o SBOM contém componentes que não foram escritos explicitamente no pom.xml?
No pom.xml só foi declarada a dependência direta jackson-databind. No entanto, o jackson-databind precisa do jackson-core e do jackson-annotations para funcionar, por isso o Maven resolve automaticamente essas dependências transitivas (como se viu no mvn dependency:tree). O SBOM descreve o que realmente vai dentro do software, e não só o que foi escrito no pom.xml, por isso inclui todo o grafo de dependências de runtime: diretas e transitivas. As dependências de teste (JUnit) não aparecem porque não fazem parte do produto final.