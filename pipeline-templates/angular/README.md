[![Blog](https://img.shields.io/website?down_color=blue&down_message=infrati.dev&label=Blog&logo=ghost&logoColor=green&style=for-the-badge&up_color=blue&up_message=infrati.dev&url=https%3A%2F%2Finfrati.dev)](https://infrati.dev)

#### 📋 infratidev

#### Template Pipeline MultiStage AzureDevops para angular. Build e Deploy em ambiente onpremise.

### Build/QA/SAST

Parametros passados para o ```azure-pipelines.yml```
~~~
parameters:
  - name: cliProjectKey
    type: string
  - name: cliProjectName
    type: string
  - name: cliProjectNameLib
    type: string
~~~

Checkout das dependências de repositórios

~~~
    - checkout: self
      displayName: Checkout - main project
    - checkout: iris-lib-angular
      displayName: Checkout - lib-angular dependent project
      path: 's/lib-angular'
~~~

Parametrização da integração com Sonar sendo executado onpremise

~~~
    - task: SonarQubePrepare@5
      displayName: 'Task01 - QA and SAST analysis in code'

      inputs:
        SonarQube: 'SonarQube_Onpremise'
        scannerMode: 'CLI'
        configMode: 'manual'
        cliProjectKey:  '${{ parameters.cliProjectKey }}'
        cliProjectName: '${{ parameters.cliProjectName }}'
        cliSources: '.'
        extraProperties: '#sonar.verbose=true'
~~~

Instalação do node

~~~
- task: NodeTool@0
      displayName: 'Task02 - Install Node.js'
      inputs:
        versionSpec: '10.x'
        checkLatest: true
~~~

Implementação de cache para otimização

~~~
- task: Cache@2
      displayName: 'Task03 - Cache npm main project'
      inputs:
        key: 'v1 | npm | "$(Agent.OS)" | $(Build.SourcesDirectory)/${{ parameters.cliProjectName }}/package.json'
        path: '$(Build.SourcesDirectory)/${{ parameters.cliProjectName }}/node_modules'
        restoreKeys: |
          v1 | npm | "$(Agent.OS)"
        cacheHitVar: CACHE_RESTORED

    - task: Cache@2
      displayName: 'Task04 - Cache npm lib angular'
      inputs:
        key: 'v1 | npm | "$(Agent.OS)" | $(Build.SourcesDirectory)/${{ parameters.cliProjectNameLib }}/package.json'
        path: '$(Build.SourcesDirectory)/${{ parameters.cliProjectNameLib }}/node_modules'
        restoreKeys: |
          v1 | npm | "$(Agent.OS)"
        cacheHitVar: CACHE_RESTORED
~~~

Parametrização, instalação do angular e dependências

~~~
- task: Npm@1  
      displayName: 'Task05 - Angular CLI Installation'  
      inputs:
        workingDir: '$(System.DefaultWorkingDirectory)/${{ parameters.cliProjectName }}'  
        command: custom  
        verbose: false  
        customCommand: 'install @angular/cli'        

    - task: Npm@1  
      displayName: 'Task06 - npm installation main project'
      condition: ne(variables.CACHE_RESTORED,'true')
      inputs:
        workingDir: '$(System.DefaultWorkingDirectory)/${{ parameters.cliProjectName }}'
        command: 'custom'
        customCommand: 'install --legacy-peer-deps'
        
    - task: Npm@1  
      displayName: 'Task07 - npm installation in lib'
      condition: ne(variables.CACHE_RESTORED,'true')
      inputs:
        workingDir: '$(System.DefaultWorkingDirectory)/lib-angular'
        command: 'custom'
        customCommand: 'install --legacy-peer-deps'
~~~

Build e verificações

~~~
- task: Npm@1 
      displayName: 'Task08 - project build'   
      inputs:  
        workingDir: '$(System.DefaultWorkingDirectory)/${{ parameters.cliProjectName }}'
        command: custom  
        verbose: false  
        customCommand: run build -- --prod --base-href "./"

    - script: |
        ls ${{ parameters.cliProjectName }}
      displayName: 'Task09 - Listing directory after Build'
    
    - script: |
        ls ${{ parameters.cliProjectName }}/dist/* | wc -l
      displayName: 'Task10 - Listing directory after Build'
~~~

Enviando as análises para o Sonar e verificação do Quality Gate

~~~
- task: SonarQubeAnalyze@5
      displayName: 'Task11 - QA and SAST analysis in code' 

    - task: SonarQubePublish@5
      displayName: 'Task12 - Deploying reports on SonarQube' 
      inputs:
        pollingTimeoutSec: '300'
    
    - task: Sonar-buildbreaker@8
      displayName: 'Task13 - Sonar Quality Gate' 
      inputs:
        SonarQube: 'SonarQube_Onpremise'
~~~

Compactando e Publicando artefato

~~~
 - task: ArchiveFiles@2
      displayName: 'Task14 - Build archive and zip' 
      condition: and(succeeded(),or(eq(variables.isHml, 'true'), eq(variables.isMain, 'true')),ne(variables['Build.Reason'],'PullRequest')) 
      inputs:
        rootFolderOrFile: '$(Build.SourcesDirectory)/${{ parameters.cliProjectName }}/dist'
        includeRootFolder: false
        archiveType: 'zip'
        archiveFile: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
        replaceExistingArchive: true
    
    - task: PublishBuildArtifacts@1
      displayName: 'Task15 - Deploy Solution artifact'
      condition: and(succeeded(),or(eq(variables.isHml, 'true'), eq(variables.isMain, 'true')),ne(variables['Build.Reason'],'PullRequest')) 
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
        ArtifactName: '${{ parameters.cliProjectName }}'
        publishLocation: 'Container'
~~~

Criação de uma issue em caso de falha no processo.

~~~
 - task: CreateWorkItem@1
      displayName: 'Task16 - Create work item on failure' 
      inputs:
        teamProject: '$(System.TeamProject)'
        workItemType: 'Issue'
        title: '$(System.TeamProject) - Build $(Build.BuildNumber) Failed'
        assignedTo: '$(Build.RequestedFor) <$(Build.RequestedForEmail)>'
        fieldMappings: 'Description=Branch: Branch $(Build.SourceBranch) build process fails. Go to Boards WorkItems and check the failure type.'
        associate: true
        linkWorkItems: true
        linkType: 'System.LinkTypes.Hierarchy-Forward'
        linkTarget: 'associate'
        linkPR: true
        preventDuplicates: true
        keyFields: |
          System.Title
          System.State
        updateDuplicates: true
        updateRules: 'Tags|=;Build $(Build.BuildNumber)'
      condition: failed()
~~~

### Deploy

Parametrizações do IIS para deploy

~~~
          - task: IISWebAppManagementOnMachineGroup@0
            displayName: 'Task01 - Setup for Deploy'
            inputs:
              IISDeploymentType: '$(Parameters.IISDeploymentType)'
              ParentWebsiteNameForApplication: '$(Parameters.ParentWebsiteNameForApplication)'
              WebsiteName: '$(Parameters.WebsiteName)'
              VirtualPathForApplication: '$(Parameters.VirtualPathForApplication)'
              PhysicalPathForApplication: '$(Parameters.WebsitePhysicalPath)'
              CreateOrUpdateAppPoolForApplication: '$(Parameters.CreateOrUpdateAppPoolForApplication)'
              AppPoolNameForApplication: '$(Parameters.AppPoolName)'
              DotNetVersionForApplication: '$(Parameters.DotNetVersionForApplication)'
              PipeLineModeForApplication: '$(Parameters.PipeLineModeForApplication)'
              AppPoolIdentityForApplication: '$(Parameters.AppPoolIdentityForApplication)'
              Bindings: false
              AppPoolNameForWebsite: 
              ParentWebsiteNameForVD: 
              VirtualPathForVD: 
              AppPoolName:
              AppCmdCommands: '$(Parameters.AppCmdCommands)'
~~~

Alterando variáveis em tempo de execução

~~~
- task: replacetokens@3
            displayName: 'Task03 - Replace Tokens'
            inputs:
              targetFiles: '$(Pipeline.Workspace)/$(Parameters.ProjectName)/deploy/**/main*.js'
              encoding: 'auto'
              writeBOM: true
              verbosity: 'detailed'
              actionOnMissing: 'warn'
              keepToken: false
              tokenPrefix: '#{'
              tokenSuffix: '}#'
              useLegacyPattern: false
              enableTelemetry: true
~~~

Deploy IIS onpremise

~~~
- task: IISWebAppDeploymentOnMachineGroup@0      
            displayName: '${{ parameters.displayName }}'
            inputs:
              WebSiteName: '$(Parameters.WebsiteName)'
              VirtualApplication: '$(Parameters.VirtualApplication)'
              Package: '$(Pipeline.Workspace)\$(Parameters.ProjectName)\Replaced-$(Build.BuildId).zip'
              TakeAppOfflineFlag: True
              XmlVariableSubstitution: True
~~~

SmokeTest

~~~
- task: PowerShell@2
            displayName: 'Task06 - Smoketest HTTP Status Code 200'

            inputs:
              targetType: 'inline'
              script: |
                $HTTP_Request = [System.Net.WebRequest]::Create('$(Parameters.URL)')                               
                $HTTP_Response = $HTTP_Request.GetResponse()
                $HTTP_Status = [int]$HTTP_Response.StatusCode
                If ($HTTP_Status -eq 200) {
                    Write-Host ""
                    Write-Host "Site OK! HTTP code status: $HTTP_Status"
                    Write-Host "URL verificada: $(Parameters.URL)"
                    Write-Host ""
                }
                Else {
                    Write-Host ""
                    Write-Host "Site FAIL ! HTTP code status: $HTTP_Status"
                    Write-Host "URL verificada: $(Parameters.URL)"
                    Write-Host ""
                }
                If ($HTTP_Response -eq $null) { }
                Else { $HTTP_Response.Close() }
~~~

Task para geração do release notes

~~~
task: XplatGenerateReleaseNotes@3
~~~

Publicação do release notes

~~~
- task: WikiUpdaterTask@1
~~~

Visualização dos workitems para envio de email
~~~
- task: XplatGenerateReleaseNotes@3
~~~


Envio de email através de um relay interno
~~~
- task: PowerShell@2
            condition: ${{ containsValue(parameters.branchOptions, variables['Build.SourceBranch']) }}
            continueOnError: true
            name: SendEmail
            displayName: 'Task10 - spreading email'

            inputs:
              targetType: 'inline'
              script: |
                Add-Type -AssemblyName System.Web
                $StageDecode = [System.Web.HttpUtility]::HtmlDecode("$(ViewWorkitemPub)")
                $DescPub = $StageDecode.replace('<div>','').replace('</div>','').replace('[','').replace(']','')
                $date=$(Get-Date -Format dd/MM/yyyy-HH:mm);               
                Send-MailMessage -To "xxx@xxx.com.br" -From "azuredevops@xxx.com.br" -Subject "$(Parameters.Subject) - $date" -SmtpServer "xxx.xxx.xxx.xxx" -Port xx -BodyAsHtml -Encoding utf8 -Body "$(Parameters.EmailBody) - Date: <b>$date</b><br><br>$DescPub"
  
~~~

<br>

[![Blog](https://img.shields.io/website?down_color=blue&down_message=infrati.dev&label=Blog&logo=ghost&logoColor=green&style=for-the-badge&up_color=blue&up_message=infrati.dev&url=https%3A%2F%2Finfrati.dev)](https://infrati.dev)
