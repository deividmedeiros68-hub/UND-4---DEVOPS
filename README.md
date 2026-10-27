# Atividade Final — CI/CD AWS CloudFormation com GitHub Actions

Este projeto reutiliza o `template.yaml` da Atividade 3 e automatiza sua validacao e implantacao na AWS.

## Arquitetura

Internet → Application Load Balancer (HTTP/80) → EC2 Amazon Linux 2023 (Apache HTTP/80). O RDS MySQL permanece nas sub-redes privadas, com acesso TCP/3306 permitido apenas pelo Security Group da EC2. A infraestrutura tambem inclui VPC, duas sub-redes publicas em AZs diferentes, duas privadas, Internet Gateway, tabelas de rotas e Security Groups.

**Limite da demonstracao:** a pagina web comprova ALB → EC2. O banco RDS e provisionado, mas a pagina estatica nao faz uma consulta MySQL; a evidencia do RDS deve ser apresentada no console AWS/CloudFormation.

## Arquivos

```text
atividade_final_github_actions/
├── .github/workflows/
│   ├── deploy.yml      # valida em PR; implanta em push para main
│   └── destroy.yml     # exclusao MANUAL com confirmacao EXCLUIR
├── iam/
│   └── politica-confianca-oidc-exemplo.json
├── template.yaml       # EC2 + ALB + MySQL RDS + rede
├── .gitignore
├── README.md
└── roteiro_video.md
```

## 1. Preparar o GitHub

1. Crie um repositorio GitHub (por exemplo, `atividade-final-aws`).
2. Envie todos os arquivos **preservando a pasta oculta `.github/workflows/`**.
3. Utilize a branch `main`.
4. Em **Settings → Secrets and variables → Actions**:
   - Em **Secrets**, cadastre `AWS_ROLE_ARN` com o ARN da role IAM assumida pelo GitHub Actions (por exemplo, `arn:aws:iam::123456789012:role/GitHubActionsDeployLab`).
   - Em **Variables**, cadastre `AWS_REGION` com uma regiao AWS que ofereca os tipos EC2 e RDS escolhidos (por exemplo, `us-east-1`).
5. No repositorio, permita execucao de GitHub Actions conforme as politicas da organizacao.

## 2. Autenticar por OIDC (sem access keys permanentes)

1. No console AWS, em **IAM → Identity providers**, registre um provedor OpenID Connect com URL `https://token.actions.githubusercontent.com` e audience `sts.amazonaws.com`, se ainda nao existir.
2. Crie uma role IAM com trusted entity do tipo Web identity usando esse provedor.
3. Use `iam/politica-confianca-oidc-exemplo.json` como **modelo**, substituindo ID_DA_CONTA, USUARIO e REPOSITORIO. Restrinja a confianca a identidade concreta do seu repositorio e a branch `main`.
4. **Atencao:** repositorios criados a partir de julho de 2026 podem usar um formato de claim `sub` com IDs do proprietario e do repositorio, diferente do formato tradicional demonstrado no JSON. Confira a documentacao atual da acao `aws-actions/configure-aws-credentials` e configure a claim exata do repositorio antes de executar. Nao amplie a confianca para `repo:*` como atalho.
5. Anexe a role uma politica de permissoes que autorize o CloudFormation e a criacao/alteracao/exclusao dos recursos EC2/VPC/ELBv2/RDS usados pelo template, alem de consultas SSM para a AMI e acesso aos recursos envolvidos na senha gerenciada do RDS. Para laboratorios sob sua propria conta isolada pode-se testar inicialmente com permissoes amplas **temporarias**, retirando-as apos a pratica; em contas institucionais solicite uma politica de menor privilegio. Nao use as chaves da conta root.
6. Adicione o ARN da role como secret `AWS_ROLE_ARN` no GitHub.

Documentacao oficial: https://github.com/aws-actions/configure-aws-credentials

## 3. Enviar codigo

```bash
git init
git branch -M main
git add .
git commit -m "Atividade final: CI/CD CloudFormation"
git remote add origin https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git
git push -u origin main
```

> Execute os comandos na pasta raiz do projeto. Se o repositorio ja possui commits, sincronize antes de enviar.

