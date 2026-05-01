# Stack de Testes Automatizados — ASP.NET Core MVC Crowdfunding

> **TDD aplicado a um sistema real de vaquinha online:** testes de unidade com xUnit + Moq, testes de integração com WebApplicationFactory, cobertura de código com Coverlet e análise contínua de qualidade via SonarCloud — tudo orquestrado por GitHub Actions.

[![.NET](https://img.shields.io/badge/.NET-8.0-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![C#](https://img.shields.io/badge/C%23-12.0-239120?style=for-the-badge&logo=csharp&logoColor=white)](https://learn.microsoft.com/dotnet/csharp/)
[![Build Status](https://img.shields.io/github/actions/workflow/status/Santosdevbjj/stack-Testes-Unidade-Integracao/dotnet.yml?style=for-the-badge&logo=githubactions&logoColor=white)](https://github.com/Santosdevbjj/stack-Testes-Unidade-Integracao/actions)
[![SonarCloud](https://img.shields.io/badge/SonarCloud-análise-F3702A?style=for-the-badge&logo=sonarcloud&logoColor=white)](https://sonarcloud.io/)
[![License](https://img.shields.io/badge/Licença-MIT-green?style=for-the-badge)](LICENSE)

---

## 1. Problema de Negócio

Entregas sem testes automatizados criam um ciclo previsível: o desenvolvedor faz uma mudança, algo quebra em produção, a correção introduz outro bug. Em sistemas que lidam com transações financeiras — como uma plataforma de doações — esse ciclo tem custo real e imediato: transações processadas incorretamente, dados corrompidos e confiança do usuário destruída.

O problema concreto que este projeto endereça é duplo:

- **Ausência de cobertura de testes** na camada de serviços: o `DonationService` processa doações sem qualquer validação automatizada de que faz o que deveria fazer — inclusive em casos de borda como doações nulas.
- **Falta de validação de contrato HTTP**: os controllers MVC não têm garantia de que respondem com os status codes corretos para cada cenário. Um refactor inocente pode fazer o endpoint de doações retornar 500 sem que ninguém perceba até a próxima reclamação de usuário.

**A questão central:** como construir uma rede de segurança automatizada que permita evoluir o código com confiança, detectar regressões antes do merge e medir objetivamente a cobertura — tudo integrado ao fluxo de CI/CD?

---

## 2. Contexto

A aplicação base é uma plataforma de crowdfunding (vaquinha online) construída em ASP.NET Core MVC, estruturada em três camadas: Controllers (entrada HTTP), Services (lógica de negócio) e Repositories (persistência). A arquitetura usa injeção de dependências, o que torna as camadas testáveis de forma isolada.

O objetivo deste repositório não é construir funcionalidades novas na aplicação — é **implementar a stack de testes completa** sobre uma aplicação existente, um cenário extremamente comum em projetos legados corporativos onde o código funciona mas não tem cobertura.

Esse cenário é deliberado. Em 15 anos em sistemas bancários críticos, aprendi que a maior parte do trabalho de qualidade não acontece em projetos novos — acontece em sistemas que já existem, já estão em produção, e precisam ser evoluídos sem risco de regressão.

---

## 3. Premissas

Para estruturar a stack de testes, foram adotadas as seguintes premissas:

- Os testes de unidade devem isolar completamente as dependências externas usando Moq. O `DonationServiceTests` nunca deve tocar o banco de dados real — ele testa apenas o comportamento do serviço diante de um repositório mockado.
- Os testes de integração devem validar o contrato HTTP dos controllers usando `WebApplicationFactory<Program>`, que sobe a aplicação real em memória sem necessidade de servidor externo.
- A cobertura mínima aceitável é de 80% de linhas, 75% de branches e 70% de métodos — thresholds configurados no pipeline de CI via ReportGenerator.
- O pipeline deve falhar se o estilo de código não estiver em conformidade com o `.editorconfig`, garantindo consistência desde o primeiro commit.
- Nenhum segredo ou connection string de produção deve transitar pelo pipeline — a aplicação usa banco em memória (`Microsoft.EntityFrameworkCore.InMemory`) nos testes.

---

## 4. Estratégia da Solução

**Testes de Unidade — camada de serviço e repositório**

O `DonationServiceTests` usa Moq para criar um mock do `IDonationRepository`, permitindo testar o `DonationService` de forma completamente isolada. Dois cenários críticos são cobertos: processamento bem-sucedido de uma doação válida (verificando que o repositório recebe exatamente uma chamada `Add`) e lançamento de `ArgumentNullException` quando a doação é nula.

O `DonationRepositoryTests` testa a implementação concreta do repositório em memória (`List<Donation>`), validando que doações são armazenadas corretamente e que a lista começa vazia.

**Testes de Integração — contrato HTTP dos controllers**

Os testes de integração usam `WebApplicationFactory<Program>` para subir a aplicação completa em memória. Isso garante que o pipeline de middleware do ASP.NET Core — roteamento, autorização, model binding — está funcionando como esperado. Dois cenários são cobertos: `GET /donations` deve retornar 200 OK com conteúdo HTML contendo "Doações", e `POST /donations` com payload válido deve retornar 302 Redirect.

**Pipeline de CI/CD com GitHub Actions**

O pipeline executa em sequência: checkout → instalação do SDK → restore → validação de estilo com `dotnet format --verify-no-changes` → build → testes de unidade com cobertura → testes de integração com cobertura → geração de relatório HTML com ReportGenerator → publicação do relatório como artefato. Se qualquer etapa falhar, o pipeline para e o merge é bloqueado.

**Análise de qualidade com SonarCloud**

O `sonar-project.properties` configura o SonarCloud para analisar apenas o código de produção (`src/`), excluindo arquivos gerados automaticamente, migrations e o próprio código de testes. Os relatórios de cobertura OpenCover gerados pelo Coverlet são passados ao SonarCloud para correlacionar cobertura com qualidade de código.

---

## 5. Decisões Técnicas e Aprendizados

### Por que xUnit e não NUnit ou MSTest?

xUnit foi projetado para .NET Core desde o início, tem melhor suporte a testes paralelos e é o framework padrão da Microsoft para novos projetos .NET. Mais importante: a injeção de dependências no xUnit (via construtor da classe de teste) se alinha naturalmente com o padrão DI do ASP.NET Core, tornando o setup dos mocks mais explícito e legível.

### Por que manter o Startup.cs separado do Program.cs?

Em .NET 8, o padrão moderno é o `Program.cs` com top-level statements. Mantive o `Startup.cs` separado porque o `WebApplicationFactory` nos testes de integração precisa de um ponto de entrada estável para substituir serviços no ambiente de teste. Sem o `Startup.cs`, seria necessário usar a API mais verbosa de `WebApplicationFactory.WithWebHostBuilder()` em cada teste. O trade-off de manter dois arquivos de configuração se justifica pela legibilidade dos testes.

### Por que thresholds de cobertura no ReportGenerator?

Cobertura de código sem enforcement vira métrica decorativa. Ao configurar `LineCoverage>80`, `BranchCoverage>75` e `MethodCoverage>70` como regras no ReportGenerator, o pipeline falha automaticamente se a cobertura cair abaixo do mínimo. Isso cria uma pressão saudável: qualquer novo código precisa vir acompanhado de testes.

### O que aprendi sobre testes de integração com WebApplicationFactory

O maior aprendizado foi entender a diferença entre o ambiente de teste e o ambiente de desenvolvimento. Na primeira tentativa, os testes de integração falhavam porque o banco em memória não era reinicializado entre os testes, causando interferência entre casos de teste. A solução foi configurar o `WebApplicationFactory` com `UseInMemoryDatabase(Guid.NewGuid().ToString())` para criar uma instância nova a cada teste.

### O que faria diferente

Adicionaria testes de mutação com Stryker.NET para validar a qualidade dos próprios testes — não apenas se o código está coberto, mas se os testes realmente detectam defeitos. Cobertura de 100% com assertions fracas é pior do que 70% com assertions precisas.

---

## 6. Resultados

O projeto entregou uma stack de testes funcional com os seguintes resultados concretos:

**Cobertura da camada de serviço:** 100% dos métodos do `DonationService` cobertos — incluindo o caminho feliz e o tratamento de exceção para entrada nula.

**Cobertura do repositório:** todos os métodos do `DonationRepository` testados com instância real em memória, sem dependências externas.

**Validação de contrato HTTP:** dois endpoints críticos do controller de doações validados via testes de integração — GET e POST — com verificação de status code e conteúdo da resposta.

**Pipeline automatizado:** build, testes e relatório de cobertura executam automaticamente a cada push em `main` e a cada pull request, com artefato HTML publicado para inspeção visual da cobertura.

**Qualidade de código:** `.editorconfig` com regras de estilo C# validadas pelo `dotnet format` no pipeline, garantindo consistência de indentação, uso de `var`, expression-bodied members e outros padrões.

**Integração SonarCloud:** análise estática configurada com exclusão de código gerado, separação entre código de produção e testes, e correlação com relatórios de cobertura Coverlet/OpenCover.

---

## 7. Próximos Passos

- Implementar testes de mutação com Stryker.NET para medir a eficácia das assertions existentes.
- Adicionar testes de carga com k6 ou NBomber para validar o comportamento do endpoint de doações sob volume.
- Configurar Quality Gate no SonarCloud para bloquear merges quando a cobertura cair ou novos code smells críticos forem introduzidos.
- Implementar testes de regressão visual para as views MVC usando Playwright.
- Expandir os testes de integração para cobrir cenários de erro: payload inválido, campos obrigatórios ausentes, valores negativos de doação.

---

## Arquitetura e Fluxo de Testes

```
stackTestesUnidIntegra.sln
│
├── src/VaquinhaOnline/               # Aplicação de produção
│   ├── Controllers/                  # Entrada HTTP → MVC
│   ├── Services/DonationService.cs   # Lógica: valida e delega ao repositório
│   ├── Repositories/                 # Persistência em memória
│   ├── Models/Donation.cs            # Name + Amount
│   ├── Program.cs                    # Startup moderno .NET 8
│   └── Startup.cs                    # Usado pelo WebApplicationFactory
│
└── tests/
    ├── VaquinhaOnline.UnitTests/     # Camada: Services + Repositories
    │   ├── Services/
    │   │   └── DonationServiceTests.cs   # Mock do repositório via Moq
    │   └── Repositories/
    │       └── DonationRepositoryTests.cs # Instância real em memória
    │
    └── VaquinhaOnline.IntegrationTests/  # Camada: Controllers HTTP
        └── Controllers/
            └── DonationControllerTests.cs # WebApplicationFactory + HttpClient

Fluxo do Pipeline CI:
push/PR → checkout → dotnet restore → dotnet format
→ dotnet build → unit tests (Coverlet) → integration tests (Coverlet)
→ ReportGenerator (HTML + thresholds) → upload artefato → SonarCloud
```

---

## Como Executar

### Pré-requisitos

- .NET SDK 8.0
- Git

### Clonar e executar a aplicação

```bash
git clone https://github.com/Santosdevbjj/stack-Testes-Unidade-Integracao.git
cd stack-Testes-Unidade-Integracao
dotnet restore
dotnet build
dotnet run --project src/VaquinhaOnline/VaquinhaOnline.csproj
# Aplicação disponível em: http://localhost:5000
```

### Executar os testes

```bash
# Testes de unidade com cobertura
dotnet test tests/VaquinhaOnline.UnitTests/VaquinhaOnline.UnitTests.csproj \
  --collect:"XPlat Code Coverage"

# Testes de integração com cobertura
dotnet test tests/VaquinhaOnline.IntegrationTests/VaquinhaOnline.IntegrationTests.csproj \
  --collect:"XPlat Code Coverage"

# Todos os testes da solução
dotnet test stackTestesUnidIntegra.sln --collect:"XPlat Code Coverage"
```

### Gerar relatório de cobertura local

```bash
# Instalar o ReportGenerator (uma vez)
dotnet tool install -g dotnet-reportgenerator-globaltool

# Gerar relatório HTML
reportgenerator \
  -reports:"**/coverage.cobertura.xml" \
  -targetdir:"coverage-report" \
  -reporttypes:HtmlInline_AzurePipelines \
  -assemblyfilters:+VaquinhaOnline*

# Abrir relatório no navegador
open coverage-report/index.html   # macOS
start coverage-report/index.html  # Windows
```

### Validar estilo de código

```bash
dotnet format --verify-no-changes
```

---

## Tecnologias Utilizadas

| Camada | Tecnologia | Função |
|---|---|---|
| **Aplicação** | .NET 8, C# 12, ASP.NET Core MVC | Plataforma de crowdfunding testada |
| **Testes de Unidade** | xUnit, Moq, FluentAssertions | Isolamento e assertions precisas |
| **Testes de Integração** | xUnit, WebApplicationFactory | Validação do contrato HTTP real |
| **Cobertura** | Coverlet + ReportGenerator | Métricas e relatório HTML com thresholds |
| **Qualidade** | SonarCloud | Análise estática, code smells, segurança |
| **Estilo** | dotnet format + .editorconfig | Consistência de código no CI |
| **CI/CD** | GitHub Actions | Orquestração de todo o pipeline |

---

## Requisitos de Hardware e Software

| Item | Recomendado |
|---|---|
| CPU | Intel i5 ou superior |
| RAM | 8 GB (SSD recomendado) |
| .NET SDK | 8.0 |
| IDE | Visual Studio 2022 ou VS Code |
| SO | Windows 10/11, macOS Ventura+, Ubuntu 22+ |

---

## Autor

**Sergio Santos**  
Data Engineer & Cloud Architect — 15+ anos em sistemas críticos bancários (Bradesco)  
Campus Expert DIO · Bootcamps: .NET, Azure, Java, Python, TDD

[![Portfólio](https://img.shields.io/badge/Portfólio-Sérgio_Santos-111827?style=for-the-badge&logo=githubpages&logoColor=00eaff)](https://portfoliosantossergio.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sérgio_Santos-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/santossergioluiz)
[![GitHub](https://img.shields.io/badge/GitHub-Santosdevbjj-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Santosdevbjj)
