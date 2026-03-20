REQUERIMENTO DE NOVO DESENVOLVIMENTO (RND)

1. Informações Gerais

Título: UNIDAS - Atualização em Massa de Condições de Preço (PB00 / PBXX) em Pedidos de Compra

Solicitante:
Área de Suprimentos

Sistema:
SAP S/4HANA

Módulo:
MM – Compras

Tipo de Demanda:
Novo Desenvolvimento

Prioridade:
Alta

2. Contextualização

2.1 Origem do Problema

Existe uma inconsistência entre os valores aprovados no sistema Locavia e os pedidos de compra criados no SAP. A raiz do problema está no fato de que o Locavia permite a aprovação de pedidos com quantidades fracionadas, enquanto o SAP aceita apenas quantidades inteiras (absolutas). Ao receber o pedido, o SAP arredonda a quantidade automaticamente, gerando uma divergência nos valores finais entre os dois sistemas.

Essa inconsistência se manifesta nas condições de preço dos itens, que podem estar registradas como PB00 (preço bruto com registro de condição) ou PBXX (preço bruto sem registro de condição), a depender da origem do pedido.

2.2 Motivação e Escopo da Solução

Este desenvolvimento tem caráter de mitigação de backlog: a solução visa corrigir em massa os pedidos já existentes que apresentam divergência de valores. A correção do processo de integração entre Locavia e SAP para evitar novas ocorrências será tratada em iniciativa separada, fora do escopo deste desenvolvimento.

Além disso, a atualização manual dos valores — realizada item a item durante o lançamento do documento financeiro (MIRO) — é:

Operacionalmente custosa

Sujeita a erros manuais

Inviável para tratamento em volume

Tentativas de automação via LSMW foram descartadas devido a limitações técnicas do ambiente, que força o uso de IDocs para o objeto BUS2012 e inviabiliza o uso de BAPI nesse contexto.

Dessa forma, faz-se necessário o desenvolvimento de uma solução customizada.

3. Solução Proposta

Desenvolvimento de um programa ABAP para atualização em massa das condições de preço PB00 e PBXX em itens de pedidos de compra, utilizando a função padrão BAPI_PO_CHANGE.

3.1 Funcionamento

O programa deverá:

Receber um arquivo CSV contendo:

Pedido (EBELN)

Item (EBELP)

Novo valor da condição

Tipo de condição (PB00 ou PBXX)

Realizar leitura, validação e tratamento dos dados

Rejeitar no log linhas com tipo de condição inválido (diferente de PB00 e PBXX)

Agrupar os registros por pedido

Para cada pedido:

Montar as estruturas POCOND e POCONDX da BAPI

Atualizar a condição de preço dos itens informados

Executar commit das alterações (ou rollback explícito em caso de erro)

Gerar log detalhado de processamento

3.2 Estrutura do Arquivo de Entrada

O arquivo CSV deve conter cabeçalho e utilizar ponto e vírgula (;) como separador:

EBELN;EBELP;VALOR;TIPO_COND
4509439884;30;32;PBXX
4509466604;40;54.54;PB00

Campos:

EBELN   – Número do pedido de compra
EBELP   – Item do pedido
VALOR   – Novo valor da condição de preço
TIPO_COND – Tipo de condição: PB00 ou PBXX

4. Detalhes Técnicos

4.1 Funções Utilizadas

BAPI_PO_CHANGE         – Atualização do pedido de compra
BAPI_TRANSACTION_COMMIT  – Confirmação da transação (com WAIT = 'X')
BAPI_TRANSACTION_ROLLBACK – Rollback explícito em caso de erro
GUI_UPLOAD             – Leitura do arquivo CSV da máquina local
F4_FILENAME            – Ajuda de pesquisa para seleção do arquivo
CONVERSION_EXIT_ALPHA_INPUT – Padding com zeros à esquerda no número do pedido (EBELN)

4.2 Estruturas da BAPI

POCOND

  PO_ITEM    – Item do pedido
  COND_TYPE  – Tipo de condição (PB00 ou PBXX, conforme arquivo)
  COND_VALUE – Novo valor informado no arquivo

