# Projeto: GitOps na Prática com a Online Boutique

Este projeto demonstra a implantação de uma aplicação de microserviços ("Online Boutique") em um cluster Kubernetes local, utilizando práticas de GitOps com ArgoCD. O objetivo é usar um repositório Git como a única fonte da verdade para definir e gerenciar o estado da aplicação no cluster.

## Tecnologias Utilizadas

* **Kubernetes:** Orquestrador de contêineres.
* **Docker Desktop:** Para rodar um cluster Kubernetes localmente no Windows.
* **ArgoCD:** Ferramenta de GitOps para entrega contínua e sincronização com o repositório Git.
* **Git & GitHub:** Para versionamento e como fonte da verdade da nossa configuração.
* **kubectl:** A ferramenta de linha de comando para interagir com o cluster Kubernetes.

---

## Passo a Passo da Implementação

### 1. Pré-requisitos

Antes de começar, garanta que os seguintes softwares estão instalados e configurados:
* Docker Desktop com o Kubernetes habilitado nas configurações.
* Git instalado.
* `kubectl` instalado e configurado para comunicar com o cluster do Docker Desktop.
* Uma conta no GitHub.

### 2. Preparação do Repositório Git

O GitOps precisa de um repositório para monitorar.

1.  **Fork do Projeto Original:** Faça um "fork" do repositório oficial da aplicação para a sua conta do GitHub: [https://github.com/GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo).
2.  **Crie seu Repositório GitOps:** Crie um novo repositório público na sua conta do GitHub. Este será o repositório que o ArgoCD irá observar.
3.  **Estruture os Manifestos:**
    * Clone o repositório que você criou no passo 2 para a sua máquina local.
    * No fork do projeto original, localize o arquivo `release/kubernetes-manifests.yaml`.
    * Dentro do seu repositório local, crie a seguinte estrutura de pastas e copie o arquivo:
        ```
        /
        └── k8s/
            └── online-boutique.yaml
        ```
4.  **Envie para o GitHub:** Faça o commit e o push da nova estrutura para o seu repositório.
    ```bash
    git add .
    git commit -m "Adiciona manifesto da aplicação"
    git push
    ```

### 3. Instalação e Acesso ao ArgoCD

1.  **Instale o ArgoCD** no seu cluster Kubernetes:
    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
    ```
2.  **Acesse a Interface Web:** O ArgoCD não fica exposto por padrão. Use `port-forward` para acessá-lo. Mantenha este comando rodando em um terminal.
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
3.  **Obtenha a Senha de Admin:** A senha inicial é gerada automaticamente.
    * **No Linux ou macOS:**
        ```bash
        kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
        ```
    * **No Windows (PowerShell):**
        ```powershell
        $passwordEncoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
        [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($passwordEncoded))
        ```
4.  **Faça o Login:** Abra o navegador em `https://localhost:8080`. Use o usuário `admin` e a senha obtida no passo anterior.

### 4. Criação da Aplicação no ArgoCD

1.  Na interface do ArgoCD, clique em **+ NEW APP**.
2.  Preencha os campos da seguinte forma:
    * **Application Name**: `online-boutique`
    * **Project Name**: `default`
    * **Sync Policy**: `Manual`
    * **Repository URL**: A URL do *seu* repositório GitOps criado no passo 2.
    * **Revision**: `HEAD`
    * **Path**: `k8s`
    * **Cluster URL**: `https://kubernetes.default.svc`
    * **Namespace**: `default`
3.  Clique em **CREATE**. A aplicação aparecerá com o status `OutOfSync`.

### 5. Sincronização e Acesso à Loja

1.  **Sincronize a Aplicação:** Clique no card da aplicação, depois no botão **SYNC** e confirme em **SYNCHRONIZE**. O ArgoCD começará a criar todos os recursos no Kubernetes.
2.  **Aguarde a Saúde:** Espere até que o status da aplicação fique `Healthy` (verde).
3.  **Acesse o Frontend:** A loja também precisa de `port-forward`. Em um novo terminal, rode:
    ```bash
    kubectl port-forward svc/frontend-external 8081:80
    ```
4.  Acesse a loja no seu navegador: `http://localhost:8081`.

### 6. GitOps em Ação (Opcional)

Para ver a mágica do GitOps, vamos escalar um dos serviços.

1.  **Edite o Manifesto:** Abra o arquivo `k8s/online-boutique.yaml` localmente.
2.  **Encontre o `recommendationservice`:** Localize o `Deployment` com `name: recommendationservice`.
3.  **Adicione a Chave de Réplicas:** Este `Deployment` não possui a chave `replicas` por padrão. Adicione-a para escalar o serviço.
    ```yaml
    # ...
    spec:
      replicas: 3 # <-- ADICIONE ESTA LINHA
      selector:
    # ...
    ```
4.  **Faça o Commit e o Push:** Salve a alteração e envie para o GitHub.
5.  **Observe no ArgoCD:** Dê um **REFRESH** na aplicação. Você verá que ela ficará `OutOfSync`.
6.  **Sincronize Novamente:** Clique em **SYNC** para que o ArgoCD aplique a mudança, escalando o serviço para 3 pods.

---