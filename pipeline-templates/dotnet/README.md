[![Blog](https://img.shields.io/website?down_color=blue&down_message=infrati.dev&label=Blog&logo=ghost&logoColor=green&style=for-the-badge&up_color=blue&up_message=infrati.dev&url=https%3A%2F%2Finfrati.dev)](https://infrati.dev)

#### 📋 infratidev

##### Template Pipeline MultiStage AzureDevops para dotnet. Build e Deploy em ambiente onpremise.

### Build/QA/SAST

Parametros passados para o ```azure-pipelines.yml```

~~~
parameters:
  - name: projectKey
    type: string
  - name: projectName
    type: string
~~~

Parametrização do Sonar

~~~
- task: SonarQubePrepare@5
      displayName: 'Task01 - QA and SAST analysis in code'

      inputs:
        SonarQube: 'SonarQube_Onpremise'
        scannerMode: 'MSBuild'
        projectKey: '${{ parameters.projectKey }}'
        projectName: '${{ parameters.projectName }}'
        extraProperties: |
          #sonar.verbose=true
~~~

Instalação do NugetTool e Restore da Solution

 ~~~
- task: NuGetToolInstaller@1
      displayName: 'Task02 - Installing NuGetTool'

    - task: NuGetCommand@2
      displayName: 'Task03 - Restoring Solution Packages'
      inputs:
        restoreSolution: '$(solution)'
 ~~~

Build 

~~~
 - task: VSBuild@1
      displayName: 'Task04 - Solution Build' 
      inputs:
        solution: '$(solution)'
        msbuildArgs: '/p:DeployOnBuild=true /p:WebPublishMethod=Package /p:PackageAsSingleFile=true /p:SkipInvalidConfigurations=true /p:PackageLocation="$(build.ArtifactStagingDirectory)"'
        platform: '$(buildPlatform)'
        configuration: '$(buildConfiguration)'
~~~

Enviando as análises para o Sonar e verificação do Quality Gate

~~~
    - task: SonarQubeAnalyze@5
      displayName: 'Task05 - QA and SAST analysis in code' 
      continueOnError: true

    - task: SonarQubePublish@5
      displayName: 'Task06 - Deploying reports on SonarQube'
      continueOnError: true 
      inputs:
        pollingTimeoutSec: '300'
    
    - task: Sonar-buildbreaker@8
      displayName: 'Task07 - Sonar Quality Gate' 
      continueOnError: true
      inputs:
        SonarQube: 'SonarQube_Onpremise'  
~~~

Publicando artefatos

~~~
 - task: PublishPipelineArtifact@1
      condition: and(succeeded(),or(eq(variables.isHml, 'true'), eq(variables.isMain, 'true')),ne(variables['Build.Reason'],'PullRequest'))  
      displayName: 'Task08 - Deploy Pipeline Solution artifact'
      inputs:
        platform: '$(buildPlatform)'
        configuration: '$(buildConfiguration)'

    - task: PublishBuildArtifacts@1
      condition: and(succeeded(),or(eq(variables.isHml, 'true'), eq(variables.isMain, 'true')),ne(variables['Build.Reason'],'PullRequest'))  
      displayName: 'Task09 - Deploy Build Solution artifact'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: '${{ parameters.projectName }}' 
~~~

Criação de uma issue em caso de falha no processo.

~~~
    - task: CreateWorkItem@1
      displayName: 'Task10 - Create work item on failure' 
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



<br>

[![Blog](https://img.shields.io/website?down_color=blue&down_message=infrati.dev&label=Blog&logo=ghost&logoColor=green&style=for-the-badge&up_color=blue&up_message=infrati.dev&url=https%3A%2F%2Finfrati.dev)](https://infrati.dev)