POCONDX (flags de atualização)

  PO_ITEM    – Item do pedido
  COND_TYPE  – Tipo de condição (PB00 ou PBXX, conforme arquivo)
  CHANGE_ID  – 'U' (Update) — obrigatório para que a BAPI aplique a alteração
  COND_VALUE – 'X'

4.3 Lógica de Processamento

Leitura do CSV com verificação de erro (sy-subrc após GUI_UPLOAD)

Validação do TIPO_COND: linhas com valor diferente de PB00 ou PBXX são rejeitadas e registradas no log com status ERRO, sem interromper o processamento das demais

Padding automático do número do pedido (EBELN) via CONVERSION_EXIT_ALPHA_INPUT

Campo EBELP (NUMC5) recebe padding numérico por atribuição direta ao tipo

Processamento agrupado por pedido (AT NEW ebeln / AT END OF ebeln)

Execução da BAPI uma vez por pedido, acumulando todos os itens do grupo

Em caso de erro retornado pela BAPI: ROLLBACK e log de erro para todos os itens do pedido

Em caso de sucesso: COMMIT com WAIT = 'X' e log de sucesso

4.4 Saída

Exibição em ALV contendo:

Pedido (EBELN)

Item (EBELP)

Tipo de Condição atualizada (COND_TYPE)

Status (OK / ERRO)

Mensagem retornada

5. Regras de Negócio

A condição (PB00 ou PBXX) deve existir previamente no pedido — o programa realiza atualização, não inclusão

Apenas itens explicitamente informados no arquivo serão alterados

O tipo de condição deve ser informado por linha; valores diferentes de PB00 e PBXX são rejeitados

Pedidos devem estar em status editável no SAP

Alterações de preço podem impactar a estratégia de liberação do pedido

6. Riscos e Impactos

Risco	Impacto	Mitigação
Condição inexistente no item	Falha na atualização	Log detalhado com mensagem da BAPI
Reset de liberação do pedido	Impacto no processo de aprovação	Alinhamento prévio com o negócio
Tipo de condição incorreto no arquivo	Linha rejeitada silenciosamente	Validação explícita com registro no log
Dados inconsistentes no CSV	Erros de processamento	Validação e limpeza dos campos na leitura

7. Roteiro de Testes

Cenário 1 – Sucesso com PBXX

Pedido com condição PBXX existente

Tipo informado no arquivo: PBXX

Atualização realizada corretamente

Cenário 2 – Sucesso com PB00

Pedido com condição PB00 existente

Tipo informado no arquivo: PB00

Atualização realizada corretamente

Cenário 3 – Erro: condição inexistente

Pedido sem a condição informada

Registro de erro no log; demais itens processados normalmente

Cenário 4 – Múltiplos itens no mesmo pedido

Itens com tipos de condição distintos (PB00 e PBXX) no mesmo pedido

Todos processados em uma única chamada da BAPI

Cenário 5 – Tipo de condição inválido no arquivo

Linha com TIPO_COND diferente de PB00/PBXX

Linha rejeitada, log registra o erro, processamento continua

Cenário 6 – Pedido inexistente

Número de pedido não encontrado no SAP

Erro retornado pela BAPI, registrado no log

8. Exclusão de Escopo

Criação de pedidos de compra

Inclusão de novas condições de preço em itens

Atualização de condições diferentes de PB00 e PBXX

Correção do processo de integração Locavia ↔ SAP (tratado em iniciativa separada)

Integração com outros sistemas

9. Documentação Gerada

Código fonte ABAP (z_update_po_cond)

Layout do arquivo de entrada (CSV com 4 colunas)

Log de execução por ALV

10. Métrica de Complexidade

Complexidade: Baixa a Média

Justificativa:

Uso de BAPI padrão (BAPI_PO_CHANGE)

Lógica simples de agrupamento e processamento sequencial

Sem integrações externas

Suporte a dois tipos de condição (PB00 / PBXX) com validação explícita
