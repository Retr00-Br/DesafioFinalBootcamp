# 📝 Resposta do Laboratório: A Wiki Perdida dos Arquivos Corporativos

Proposta de arquitetura serverless na AWS para transformação de acervo documental heterogêneo em uma Wiki Corporativa Inteligente com suporte a RAG e citações de fontes.

---

## 👤 Identificação

* **Nome:** Pedro Barbosa de Andrade
* **Data:** 11/10/2026
* **Link do repositório:** https://github.com/pedro-andrade/laboratorio-wiki-aws

---

## ✅ Quest 1: O Mapa dos Arquivos Perdidos

### 1.1 Formatos encontrados na pasta `raw/`

* **`ata_reuniao_vendas_sa.pdf`:** PDF Digital nativo (5 páginas). Não necessita de OCR tradicional; permite a extração direta da camada de texto, tabelas de pauta e blocos estruturados em JSON no final do arquivo.
* **`ata_resultados_vendas_novos_dados.png`:** Imagem / Folha digitalizada (1 página). Necessita obrigatoriamente de OCR com suporte a visão computacional e leitura manuscrita (Amazon Textract), contendo texto impresso, anotações à mão (*"conferir CRM"*, *"Acao prioritaria"*) e carimbos (*"DOCUMENTO FICTICIO"*).
* **`vendas_sa_dados_ficticios_laboratorio.csv`:** Arquivo Tabular Estruturado (CRM com 19 colunas e centenas de linhas). Não necessita de OCR. Exige parsing de dados relacionais e vetorização baseada em registros de oportunidades.

### 1.2 Principais desafios encontrados

* **Inconsistência de Tipos de Mídia no Mesmo Diretório:** Todos os arquivos residem soltos na pasta `raw/`, exigindo roteamento dinâmico sem depender de organização por subpastas.
* **Presença de Elementos Não Estruturados e Manuscritos:** A imagem PNG contém anotações manuais e marcas d'água que OCRs tradicionais baseados em padrões rígidos falham em interpretar.
* **Formatos Tabulares x Textuais:** O CSV traz dados quantitativos e colunares de vendas, enquanto as atas trazem decisões e narrativas. A arquitetura precisa normalizar esses dois universos para que a busca em linguagem natural funcione com ambos.

### 1.3 Informações importantes a serem extraídas

* **Atas (PDF/PNG):** Data da reunião, participantes, metas de faturamento, indicadores comerciais (ticket médio, conversão, churn), decisões aprovadas (IDs D-001 a D-005), plano de ação com responsáveis e prazos, além de anotações marginais.
* **CRM (CSV):** IDs de oportunidade, contas-alvo, representantes de vendas, valores negociados, etapas do funil, regiões e motivos de perda.

### 1.4 Estratégia de classificação inicial

A classificação será feita de forma automatizada por um manipulador de eventos na ingestão:
1. **Extensão do arquivo:** Roteamento inicial via sufixo (`.csv` -> Pipeline de Dados, `.png` / `.jpg` -> Pipeline OCR Textract, `.pdf` -> Inspeção de Camada de Texto).
2. **Inspeção de MIME Type e Metadados:** Validação do cabeçalho do arquivo via código Python no AWS Lambda para determinar se o PDF contém texto nativo ou se é um PDF de imagem escaneada.

---

## ✅ Quest 2: O Portal de Entrada na AWS

### 2.1 Armazenamento dos arquivos brutos

* **Amazon S3 (Bucket Raw):** Criação de um bucket dedicado `s3://wiki-corporativa-raw-storage/`.
* **AWS IAM:** Políticas de mínimo privilégio restringindo escrita apenas a papéis autorizados.
* **AWS KMS:** Criptografia em repouso (*SSE-KMS*) para proteção contra acesso não autorizado aos dados corporativos.
* **Amazon S3 Versioning:** Habilitado para manter histórico e evitar perda por sobrescrita involuntária.

### 2.2 Preservação dos arquivos originais

* A pasta/bucket `raw/` atua como uma zona de dados imutável (*Landing Zone*).
* Atribuição do **S3 Object Lock** em modo *Compliance* ou *Governance* para impedir a exclusão ou alteração de documentos originais.
* Todas as transformações, extrações de OCR e textos limpos são gravados exclusivamente em buckets secundários (`s3://wiki-corporativa-processed/`).

### 2.3 Extração de texto dos documentos

