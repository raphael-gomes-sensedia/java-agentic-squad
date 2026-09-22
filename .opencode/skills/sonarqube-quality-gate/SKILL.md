---
name: sonarqube-quality-gate
description: Gera o relatório de qualidade (quality gate) do SonarQube Community Edition v9.9 via curl no terminal. Use sempre que o usuário pedir o relatório de qualidade, o status do quality gate, ou um code review com base no SonarQube.
---

# SonarQube Quality Gate

## Quando usar
Sempre que for solicitado o relatório de qualidade, o status do quality gate, ou um code review fundamentado no SonarQube do projeto. Vale para SonarQube Community Edition v9.9, em que os assistants MCP nativos de métricas não estão disponíveis — por isso as consultas são feitas via `curl` com a ferramenta `bash`.

## Instruções (passo a passo)
1. Confirme que as variáveis de ambiente `$SONAR_TOKEN` (token de autenticação), `$SONAR_URL` (endpoint do servidor) e `$PROJECT_KEY` (chave do projeto) estão definidas; se alguma não estiver, sinalize e pare — não invente valores.
2. Busque o status do quality gate executando no terminal:

```bash
curl -s -u "$SONAR_TOKEN:" "$SONAR_URL/api/qualitygates/project_status?projectKey=$PROJECT_KEY"
```

3. Interprete a resposta JSON: o campo `projectStatus.status` será `OK` (gate aprovado) ou `ERROR` (gate reprovado); em caso de `ERROR`, liste as `conditions` cujo campo `status` seja `ERROR`, com `metricKey`, `actualValue`, `comparator` e `errorThreshold`.
4. Apresente o resultado ao usuário indicando: status do gate, métricas que reprovaram (quando houver) e o valor atual versus o limiar configurado de cada métrica reprovada.
5. Se a requisição falhar (timeout, 401 ou 404), verifique as variáveis de ambiente e a conectividade com `$SONAR_URL`; não retente indefinidamente — sinalize o erro comum (token inválido, projeto inexistente ou servidor indisponível) e pare.

## Resultado esperado
Um relatório curto em texto/markdown contendo: status do quality gate (`OK` ou `ERROR`), e, se reprovado, uma lista com métrica, valor atual, comparador e limiar para cada condição reprovada. Sem tabelas gigantes ou dados de métricas além do que a API retornou.

## Exemplos
1. Resposta `"projectStatus":{"status":"OK"}` → relatório: "Quality Gate: APROVADO (OK)."
2. Resposta com `"status":"ERROR"` e condition `"metricKey":"new_coverage","comparator":"LT","errorThreshold":"80","actualValue":"65"` → relatório apontando "Coverage: 80% exigidos no new code, obtido 65% — reprovação".

## Restrições
Não armazenar, imprimir ou propagar `$SONAR_TOKEN` em logs, comentários de código, commits ou arquivos. Não inventar endpoints ou métricas que não constam neste documento — quando o relatório exigir dados não cobertos aqui, sinalize a lacuna em vez de estimar. Não marcar o gate como aprovado por aproximação: use apenas o `status` retornado pela API.