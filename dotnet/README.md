[![Blog](https://img.shields.io/website?down_color=blue&down_message=infrati.dev&label=Blog&logo=ghost&logoColor=green&style=for-the-badge&up_color=blue&up_message=infrati.dev&url=https%3A%2F%2Finfrati.dev)](https://infrati.dev)

#### 📋 infratidev

##### Pipeline MultiStage AzureDevops para dotnet.

##### Requirements

* Agente AzureDevops instalado
* Servidor IIS OnPremise
* SonarQube onPremise integrado com AzureDevops
* Polices aplicadas nas branches.
    * Nenhum commit é feito direto na branch
    * Reviewers parametrizados
    * Delimitando os tipos de merge
    * Inclusão automática dos reviewers

##### Branches

* dev
* hml
* main

```dev -> hml -> main```

##### Etapas Descrição:

* Na Branch dev iremos executar o Build Validation (Stage01) no PR e realizar as verificações necessárias de QA/SAST(com Sonar) e Build. Somente após bem sucedido o Build Validation é liberado para os reviewers aprovarem o PR.
* PR aprovado o merge é realizado em dev. Com ambiente separado para os devs realizarem os testes.
* Testes bem sucedidos em dev, PR para hml criado e aprovado é executado o pipeline de Stage01 de Build/QA/SAST e Stage02 de deploy no ambiente HML Onpremise. Detecção do ambiente é feito pelas regras de environment pré-estabelecidas.
* Testes bem sucedidos em hml, PR para main criado e aprovado é executado o pipeline de Stage01 de Build/QA/SAST e Stage02 de deploy no ambiente Produção Onpremise. Detecção do ambiente é feito pelas regras de environment pre-estabelecidas.

##### Trigger do pipeline feito o commit

~~~
trigger:
  branches:
    include:
    - main
    - hml

  paths:
    exclude:
    - README.md
~~~

##### Definições de variáveis

~~~
variables:
  solution: '**/*.sln'
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'
  NUGET_PACKAGES: $(Pipeline.Workspace)/.nuget/packages
  CACHE_RESTORED: 'false'
  isMain: $[eq(variables['Build.SourceBranch'], 'refs/heads/main')]
  isHml: $[eq(variables['Build.SourceBranch'], 'refs/heads/hml')]
  isDev: $[eq(variables['Build.SourceBranch'], 'refs/heads/dev')]
  isPrSbDev:  $[eq(variables['System.PullRequest.SourceBranch'], 'refs/heads/dev')]
  isPrSbHml:  $[eq(variables['System.PullRequest.SourceBranch'], 'refs/heads/hml')]
  isPrTbDev:  $[eq(variables['System.PullRequest.TargetBranch'], 'refs/heads/dev')]
  isPrTbHml:  $[eq(variables['System.PullRequest.TargetBranch'], 'refs/heads/hml')]
  isPrTbMain:  $[eq(variables['System.PullRequest.TargetBranch'], 'refs/heads/main')]
  WebsitePhysicalPath: '%SystemDrive%\inetpub\wwwroot\App01'
  VirtualPathForApplication: '/App01'
  AppPoolName: 'App01'
  VirtualApplication: 'App01'
  ProjectName: 'project-app'
~~~

##### Checkout do repositorio dependente e template dos stages de build e deploy

~~~
resources:
 repositories:
   - repository: templates
     type: git
     name: PROJECTA/pipelines-templates
~~~

##### Build/QA/SAST Stage

~~~
stages:
- stage: Build_QA_SAST
  displayName: 'Stage01 - Build, QA e SAST'

  jobs:
  - job: Job_Build_QA_SAST
    displayName: 'Job01 - Job Build,QA e SAST'
    pool:
      vmImage: 'windows-2019'
      
    steps:
    - template: dotnet/dotnet-stage-build.yml@templates
      parameters:
        projectKey: 'PROJECTA_project-app'
        projectName: 'project-app'
~~~

##### Deploy HML Stage

~~~
- stage: Project_Deploy_HML
  displayName: 'Stage02 - Deploy Environment Hml'
  dependsOn: Build_QA_SAST
  condition: and(succeeded('Build_QA_SAST'), eq(variables.isHml, 'true'))
  variables:
    - group: dotNet-Environment-Hml

  jobs:
  - job: Job_Deploy_HML
    displayName: 'Job01 - Job de Deploy em HML'
  - deployment: Deploy_HML
    displayName: 'Job02 - Deployment em HML'
    environment: 
      name: 'InfraHML'
      resourceType: VirtualMachine
      resourceName: SERVERHML
    variables:
      Parameters.WebsiteName: 'SERVERHML'
      Parameters.ParentWebsiteNameForApplication: 'SERVERHML'
      Parameters.AppCmdCommands: 'set apppool /apppool.name:$(Parameters.AppPoolName) /enable32BitAppOnWin64:true'
      Parameters.URL: 'https://hml-url-do-sistema.com.br/App01/'
      #fixed variables
      Parameters.IISDeploymentType: 'IISWebApplication'
      Parameters.ActionIISWebsite: 'CreateOrUpdateWebsite'    
      Parameters.WebsitePhysicalPath: '$(WebsitePhysicalPath)'
      Parameters.AddBinding: false
      Parameters.VirtualPathForApplication: '$(VirtualPathForApplication)'
      Parameters.AppPoolName: '$(AppPoolName)'
      Parameters.VirtualApplication: '$(VirtualApplication)'
      Parameters.Package: '$(Pipeline.Workspace)\$(Parameters.ProjectName)\*.zip'
      Parameters.RemoveAdditionalFilesFlag: true
      Parameters.TakeAppOfflineFlag: true
      Parameters.XmlTransformation: true
      Parameters.XmlVariableSubstitution: true
      Parameters.CreateOrUpdateAppPoolForApplication: true
      Parameters.DotNetVersionForApplication: 'v4.0'
      Parameters.PipeLineModeForApplication: 'Integrated'
      Parameters.AppPoolIdentityForApplication: 'ApplicationPoolIdentity'      
      Parameters.ProjectName: '$(ProjectName)'

    workspace:
      clean: all
    strategy:
     runOnce:
       deploy:
         steps:
           - template: dotnet/dotnet-stage-deploy.yml@templates
             parameters:
              displayName: 'Task02 - Deploy HML'
~~~

##### Deploy PROD Stage

~~~
- stage: Project_Deploy_PROD
  displayName: 'Stage02 - Deploy Environment Prod'
  dependsOn: Build_QA_SAST
  condition: and(succeeded('Build_QA_SAST'), eq(variables.isMain, 'true'))
  variables:
    - group: dotNet-Environment-Prod

  jobs:
  - job: Job_Deploy_PROD
    displayName: 'Job01 - Job de Deploy PROD'  
  - deployment: Deploy_PROD
    displayName: 'Job02 - Deployment em PROD'
    environment: 
      name: 'InfraPROD'
      resourceType: VirtualMachine
      resourceName: SERVERPROD
    variables:
      Parameters.WebsiteName: 'SERVERPROD'
      Parameters.ParentWebsiteNameForApplication: 'SERVERPROD'
      Parameters.AppCmdCommands: 'set apppool /apppool.name:$(Parameters.AppPoolName) /enable32BitAppOnWin64:true'
      Parameters.URL: 'https://prod-url-do-sistema.com.br/App01/'
      Parameters.Subject: 'Nova versão do sistema X foi publicada.'               
      Parameters.EmailBody: 'Foi publicado uma nova versão do sistema X'      
      #fixed variables 
      Parameters.IISDeploymentType: 'IISWebApplication'
      Parameters.ActionIISWebsite: 'CreateOrUpdateWebsite'    
      Parameters.WebsitePhysicalPath: '$(WebsitePhysicalPath)'
      Parameters.AddBinding: false
      Parameters.VirtualPathForApplication: '$(VirtualPathForApplication)'
      Parameters.AppPoolName: '$(AppPoolName)'
      Parameters.VirtualApplication: '$(VirtualApplication)'
      Parameters.Package: '$(Pipeline.Workspace)\$(Parameters.ProjectName)\*.zip'
      Parameters.RemoveAdditionalFilesFlag: true
      Parameters.TakeAppOfflineFlag: true
      Parameters.XmlTransformation: true
      Parameters.XmlVariableSubstitution: true
      Parameters.CreateOrUpdateAppPoolForApplication: true
      Parameters.DotNetVersionForApplication: 'v4.0'
      Parameters.PipeLineModeForApplication: 'Integrated'
      Parameters.AppPoolIdentityForApplication: 'ApplicationPoolIdentity'      
      Parameters.ProjectName: '$(ProjectName)'

    workspace:
      clean: all
    strategy:
     runOnce:
       deploy:
        steps:
          - template: dotnet/dotnet-stage-deploy.yml@templates
            parameters:
              displayName: 'Task02 - Deploy PROD'
~~~

<br>

[![Blog](https://img.shields.io/website?down_color=blue&down_message=infrati.dev&label=Blog&logo=ghost&logoColor=green&style=for-the-badge&up_color=blue&up_message=infrati.dev&url=https%3A%2F%2Finfrati.dev)](https://infrati.dev)
