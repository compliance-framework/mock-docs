---
sidebar_position: 1
sidebar_label: O Continuous Compliance Framework
---

# O que é o Continuous Compliance Framework?

O Continuous Compliance Framework (CCF) é um sistema de código aberto para testes e relatórios automatizados de conformidade. Ele ajuda você e sua organização a avaliar continuamente a conformidade com normas como NIST SP 800-53, SOC 2, PCI DSS e GDPR. O CCF integra verificações baseadas em políticas, monitoramento em tempo real e relatórios estruturados para apoiar auditorias.

O CCF promove a verificação contínua e a colaboração na comunidade de governança, risco e controle (GRC). Seus colaboradores participam dos grupos de trabalho OSCAL, CNCF Compliance e NIST sobre automação da conformidade e de relatórios.

## Principais recursos

### Código aberto

O CCF é um projeto de código aberto.

### Verificação automatizada e contínua de conformidade

Com o CCF, sua organização coleta evidências e valida a conformidade com requisitos regulatórios, de segurança e operacionais de forma contínua e automatizada. Isso reduz o trabalho manual na elaboração de relatórios de conformidade.

Notificações e alertas em tempo real permitem responder a desvios dos requisitos.

### API centralizada de conformidade

A API centraliza os dados de conformidade para coleta e elaboração de relatórios.

### Extensibilidade

A API aceita dados compatíveis com OSCAL de outras ferramentas. Você pode enviar informações de conformidade dessas ferramentas e incluí-las nos relatórios da sua organização.

Você pode integrar ferramentas de diferentes categorias:

- Provedores de serviços em nuvem
  - AWS
  - Azure
  - GCP
  - Alibaba
  - Oracle
- Ambientes de execução de contêineres
  - Kubernetes
- Sistemas de controle de versão
  - GitHub
  - GitLab
- Analisadores de vulnerabilidades em contêineres
  - Clair
  - Trivy
  - Docker
- Analisadores de segurança de infraestrutura
  - Terraform
  - GitLab
  - Snyk
- Analisadores de software de código aberto e de conformidade de licenças (SCA, SAST, DAST)
  - SonarQube
  - GitLab
  - Snyk
  - Black Duck

### Arquitetura de plugins com gRPC

Os programas CCF Agent coletam informações de conformidade continuamente por meio de CCF Plugins. Eles validam essas informações com CCF Policies definidas em código e enviam os resultados à API.

A arquitetura de plugins permite criar novos tipos de verificações e validações de conformidade. Assim, você pode incluir diferentes fontes de dados na avaliação de conformidade.

### Políticas como código

O CCF usa Rego, do OPA, e Kyverno-JSON como mecanismos de políticas para validar as informações coletadas pelos plugins. Você pode gerenciar as políticas da sua organização de forma contínua e iterativa e associá-las a controles regulatórios para elaborar relatórios de auditoria.

### Plugins e políticas compatíveis com OCI

O CCF permite distribuir políticas de conformidade e plugins de forma gradual por meio de registros OCI.

### Relatórios compatíveis com OSCAL

O CCF gera relatórios estruturados no formato OSCAL do NIST. Você pode compartilhá-los com auditores para verificar a conformidade regulatória e gerenciar o ciclo de GRC.

- Painéis de conformidade em tempo real
- Relatórios abrangentes de conformidade
- Automação dos processos de elaboração de relatórios e auditoria
- Exportação de dados e relatórios de conformidade nos formatos PDF e OSCAL