* **PDFs Digitais (`.pdf` nativo):** Processados via AWS Lambda utilizando bibliotecas como `pypdf` ou `pdfplumber` para extração direta e rápida da camada de texto sem custo de OCR.
* **Imagens (`.png`, `.jpg`) e PDFs Escaneados:** Encaminhados automaticamente para o **Amazon Textract** usando a API `AnalyzeDocument` (recursos `TABLES` e `FORMS`), garantindo o mapeamento de tabelas e anotações manuscritas.
* **CSVs e Tabulares (`.csv`):** Processados via script Python no Lambda (usando `pandas`) ou job do **AWS Glue** para converter cada linha/registro em um trecho de texto enriquecido com contexto colunar.
* **Orquestração e Monitoramento:** O **AWS Step Functions** orquestra o fluxo de processamento de acordo com o tipo de arquivo, emitindo métricas e logs no **Amazon CloudWatch**.

### 2.4 Tratamento de falhas

* **Dead Letter Queues (DLQ) no Amazon SQS:** Arquivos que falharem no processamento (como imagens corrompidas ou CSVs fora do padrão) são redirecionados para uma fila DLQ.
* **Alertas no CloudWatch:** Alarmes configurados para notificar a equipe técnica via SNS caso a taxa de falhas na extração ultrapasse os limites aceitáveis.

---

## ✅ Quest 3: A Relíquia dos Metadados

### 3.1 Padronização dos textos processados

Os textos extraídos de qualquer formato passam por uma etapa de limpeza e normalização em Python (AWS Lambda):
* Remoção de caracteres nulos e quebras de linha indesejadas.
* Conversão de acentuação e padronização da codificação para `UTF-8`.
* Estruturação do texto em arquivos no formato `JSON` padronizado.

### 3.2 Metadados propostos

| Metadado | Por que ele é importante? |
| :--- | :--- |
| **Nome do documento** | Garante a rastreabilidade direta do arquivo de origem na citação da Wiki. |
| **Tipo do documento** | Identifica se a fonte é uma Ata de Reunião, Relatório de CRM ou Documento Escaneado. |
| **Data identificada** | Permite ordenação cronológica e filtros temporais nas pesquisas. |
| **Tema principal** | Categorização do conteúdo (ex: Vendas, Resultados 2S, Inadimplência). |
| **Participantes / Autores** | Identifica pessoas associadas ao documento ou reunião. |
| **Decisões tomadas** | Mapeia os IDs das decisões aprovadas para consulta rápida de liderança. |
| **Responsáveis** | Vincula ações registradas a papéis ou nomes específicos. |
| **Próximos passos** | Rastreia pendências e prazos futuros. |
| **Nível de confidencialidade**| Aplica governança de dados e restrição de acesso por perfil. |
| **Caminho do arquivo original**| Link URI S3 (`s3://...`) para download do arquivo bruto original. |

### 3.3 Uso de IA para enriquecimento dos documentos

Através do **Amazon Bedrock**, utilizando chamadas a modelos de linguagem (como Claude 3 / Titan Text) via `boto3`:
* Leitura do texto bruto extraído e geração de resumos executivos automáticos.
* Extração de Entidades Nomeadas (NER): Identificação automática de decisões, nomes, prazos e métricas financeiras.
* Classificação taxonômica automática e criação das tags de metadados em formato JSON.

### 3.4 Armazenamento dos metadados

* **Amazon DynamoDB:** Armazenamento chave-valor para busca rápida dos metadados indexados pelo ID do documento.
* **Amazon Bedrock Knowledge Bases:** Integração direta dos metadados junto com o texto indexado no repositório vetorial, permitindo filtragem por metadados durante a busca semântica.

---

## ✅ Quest 4: O Oráculo da Wiki Inteligente

### 4.1 Estratégia de indexação

* **Chunking Estruturado:** Divisão dos documentos em trechos (*chunks*) de 500 a 1000 tokens com *overlap* (sobreposição) de 10% a 15% para manter a continuidade do contexto entre blocos.
* Inclusão do cabeçalho do documento e nome da fonte em cada *chunk* individual.

### 4.2 Busca semântica e base vetorial

* **Geração de Embeddings:** Processamento dos *chunks* através dos modelos **Amazon Titan Text Embeddings** no Amazon Bedrock.
* **Base Vetorial:** Armazenamento dos vetores no **Amazon OpenSearch Serverless** ou na infraestrutura nativa do **Amazon Bedrock Knowledge Bases**, permitindo busca por similaridade de cosseno.

### 4.3 Geração de respostas com IA

1. **Recepção:** A pergunta em linguagem natural do usuário é enviada para a API do Bedrock.
2. **Retrieval (Recuperação):** O sistema converte a pergunta em vetor e busca os *chunks* mais relevantes no OpenSearch Serverless.
3. **Augmentation & Generation (Geração Aumentada):** O LLM no Bedrock recebe a pergunta acompanhada estritamente dos trechos recuperados como contexto.
4. **Citação de Fontes:** O prompt instrui o modelo a responder citando explicitamente o arquivo e a página de onde a informação foi extraída (ex: *"Conforme a ata de reunião `ata_reuniao_vendas_sa.pdf` (Página 3)..."*).