## 4. Pipeline e comandos equivalentes ao Terraform

O projeto usa **CloudFormation** (o mesmo IaC da Atividade 3):

| Terraform | Equivalente no pipeline CloudFormation |
|---|---|
| `terraform init` | Nao se aplica: AWS CloudFormation e um servico gerenciado |
| `terraform plan` | Validacao com `cfn-lint` e `aws cloudformation validate-template` (nao sao planos de mudancas) |
| `terraform apply` | `aws cloudformation deploy --template-file template.yaml --stack-name atividade-final-iac` |
| `terraform destroy` | `aws cloudformation delete-stack` no workflow manual |

**Observacao:** se for exigido um plano de alteracoes equivalente ao `terraform plan`, crie e inspecione um **CloudFormation Change Set** antes de executa-lo. O pipeline fornecido valida e faz deploy diretamente, sem etapa de aprovacao de changeset.

- **Pull request para `main`:** lint do template, sem credenciais AWS e sem deploy.
- **Push para `main`:** lint, federacao OIDC, validacao remota, deploy/update da stack, recuperacao da URL e teste HTTP com `curl`.
- **Run workflow manual:** opcao adicional para acionar deploy.
- **Artifacts:** `template-cloudformation-<run_number>` armazenado por 7 dias.
- **Logs:** consulte `Actions → CI-CD CloudFormation AWS → execucao → jobs` e o resumo do job.
- **State:** CloudFormation controla o estado da stack internamente; nao existe arquivo `.tfstate` para salvar em S3 neste projeto.

## 5. Confirmar recursos AWS

1. Abra `Actions`, acompanhe o job `Verificacao sintatica do template` e o job `Deploy na AWS` (indicados em verde apos sucesso).
2. Nos logs de deploy, veja o status da stack e a URL do ALB. Abra-a no navegador: deve mostrar **Aplicação EC2 funcionando!**.
3. No Console AWS, abra CloudFormation → stack `atividade-final-iac` (status esperado `CREATE_COMPLETE` ou `UPDATE_COMPLETE`).
4. Verifique EC2 → Instances, EC2 → Load Balancers → Target groups (`healthy`) e RDS → Databases (status `Available`). Aguarde a inicializacao e os health checks, se necessario.
5. A confirmacao de status AWS nao foi executada durante a preparacao deste projeto: so sera conhecida apos o pipeline rodar na sua conta.

## 6. Evitar cobrancas: exclusao apos a demonstracao

Em `Actions → Excluir infraestrutura AWS (manual) → Run workflow`, selecione branch `main`, digite `EXCLUIR` e execute. O workflow chama `delete-stack` e espera a conclusao. Confira CloudFormation, EC2, ALB e RDS no console.

O ALB e o RDS podem gerar cobrancas mesmo sem trafego. O RDS esta definido com `DeletionPolicy: Delete`, sem protecao contra exclusao e sem backups automaticos, **apenas para o laboratorio**. O banco sera apagado e os dados perdidos na exclusao. Nao use este template como configuracao de producao. O endpoint do site e HTTP, nao HTTPS. Nao exponha dados sensiveis.

## Problemas comuns

- **Not authorized to perform sts:AssumeRoleWithWebIdentity:** verifique provider OIDC, ARN, audience e formato `sub` da trust policy.
- **AccessDenied em CloudFormation ou outros servicos:** a role necessita das permissoes adequadas para todos os recursos provisionados.
- **Insufficient capacity / instance not supported:** ajuste `EC2InstanceType`, `DBInstanceClass` ou `AWS_REGION` conforme a disponibilidade regional.
- **Stack ROLLBACK_COMPLETE:** examine Events no CloudFormation; exclua a stack que falhou antes de nova criacao, apos diagnosticar a causa.
- **Health check unhealthy:** espere inicializacao do Apache e consulte os eventos EC2/Target Group; verifique grupo de seguranca porta 80.
- **RDS demorando:** criacao e exclusao de bancos podem levar varios minutos.

## Fontes

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudformation/deploy/index.html
- https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
- https://github.com/aws-actions/configure-aws-credentials
