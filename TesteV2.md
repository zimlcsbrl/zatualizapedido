REPORT z_update_po_cond.

*---------------------------------------------------------------------*
* Seleção
*---------------------------------------------------------------------*
PARAMETERS: p_file TYPE rlgrap-filename OBLIGATORY.

*---------------------------------------------------------------------*
* Tipos
*---------------------------------------------------------------------*
* Formato do CSV: EBELN;EBELP;VALOR;TIPO_COND
* TIPO_COND aceita: PB00 ou PBXX
*---------------------------------------------------------------------*
TYPES: BEGIN OF ty_input,
         ebeln     TYPE ebeln,
         ebelp     TYPE ebelp,
         valor     TYPE kbetr,
         cond_type TYPE kscha, "PB00 ou PBXX
       END OF ty_input.

TYPES: BEGIN OF ty_log,
         ebeln     TYPE ebeln,
         ebelp     TYPE ebelp,
         cond_type TYPE kscha,
         status    TYPE char10,
         message   TYPE string,
       END OF ty_log.

*---------------------------------------------------------------------*
* Dados
*---------------------------------------------------------------------*
DATA: gt_input TYPE STANDARD TABLE OF ty_input,
      gs_input TYPE ty_input,
      gt_log   TYPE STANDARD TABLE OF ty_log,
      gs_log   TYPE ty_log.

DATA: gt_file TYPE STANDARD TABLE OF string,
      gv_line TYPE string.

*---------------------------------------------------------------------*
* F4 ajuda arquivo
*---------------------------------------------------------------------*
AT SELECTION-SCREEN ON VALUE-REQUEST FOR p_file.
  CALL FUNCTION 'F4_FILENAME'
    IMPORTING
      file_name = p_file.

*---------------------------------------------------------------------*
* START-OF-SELECTION
*---------------------------------------------------------------------*
START-OF-SELECTION.

  PERFORM read_file.
  IF gt_input IS INITIAL.
    WRITE: / 'Arquivo vazio ou inválido.'.
    EXIT.
  ENDIF.

  PERFORM process_data.
  PERFORM display_log.

*---------------------------------------------------------------------*
* Leitura do CSV
*---------------------------------------------------------------------*
FORM read_file.

  DATA: lv_ebeln_raw  TYPE string,
        lv_ebelp_raw  TYPE string,
        lv_valor_raw  TYPE string,
        lv_ctype_raw  TYPE string.

  CALL FUNCTION 'GUI_UPLOAD'
    EXPORTING
      filename = p_file
      filetype = 'ASC'
    TABLES
      data_tab = gt_file
    EXCEPTIONS
      OTHERS   = 1.

  IF sy-subrc <> 0.
    MESSAGE 'Erro ao ler o arquivo. Verifique o caminho informado.' TYPE 'E'.
    EXIT.
  ENDIF.

  LOOP AT gt_file INTO gv_line.

    " pula header
    IF sy-tabix = 1.
      CONTINUE.
    ENDIF.

    CLEAR: gs_input, lv_ebeln_raw, lv_ebelp_raw, lv_valor_raw, lv_ctype_raw.

    SPLIT gv_line AT ';'
      INTO lv_ebeln_raw
           lv_ebelp_raw
           lv_valor_raw
           lv_ctype_raw.

    CONDENSE lv_ebeln_raw NO-GAPS.
    CONDENSE lv_ebelp_raw NO-GAPS.
    CONDENSE lv_valor_raw NO-GAPS.
    CONDENSE lv_ctype_raw NO-GAPS.

    " padding zeros à esquerda no número do pedido (CHAR/ALPHA)
    CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
      EXPORTING
        input  = lv_ebeln_raw
      IMPORTING
        output = gs_input-ebeln.

    " ebelp é NUMC5 — atribuição direta garante padding numérico
    gs_input-ebelp = lv_ebelp_raw.
    gs_input-valor = lv_valor_raw.

    " valida tipo de condição - Na unidas tem PB00 e PBXX
    IF lv_ctype_raw = 'PB00' OR lv_ctype_raw = 'PBXX'.
      gs_input-cond_type = lv_ctype_raw.
    ELSE.
      " tipo inválido — registra no log e ignora a linha
      gs_log-ebeln     = gs_input-ebeln.
      gs_log-ebelp     = gs_input-ebelp.
      gs_log-cond_type = lv_ctype_raw.
      gs_log-status    = 'ERRO'.
      gs_log-message   = |Tipo de condição inválido: { lv_ctype_raw }. Use PB00 ou PBXX.|.
      APPEND gs_log TO gt_log.
      CONTINUE.
    ENDIF.

    APPEND gs_input TO gt_input.

  ENDLOOP.

