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