### 4.4 Interface de consulta

* **Amazon Q Business** ou Web App Serverless construída com **AWS Amplify** + **API Gateway** + **AWS Lambda**, autenticada via **Amazon Cognito** para gestão de permissões de usuários.

### 4.5 Segurança, auditoria e monitoramento

* **AWS IAM & Cognito:** Controle de acesso baseado em funções (RBAC) e autenticação de usuários.
* **AWS KMS:** Criptografia de dados em repouso e em trânsito (TLS 1.3).
* **AWS CloudTrail:** Auditoria completa de todas as chamadas de API feitas na conta AWS.
* **Amazon Macie:** Varredura preventiva para identificar dados sensíveis (PII) nos arquivos brutos.
* **AWS Cost Explorer & CloudWatch:** Dashboard para acompanhamento diário de custos dos modelos Bedrock e chamadas do Textract.

---

## 🧩 Arquitetura Final da Solução

### 1. Visão geral

Arquitetura Serverless e acoplada por eventos (*Event-Driven Architecture*) que automatiza a ingestão de múltiplos formatos de arquivos em S3, executa extração contextual (Texto Nativo, Textract OCR ou Parsing CSV), enriquece dados via Amazon Bedrock e os disponibiliza em uma interface RAG com citações precisas das fontes.

### 2. Serviços AWS utilizados

| Serviço AWS | Papel na solução |
| :--- | :--- |
| **Amazon S3** | Armazenamento imutável da pasta `raw/` e repositório de arquivos processados. |
| **Amazon Textract** | OCR inteligente para imagens (`.png`) e identificação de tabelas/manuscritos. |
| **AWS Lambda** | Processamento sem servidor, execução de roteamento e parsing de CSV/PDF. |
| **AWS Step Functions** | Orquestração do pipeline de processamento do arquivo bruto até a indexação. |
| **Amazon Bedrock** | Extração de metadados, geração de embeddings e síntese de respostas em linguagem natural. |
| **Amazon Bedrock Knowledge Bases** | Gerenciamento integrado do pipeline de RAG e integração com a base vetorial. |
| **Amazon OpenSearch Serverless** | Armazenamento dos vetores e execução da busca semântica. |
| **Amazon DynamoDB** | Repositório de metadados estruturados e controle de estado dos arquivos. |
| **AWS IAM & KMS** | Segurança, controle de acesso e criptografia das chaves corporativas. |
| **Amazon CloudWatch** | Monitoramento de métricas, erros e logs do pipeline. |

### 3. Fluxo de dados de ponta a ponta

1. Arquivos são carregados no bucket de entrada `s3://wiki-corporativa-raw-storage/`.
2. Evento S3 aciona o AWS Step Functions para orquestrar o pipeline.
3. Roteamento inteligente identifica o tipo de arquivo:
   * `.pdf` (digital) -> AWS Lambda extrai texto nativo.
   * `.png` (imagem) -> Amazon Textract executa OCR para texto e manuscrito.
   * `.csv` (CRM) -> AWS Lambda/Glue realiza o parsing colunar.
4. O texto extraído é padronizado e limpo pelo AWS Lambda.
5. Amazon Bedrock analisa o texto, gerando resumos e extraindo metadados.
6. Os metadados são salvos no Amazon DynamoDB.
7. O texto padronizado é fragmentado (*chunking*) e convertido em vetores via Titan Embeddings.
8. Os vetores são indexados no Amazon OpenSearch Serverless via Bedrock Knowledge Bases.
9. O usuário envia uma pergunta na interface web.
10. A solução recupera os *chunks* relevantes e o Bedrock gera a resposta citando o documento original.

### 4. Diagrama textual da arquitetura

[Arquivos raw/ (PDF/PNG/CSV)] 
       │
       ▼
[Amazon S3 (Bucket Raw)] ──(Event)──► [AWS Step Functions]
                                              │
                      ┌───────────────────────┼───────────────────────┐
                      ▼                       ▼                       ▼
             [Lambda (PDF Native)]   [Textract (PNG/OCR)]    [Lambda/Glue (CSV)]
                      │                       │                       │
                      └───────────────────────┼───────────────────────┘
                                              ▼
                                 [Lambda (Normalização)]
                                              │
                      ┌───────────────────────┴───────────────────────┐
                      ▼                                               ▼
          [Amazon Bedrock (Metadata)]                     [Amazon Bedrock (Embeddings)]
                      │                                               │
                      ▼                                               ▼
             [Amazon DynamoDB]                         [OpenSearch Serverless]
                                                                      │
                                                                      ▼
 [Usuário] ◄── [Amplify / API Gateway] ◄── [Bedrock RAG Engine] ◄─────┘