ENDFORM.

*---------------------------------------------------------------------*
* Processamento
*---------------------------------------------------------------------*
FORM process_data.

  DATA: lt_pocond  TYPE TABLE OF bapimepocond,
        lt_pocondx TYPE TABLE OF bapimepocondx,
        ls_pocond  TYPE bapimepocond,
        ls_pocondx TYPE bapimepocondx,
        lt_return  TYPE TABLE OF bapiret2,
        ls_return  TYPE bapiret2.

  DATA: lv_ebeln TYPE ebeln.

  " agrupa por pedido para chamar a BAPI uma vez por PO
  SORT gt_input BY ebeln.

  LOOP AT gt_input INTO gs_input.

    AT NEW ebeln.
      CLEAR: lt_pocond, lt_pocondx.
      lv_ebeln = gs_input-ebeln.
    ENDAT.

    " --- POCOND ---
    CLEAR ls_pocond.
    ls_pocond-po_item    = gs_input-ebelp.
    ls_pocond-cond_type  = gs_input-cond_type.
    ls_pocond-cond_value = gs_input-valor.
    APPEND ls_pocond TO lt_pocond.

    " --- POCONDX ---
    CLEAR ls_pocondx.
    ls_pocondx-po_item    = gs_input-ebelp.
    ls_pocondx-cond_type  = gs_input-cond_type.
    ls_pocondx-change_id  = 'U'.   " U = Update — necessário para a BAPI aplicar a alteração
    ls_pocondx-cond_value = 'X'.
    APPEND ls_pocondx TO lt_pocondx.

    AT END OF ebeln.

      CLEAR lt_return.

      CALL FUNCTION 'BAPI_PO_CHANGE'
        EXPORTING
          purchaseorder = lv_ebeln
        TABLES
          pocond        = lt_pocond
          pocondx       = lt_pocondx
          return        = lt_return.

      READ TABLE lt_return INTO ls_return WITH KEY type = 'E'.

      IF sy-subrc = 0.
        " erro — loga cada item do pedido
        LOOP AT lt_pocond INTO ls_pocond.
          CLEAR gs_log.
          gs_log-ebeln     = lv_ebeln.
          gs_log-ebelp     = ls_pocond-po_item.
          gs_log-cond_type = ls_pocond-cond_type.
          gs_log-status    = 'ERRO'.
          gs_log-message   = ls_return-message.
          APPEND gs_log TO gt_log.
        ENDLOOP.

        " rollback explícito em caso de erro
        CALL FUNCTION 'BAPI_TRANSACTION_ROLLBACK'.

      ELSE.
        CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'
          EXPORTING
            wait = 'X'.

        LOOP AT lt_pocond INTO ls_pocond.
          CLEAR gs_log.
          gs_log-ebeln     = lv_ebeln.
          gs_log-ebelp     = ls_pocond-po_item.
          gs_log-cond_type = ls_pocond-cond_type.
          gs_log-status    = 'OK'.
          gs_log-message   = 'Atualizado com sucesso'.
          APPEND gs_log TO gt_log.
        ENDLOOP.

      ENDIF.

    ENDAT.

  ENDLOOP.

ENDFORM.

*---------------------------------------------------------------------*
* Log
*---------------------------------------------------------------------*
FORM display_log.

  DATA: lt_fieldcat TYPE slis_t_fieldcat_alv,
        ls_fieldcat TYPE slis_fieldcat_alv.

  DEFINE add_field.
    CLEAR ls_fieldcat.
    ls_fieldcat-fieldname = &1.
    ls_fieldcat-seltext_m = &2.
    APPEND ls_fieldcat TO lt_fieldcat.
  END-OF-DEFINITION.

  add_field 'EBELN'     'Pedido'.
  add_field 'EBELP'     'Item'.
  add_field 'COND_TYPE' 'Condição'.
  add_field 'STATUS'    'Status'.
  add_field 'MESSAGE'   'Mensagem'.

  CALL FUNCTION 'REUSE_ALV_GRID_DISPLAY'
    EXPORTING
      it_fieldcat = lt_fieldcat
    TABLES
      t_outtab    = gt_log.

ENDFORM.
