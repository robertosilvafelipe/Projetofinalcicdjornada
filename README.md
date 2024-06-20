# Projetofinalcicdjornada

## Introduction 
Este projeto implementa uma pipeline de Integração e Entrega Contínua (CI/CD) utilizando o Azure DevOps. A pipeline está configurada para executar testes automáticos, construir uma imagem Docker e realizar o push para um registro de containers. Além disso, a pipeline está configurada para implantar a aplicação em um ambiente de Container Apps no Azure.

## Getting Started

### 1. Installation process
Para configurar este projeto no seu ambiente local, siga os passos abaixo:

1. Clone o repositório do Azure DevOps para sua máquina local:
    ```bash
    git clone <URL_DO_REPOSITORIO_AZURE_DEVOPS>
    ```
2. Navegue até o diretório clonado:
    ```bash
    cd <NOME_DO_DIRETORIO>
    ```

### 2. Software dependencies
As dependências de software para este projeto incluem:

- **Azure DevOps**: Para configuração e execução das pipelines.
- **Docker**: Para construção e push das imagens Docker.
- **Python 3.x**: Para execução dos testes automatizados.
- **Terraform**: (opcional, para configuração de infraestrutura como código em um momento posterior).

### 3. Latest releases
Informações sobre as últimas versões podem ser encontradas no histórico de commits do repositório.

### 4. API references
Não se aplicam diretamente a este projeto, pois ele se concentra em CI/CD e deploy de aplicações.

## Build and Test

### Pipeline de CI/CD
A configuração da pipeline está definida no arquivo `azure-pipelines.yaml`.

#### Arquivo `azure-pipelines.yaml`

```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: 'variable-group-app'  # Nome do variable group criado no Azure DevOps

stages:
- stage: Test
  jobs:
  - job: RunTests
    displayName: 'Run Tests'
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'
        addToPath: true
    - script: |
        python -m venv venv
        source venv/bin/activate
        pip install -r requirements.txt
        cd tests
        pytest
      displayName: 'Run Unit Tests'

- stage: Build
  dependsOn: Test
  jobs:
  - job: BuildDockerImage
    displayName: 'Build Docker Image'
    steps:
    - script: |
        echo $(DOCKER_HUB_PASSWORD) | docker login -u $(DOCKER_HUB_USERNAME) --password-stdin
      displayName: 'Login to Docker Hub'
    - script: |
        docker info
        docker images
      displayName: 'Docker Info and Images'
    - task: Docker@2
      inputs:
        containerRegistry: 'sc-dockerhub-cicd'  # Referência à conexão de serviço configurada na biblioteca
        repository: 'robertosilvafelipe/nginx-custom'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        buildContext: '$(Build.SourcesDirectory)'
        tags: |
          $(Build.BuildId)
    - script: |
        docker images
        echo "Docker build and push completed successfully"
      displayName: 'Log Build Completion'

- stage: Deploy
  dependsOn: Build
  jobs:
  - deployment: DeployToContainerApps
    displayName: 'Deploy to Azure Container Apps'
    environment: 'staging'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              az login --service-principal -u $(servicePrincipalId) -p $(servicePrincipalKey) --tenant $(tenantId)
              az containerapp update --name $(containerAppName) --resource-group $(resourceGroup) --image robertosilvafelipe/nginx-custom:$(Build.BuildId)
            displayName: 'Deploy to Azure Container Apps'
          - script: |
              echo "Deployment initiated, waiting for application to start..."
              sleep 10
              app_url=$(az containerapp show --name $(containerAppName) --resource-group $(resourceGroup) --query "properties.configuration.ingress.fqdn" -o tsv)
              echo "Application URL: https://$app_url"
              response=$(curl -s -o /dev/null -w "%{http_code}" https://$app_url)
              echo "HTTP Response Code: $response"
              if [ $response -eq 200 ]; then
                echo "Application is running successfully."
              else
                echo "Application is not responding as expected."
                exit 1
              fi
            displayName: 'Verify Application Status'
