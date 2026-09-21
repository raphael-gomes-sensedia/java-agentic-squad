---
name: sonarqube-quality-gate
description: Orientações de análise de métricas via MCP para SonarQube Community Edition v9.9.
---

Quando for solicitado o relatório de qualidade ou code review do SonarQube, utilize a ferramenta `bash` para executar as requisições `curl` abaixo.

## Buscar Status do Quality Gate
Execute no terminal:
```bash
curl -s -u "$SONAR_TOKEN:" "$SONAR_URL/api/qualitygates/project_status?projectKey=$PROJECT_KEY"