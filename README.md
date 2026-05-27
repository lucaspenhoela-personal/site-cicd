Pipeline CI/CD com GitHub Actions + AWS S3

Projeto de estudo de DevOps/Cloud demonstrando deploy automatizado de um site estático para AWS S3 usando GitHub Actions com autenticação via OIDC (sem credenciais armazenadas).

 Sobre o Projeto
Este projeto implementa uma pipeline de Continuous Deployment (CD) completa: a cada git push na branch main, o GitHub Actions executa um workflow que sincroniza automaticamente os arquivos do repositório com um bucket Amazon S3 configurado como site estático.
A autenticação entre o GitHub e a AWS é feita via OpenID Connect (OIDC), eliminando a necessidade de armazenar credenciais de longa duração (Access Keys) como secrets — uma prática de segurança recomendada em ambientes corporativos.
🎯 Objetivos de aprendizado

Configurar pipelines CI/CD com GitHub Actions
Implementar autenticação federada via OIDC entre GitHub e AWS
Provisionar e configurar buckets S3 para hosting estático
Aplicar o princípio do menor privilégio em políticas IAM
Praticar troubleshooting de pipelines e debugging de IAM


🏗️ Arquitetura
┌─────────────┐    git push     ┌──────────────────┐    OIDC Auth    ┌──────────┐
│   GitHub    │────────────────▶│  GitHub Actions  │────────────────▶│   AWS    │
│  Repository │                 │    Workflow      │   (sts:Assume)  │   IAM    │
└─────────────┘                 └────────┬─────────┘                 └────┬─────┘
                                         │                                │
                                         │ aws s3 sync                    │ assume role
                                         ▼                                ▼
                                ┌──────────────────────────────────────────┐
                                │           AWS S3 Bucket                  │
                                │      (Static Website Hosting)            │
                                └──────────────────────────────────────────┘
                                                  │
                                                  ▼
                                          🌐 Site publicado
Fluxo de execução

Desenvolvedor faz git push na branch main
GitHub Actions detecta o evento e dispara o workflow deploy.yml
O workflow solicita um token OIDC temporário ao GitHub
A action aws-actions/configure-aws-credentials troca o token OIDC por credenciais temporárias da AWS via sts:AssumeRoleWithWebIdentity
O comando aws s3 sync envia os arquivos do repositório para o bucket S3
O S3 serve o conteúdo via URL pública de Static Website Hosting


🛠️ Stack Tecnológica
CategoriaTecnologiaCloud ProviderAWS (Amazon Web Services)Armazenamento/HostingAmazon S3 (Static Website Hosting)Identidade e AcessoAWS IAM, OIDC Identity ProviderCI/CDGitHub ActionsLinguagensHTML5, CSS3, YAMLVersionamentoGit, GitHub

📂 Estrutura do Projeto
site-cicd/
│
├── .github/
│   └── workflows/
│       └── deploy.yml          # Pipeline CI/CD do GitHub Actions
│
├── index.html                  # Página estática hospedada no S3
│
└── README.md                   # Este arquivo

⚙️ Como funciona o workflow
yamlname: Deploy site estático para S3

on:
  push:
    branches: [ main ]

permissions:
  id-token: write   # Permite obter o token OIDC
  contents: read    # Permite ler o código do repositório

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Baixar código do repositório
        uses: actions/checkout@v4

      - name: Configurar credenciais AWS via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsDeployRole
          aws-region: us-east-1

      - name: Sincronizar arquivos com o bucket S3
        run: |
          aws s3 sync . s3://<NOME_DO_BUCKET> \
            --exclude ".git/*" \
            --exclude ".github/*" \
            --exclude "README.md" \
            --delete
Por que OIDC e não Access Keys?
Access Keys (forma antiga)OIDC (forma moderna) ✅Credenciais de longa duração armazenadas como secretsTokens temporários (15 min a 1h)Risco de vazamento permanente em caso de comprometimentoRotação automática a cada execuçãoDifícil de auditar quem usouCada execução é rastreável via CloudTrailRequer rotação manualSem necessidade de rotação

🔐 Configuração de Segurança IAM
Identity Provider (OIDC)

Provider URL: https://token.actions.githubusercontent.com
Audience: sts.amazonaws.com

IAM Role: GitHubActionsDeployRole
Trust Policy (configurada para aceitar apenas pushes do repositório específico):
json{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:lucaspenhoela-personal/site-cicd:*"
        }
      }
    }
  ]
}
Permissions: AmazonS3FullAccess (em produção, recomenda-se uma policy customizada com escopo restrito ao bucket específico).

🐛 Desafios enfrentados e soluções
Durante a implementação, alguns problemas reais surgiram e foram resolvidos — parte essencial do aprendizado em DevOps:
ProblemaCausaSoluçãoErro 404 NoSuchKey ao acessar o siteUpload do index.html não chegou ao S3 na primeira execuçãoAjuste de variáveis no workflow e novo commitWorkflow não disparava (0 runs)Remote local apontava para repositório diferentegit remote remove + git remote add com URL corretaFalha na autenticação OIDCTrust policy da IAM Role apontava para repo antigoAtualização do sub no trust policy para repo:user/site-cicd:*git push retornava "Everything up-to-date" sem disparar workflowSem novos commits para enviarUso de git commit --allow-empty para forçar trigger

🚀 Como reproduzir este projeto
Pré-requisitos

Conta AWS (Free Tier elegível)
Conta GitHub
AWS CLI configurado localmente
Git instalado

Passo a passo resumido

Criar bucket S3 com Static Website Hosting habilitado e bucket policy pública
Configurar OIDC Provider na AWS apontando para token.actions.githubusercontent.com
Criar IAM Role com trust policy restrita ao repositório GitHub e permissões S3
Criar repositório GitHub com index.html e workflow .github/workflows/deploy.yml
Substituir variáveis no workflow (Account ID e nome do bucket)
git push e ver a mágica acontecer 🎉


📚 Tutorial completo com passo a passo detalhado disponível em guia-projetos-aws.md (se aplicável)


💡 Aprendizados-chave

OIDC é o padrão moderno para CI/CD em cloud — qualquer projeto novo deve usá-lo em vez de Access Keys
Trust policies são literais: nomes de repositório, branches e organizações precisam bater exatamente
Permissões IAM são a causa de ~80% dos erros em projetos cloud — sempre comece por aí no debug
aws s3 sync --delete mantém o bucket em espelho com o repo (remove arquivos deletados localmente)
GitHub Actions só dispara em pushes com mudanças reais — --allow-empty é útil pra forçar reexecução



📚 Referências

AWS — Configuring OpenID Connect in Amazon Web Services
GitHub Actions — aws-actions/configure-aws-credentials
AWS S3 — Hosting a static website
AWS IAM Best Practices


👨‍💻 Autor
Lucas Penhoela Ribeiro

GitHub: @lucaspenhoela-personal
