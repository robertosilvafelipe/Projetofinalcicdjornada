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
```

### Executando a Pipeline no Azure DevOps

1. **Service Connection**: Configure uma conexão de serviço para o Docker Hub e outra para o Azure Container Apps. Certifique-se de que as credenciais estão corretas e que a conexão está funcionando.

2. **Agent Pool**: Utilize o agente padrão do Azure DevOps `ubuntu-latest` conforme especificado na pipeline.

3. **Variable Group**: Crie um grupo de variáveis no Azure DevOps chamado `variable-group-app` e adicione as seguintes variáveis:
   - `DOCKER_HUB_USERNAME`: Seu nome de usuário do Docker Hub.
   - `DOCKER_HUB_PASSWORD`: Sua senha do Docker Hub.
   - `servicePrincipalId`: ID do service principal para login no Azure.
   - `servicePrincipalKey`: Chave do serviço principal para login no Azure.
   - `tenantId`: ID do tentant do Azure.
   - `containerAppName`: Nome do container apps no Azure.
   - `resourceGroup`: Resource Group no Azure.

4. **Pipeline Execution**:

    #### Criando a Pipeline

    1. **Navegue até a seção de Pipelines**: No Azure DevOps, vá até o seu projeto e clique em "Pipelines" no menu lateral esquerdo.

    2. **Criar Nova Pipeline**: Clique em "New Pipeline" para iniciar o processo de criação de uma nova pipeline.

    3. **Selecionar Repositório**: Escolha a opção onde está armazenado o seu código. Normalmente, você selecionará "Azure Repos Git" se estiver usando um repositório hospedado no Azure DevOps. Selecione o repositório que contém o arquivo `azure-pipelines.yaml`.

    4. **Configurar Pipeline**: Na próxima tela, selecione a opção "YAML" para definir a pipeline a partir de um arquivo YAML existente. Procure e selecione o arquivo `azure-pipelines.yaml` no seu repositório.

    5. **Revisar e Salvar**: Revise a configuração da pipeline conforme definida no arquivo YAML. Se tudo estiver correto, clique em "Save" e depois em "Run" para iniciar a execução da pipeline.

    #### Executando a Pipeline

    Após salvar e iniciar a pipeline, você pode acompanhar o progresso das etapas através do painel do Azure DevOps. A pipeline executará as seguintes etapas:

    - **Executando Testes**: A primeira etapa da pipeline é executar testes automáticos. A pipeline configurará um ambiente Python, instalará as dependências e executará os testes definidos no diretório `tests`.
    - **Construindo Imagem Docker**: Após os testes serem executados com sucesso, a pipeline construirá uma imagem Docker e realizará o push para o registro de containers configurado (Docker Hub).
    - **Deploy**: Finalmente, a pipeline fará o deploy da aplicação para o Azure Container Apps, utilizando as credenciais e configurações definidas.


   ![Estrutura do Projeto](images\cicdpipeline.png)

5. **Verificar Resultados**: Após a execução da pipeline, verifique os logs e resultados para assegurar que todas as etapas foram concluídas com sucesso. 

    - **Logs de Testes**: Verifique se todos os testes passaram.
    - **Logs de Build**: Confirme que a imagem Docker foi construída e enviada corretamente.
    - **Logs de Deploy**: Assegure-se de que o deploy foi realizado sem erros e que a aplicação está funcionando corretamente no ambiente de destino.
