# Exercício Continuous Delivery

Esta é uma aplicação baseada em Node.js utilizada para exercitar conceitos de Continuous Delivery (CD)

\* Based on the work of [Dan Arias](https://twitter.com/getDanArias): <https://auth0.com/blog/create-a-simple-and-stylish-node-express-app>

## 2. Configurando sua conta na Azure

### 2.1 Subscriptions

Você já deve ter uma subscrição criada na sua conta Azure, uma vez que ela foi gerada a partir da sua conta de estudante. Caso não haja nenhuma, utilize [este tutorial](https://docs.microsoft.com/en-us/azure/cost-management-billing/manage/create-subscription).

### 2.2 App Services

Precisamos criar um [Azure App Service](https://docs.microsoft.com/azure/app-service/app-service-plan-manage#create-an-app-service-plan) para o alvo dos nossos deployments.

1. Na tela inicial selecione `App Service` e em seguida `Create`
2. Na tela de criação do Web App, preencha as seguintes informações:
   1. **Resource Group:** `Create new`: continuous-deployment-<seu_usuario>
   2. **Name:** aula-cd-<seu_usuario>
   3. **Runtime stack:** Nodes 14 LTS
3. Clique `Review + create`
4. Clique `Create`

:tada: Pronto, já temos recursos em nuvem para receber nossa aplicação!

Agora precisamos criar um workflow para entregar o nosso app na nuvem.

### 2.3 Deployment Center

Vamos usar o _wizard_ da Azure para nos ajudar na criação do workflow.

1. No App Service que você acabou de criar, vá na opção `Deployment Center`
2. Em **Source**, selecione `GitHub`
3. Autorize o acesso a sua conta pessoal
4. **Organization:** seu usuário
5. **Repository:** sample-deployment
6. **Branch:** main
7. **Runtime stack:** Node
8. **Version:** Node 14 LTS
9. Clique `Save`

#### :question: O que mudou no seu repositório?

🚨 Nosso app está chegando na nuvem, como queríamos, mas direto no ambiente de produção. Como fazemos para que ele escale pelos ambientes de **Testes/QA** e **Homologação** antes de chegar em produção?

### 2.4 Deployment Slots

Você pode utilizar `Deployment Slots` para isolar ambientes de deployment, realizar testes A/B ou organizar a sua entrega de aplicações com estratégias Blue/Green (veremos em mais detalhes na próxima aula)

1. Selecione a opção `Deployment slots`
2. Você verá que já existe um slot com o *label* `Production`. Vamos criar novos slots. Selecione a opção `Add Slot`
3. **Nome:** qa
4. **Clone settings from:** aula-cd-<seu_usuario>
5. Clique `Add`
6. Siga o mesmo procedimento e crie um novo slot chamado `hom`

## 3. Configurando o deployment para promoção de artefatos

### 3.1 Criando novos ambientes (Environments)

1. No seu repositório no GitHub selecione `Settings` > `Environments`
2. Clique em `New environment` para criar um novo ambiente de deployment
3. Dê o nome de `QA` e clique em `Configure environment`
4. Na seção **Environment secrets**, clique em `Add Secret`
5. **Name:** AZURE_WEBAPP_PUBLISH_PROFILE
6. Agora volte para a sua conta da Azure, vá em **Deployment slots** e selecione o slot de QA (terminado em `QA`)
7. Clique em `Get publish profile`e baixe o arquivo.
8. Abra o arquivo no VSCode e cole o seu conteúdo no campo `Value` do secret que criamos no GitHub

Repita o processo processo para criar um novo ambiente chamado `HOM`. Não há problema em os secrets terem o mesmo nome. :question: Por que?

### 3.2 Ajustando o workflow

O nosso workflow agora precisa refletir a capacidade de fazer a promoção do nosso build entre ambientes. Para isso precisamos adicionar dois passos após o build e antes do deploy em produção.

1. Renomeie o seu job de `deploy` para `deploy-to-production`
2. Copie todo o conteúdo do job `deploy-to-production` e cole acima dele mesmo renomeando o job para `deploy-to-hom`
3. Substitua o valor da variável `environment.name` por `HOM`
4. Substitua o valor de `slot-name` por `hom`
5. Substitua o valor de `publish-profile` por `${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}`

Repita o processo para criar um job `deploy-to-qa` antes de `deploy-to-hom`

O seu workflow final deve ficar parecido com o abaixo (❗ lembre-se de atualizar o valor do nome do app para o que você criou e o nome dos secrets):

```yml
# Docs for the Azure Web Apps Deploy action: https://github.com/Azure/webapps-deploy
# More GitHub Actions for Azure: https://github.com/Azure/actions

name: Build and deploy Node.js app to Azure Web App - aula-cd-pedrolacerda

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  app-name: aula-cd-pedrolacerda

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - name: Set up Node.js version
      uses: actions/setup-node@v1
      with:
        node-version: '14.x'

    - name: npm install, build, and test
      run: |
        npm install
        npm run build --if-present
        npm run test --if-present

    - name: Upload artifact for deployment job
      uses: actions/upload-artifact@v2
      with:
        name: node-app
        path: .
        
  deploy-to-qa:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: 'QA'
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}

    steps:
    - name: Download artifact from build job
      uses: actions/download-artifact@v2
      with:
        name: node-app

    - name: 'Deploy to Azure Web App'
      id: deploy-to-webapp
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ env.app-name }}
        slot-name: 'qa'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: .
  
  deploy-to-hom:
    runs-on: ubuntu-latest
    needs: deploy-to-qa
    environment:
      name: 'HOM'
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}

    steps:
    - name: Download artifact from build job
      uses: actions/download-artifact@v2
      with:
        name: node-app

    - name: 'Deploy to Azure Web App'
      id: deploy-to-webapp
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ env.app-name }}
        slot-name: 'hom'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: .

  deploy-to-production:
    runs-on: ubuntu-latest
    needs: deploy-to-hom
    environment:
      name: 'production'
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}

    steps:
    - name: Download artifact from build job
      uses: actions/download-artifact@v2
      with:
        name: node-app

    - name: 'Deploy to Azure Web App'
      id: deploy-to-webapp
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ env.app-name }}
        slot-name: 'production'
        publish-profile: ${{ secrets.AzureAppService_PublishProfile_abc7038203244546b20cf50d0ab5bd7e }}
        package: .
```

#### :tada: Perfeito! Nosso app agora passa pelos ambientes de QA e de Homologação antes de chegar em Produção

#### :question: Mas e se durante a fase de QA a equipe encontrar algum problema na aplicação e não quiser promovê-la para Homologação?

## 3.3 Configurando aprovações manuais

Agora vamos configurar os nossos ambientes para que sejam necessárias revisões e aprovações antes de ser realizado o deploy.

1. No seu repositório vá em `Settings` > `Environments`
2. Selecione o ambiente `HOM`
3. Marque o *checkbox* `Required reviewers`
4. Adicione o seu próprio usuário no campo de texto e clique em `Save protectioin rules`

Repita o procedimento para o ambiente `production` mas não para o ambiente `QA`.

### :question: Execute o workflow novamente. Qual a diferença?

O nosso workflow está pronto para ser implementado em um ambiente de trabalho real! :wink:
