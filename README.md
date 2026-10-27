# CI-reusable-source

Workflow compartilhado para testes, análise de segurança e publicação de imagens
Go no ECR. As publicações ocorrem por push ou execução manual na `aws-cloud-k8s`;
pull requests não acessam a AWS nem publicam imagens.

## Autenticação AWS

`aws-auth-mode` aceita `oidc` (padrão) ou `academy`.

- **OIDC:** informe `aws-role-arn` e conceda `id-token: write` no workflow chamador.
- **Academy:** passe o secret `AWS_ACADEMY_CREDENTIALS` com o bloco de credenciais
  do laboratório. Não precisa de `aws-role-arn` nem de `id-token: write`.

Exemplo de chamada para o Academy; substitua `COMMIT_SHA` pelo commit do workflow:

```yaml
permissions:
  contents: read

jobs:
  ci:
    uses: ramondata/CI-reusable-source/.github/workflows/ci-reusable.yml@COMMIT_SHA
    with:
      aws-auth-mode: academy
      aws-region: ${{ vars.AWS_REGION }}
      ecr-repository: ${{ vars.ECR_REPOSITORY }}
      image-name: auth-service
    secrets:
      AWS_ACADEMY_CREDENTIALS: ${{ secrets.AWS_ACADEMY_CREDENTIALS }}
```

O secret contém `aws_access_key_id`, `aws_secret_access_key` e `aws_session_token`,
com ou sem cabeçalho `[default]`. O parser valida os campos, mascara os valores
nos logs e não executa o conteúdo como comandos. Renove o bloco inteiro quando a
sessão do Academy mudar; as credenciais não são renovadas pelo workflow.

O workflow usa as permissões da sessão do Academy diretamente, sem tentar
`AssumeRole` ou `AssumeRoleWithWebIdentity`. O ECR deve existir na conta e região
configuradas. A autenticação é validada antes da construção da imagem.

Os outputs `image-tag` e `image-uri` informam a tag e o endereço completo da
imagem publicada para consumidores GitOps. As verificações de Go, lint, SCA e
SAST existentes são preservadas. O modo OIDC continua sendo o padrão para os
demais consumidores.

## Serviços Python

O arquivo `.github/workflows/ci-python-reusable.yml` oferece o mesmo contrato
AWS e os mesmos outputs do fluxo Go, usando Python 3.12, pytest, Ruff, Bandit
e Trivy. A publicação permanece restrita à `aws-cloud-k8s`, por push ou execução
manual. O modo Academy utiliza o mesmo parser de credenciais e a mesma
configuração AWS já validada pelo auth-service.

O input booleano `tests-required` é `true` por padrão. Consumidores sem testes
existentes, como analytics-service, podem definir `false`: somente a ausência
de testes (código 5 do pytest) vira aviso. Falhas de testes e de coleta continuam
bloqueando o CI. A compilação sintática é sempre executada. As regras de Ruff,
o gate alto/alta confiança do Bandit e os gates CRITICAL do Trivy são mantidos.
O scan de imagem Python preserva `--ignore-unfixed` do workflow Python anterior.
