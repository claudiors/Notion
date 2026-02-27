# Documentação do Workflow: GRID - Revisão Gramatical de PDFs - V1

## Visão Geral
Este workflow automatiza a revisão gramatical e ortográfica de apresentações em PDF armazenadas no Google Drive. Ele utiliza a inteligência artificial da Anthropic (modelo Claude Sonnet) para analisar o conteúdo, identifica erros slide a slide e gera um relatório detalhado em uma base de dados do Notion. Além disso, implementa um sistema de fila via Redis para evitar processamento paralelo conflitante.

## Arquitetura do Sistema
O fluxo segue as seguintes etapas lógicas:
1. **Gatilho**: Monitoramento de pasta específica no Google Drive.
2. **Controle de Concorrência**: Fila baseada em Mutex (Redis) para garantir que apenas um arquivo seja processado por vez.
3. **Extração e Análise**: Conversão do PDF para Base64 e envio para a API da Anthropic com um prompt especializado em revisão gramatical brasileira.
4. **Processamento de Dados**: Limpeza e normalização da resposta JSON recebida da IA.
5. **Relatórios no Notion**: Criação de registro na base de dados e formatação de uma página interna com callouts detalhando cada erro encontrado.

---

## Detalhamento dos Nodes

### 1. Gatilho (Google Drive Trigger)
- **Função**: Monitora uma pasta específica no Google Drive para novos arquivos criados.
- **Configuração**: Polling a cada 1 minuto.
- **Evento**: `fileCreated`.

### 2. Gestão de Fila (Redis Mutex)
- **CheckLock**: Verifica se a chave `lock_grid_revisao` existe no Redis.
- **Lock Ativo? (If)**: Se a chave existir (lock ativo), o fluxo aguarda 30 segundos e verifica novamente.
- **Lock**: Se a chave não existir, cria a chave `lock_grid_revisao` com valor "ativo" e TTL de 300 segundos, reservando a execução.
- **UnLock**: Após a análise da IA, exclui a chave para liberar a fila.

### 3. Processamento de PDF (Code & HTTP)
- **Download PDF**: Baixa o binário do arquivo detectado.
- **Extrair Base64**: Código JavaScript que converte o buffer do PDF para String Base64 e monta o payload para a Anthropic.
- **Revisão Gramatical (HTTP Request)**: Chamada POST para a API da Anthropic (`v1/messages`). Utiliza o modelo Claude com instruções para retornar um JSON estruturado com os slides e seus respectivos erros.

### 4. Normalização (Parsear Resposta)
- **Função**: Código JavaScript para remover blocos de código markdown da resposta da IA (` ```json `) e realizar o `JSON.parse`. Garante que os campos de resumo e slides estejam presentes mesmo em caso de falhas leves na resposta.

### 5. Registro de Resultados (Notion)
- **Criar Registro**: Cria uma nova página na base de dados "Revisor textual" com propriedades como Nome do Arquivo, Data, Total de Ajustes e Link do Drive.
- **Formatar Página**: Gera um array de blocos do Notion (H1, Dividers, Callouts) para o corpo da página.
- **Update Page (HTTP Request)**: Insere os blocos formatados no corpo da página recém-criada via API do Notion (PATCH children).

### 6. Lógica de Status (IF)
- **Tem erros?**: Verifica se a contagem de erros é maior que zero.
  - **Sim**: Define o status da página no Notion como **"Ajustar"**.
  - **Não**: Define o status como **"Aprovado"**.

---

## Configuração e Pré-requisitos

### Credenciais Necessárias
- **Google Drive OAuth2**: Para monitoramento e download de arquivos.
- **Anthropic API Key**: Para processamento de linguagem natural (Claude).
- **Redis (Cache & Queue)**: Para controle de fila.
- **Notion Integation Token**: Para criação e atualização de registros.

### Variáveis e IDs (Redigidos)
- **ID da Pasta GDrive**: `[REDACTED_DRIVE_FOLDER_ID]`
- **ID da Base Notion**: `[REDACTED_NOTION_DB_ID]`
- **Chave Redis**: `lock_grid_revisao`

---

## Manual de Uso (Sticky Notes)
1. Faça o upload do material para a pasta designada no Google Drive.
2. Evite subir mais de 2 arquivos simultaneamente para manter a eficiência da fila.
3. Aguarde cerca de 1 minuto para o gatilho e o processamento da IA.
4. Confira o resultado detalhado no Notion na base "Revisor textual".

---
*Gerado automaticamente para documentação técnica no Github.*
