```
CLASS zcl_data_masking DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.
    "! Masks SAP FCY Call account numbers (Format: XXXXXXXX-XAD-30)
    "! @parameter iv_account | Raw account string
    "! @parameter rv_masked  | Masked account string
    METHODS mask_sap_fcy_call
      IMPORTING iv_account       TYPE string
      RETURNING VALUE(rv_masked) TYPE string.

    "! Masks IBAN account numbers
    "! @parameter iv_account | Raw IBAN string
    "! @parameter rv_masked  | Masked IBAN string
    METHODS mask_iban
      IMPORTING iv_account       TYPE string
      RETURNING VALUE(rv_masked) TYPE string.

    "! Masks 15-character Host account numbers (Format: XX-XXXX-XXXXX45-00)
    "! @parameter iv_account | Raw Host account string
    "! @parameter rv_masked  | Masked Host account string
    METHODS mask_host_account_15
      IMPORTING iv_account       TYPE string
      RETURNING VALUE(rv_masked) TYPE string.

    "! Masks 16-character Host account numbers (Format: XX-XXXX-XXXXXX5-000)
    "! @parameter iv_account | Raw Host account string
    "! @parameter rv_masked  | Masked Host account string
    METHODS mask_host_account_16
      IMPORTING iv_account       TYPE string
      RETURNING VALUE(rv_masked) TYPE string.

  PROTECTED SECTION.
  PRIVATE SECTION.
    "! General Fallback Rule: All but the final 4 characters
    "! @parameter iv_account | Raw account string
    "! @parameter rv_masked  | Masked account string
    METHODS mask_fallback
      IMPORTING iv_account       TYPE string
      RETURNING VALUE(rv_masked) TYPE string.
ENDCLASS.

CLASS zcl_data_masking IMPLEMENTATION.

  METHOD mask_sap_fcy_call.
    " Target Format: XXXXXXXX-XAD-30 
    " Requirement leaves the last 4 clear, distributed around hyphens.
    DATA(account_len) = strlen( iv_account ).
    
    IF account_len = 13.
      " Offsets capture the specific clear characters required by the layout
      rv_masked = |XXXXXXXX-X{ iv_account+9(2) }-{ iv_account+11(2) }|.
    ELSE.
      " Scenario Fallback: Apply general rule if length deviates
      rv_masked = mask_fallback( iv_account = iv_account ).
    ENDIF.
  ENDMETHOD.

  METHOD mask_iban.
    " Target Requirement: X in pos 3,4 and pos 9 to the final 3 characters.
    rv_masked = iv_account.
    DATA(account_len) = strlen( rv_masked ).

    IF account_len >= 12.
      " Mask pos 3 and 4 (Offset 2)
      rv_masked+2(2) = 'XX'.

      " Mask from pos 9 (Offset 8) leaving the final 3 clear
      DATA(mask_len) = account_len - 11. 
      
      IF mask_len > 0.
        rv_masked+8(mask_len) = repeat( val = 'X' occ = mask_len ).
      ENDIF.
    ELSE.
      " Scenario Fallback
      rv_masked = mask_fallback( iv_account = iv_account ).
    ENDIF.
  ENDMETHOD.

  METHOD mask_host_account_15.
    " Target Format: XX-XXXX-XXXXX45-00
    DATA(account_len) = strlen( iv_account ).
    
    IF account_len = 15.
      " 11 Xs total, leaving the last 4 characters clear and distributed
      rv_masked = |XX-XXXX-XXXXX{ iv_account+11(2) }-{ iv_account+13(2) }|.
    ELSE.
      " Scenario Fallback
      rv_masked = mask_fallback( iv_account = iv_account ).
    ENDIF.
  ENDMETHOD.

  METHOD mask_host_account_16.
    " Target Format: XX-XXXX-XXXXXX5-000
    DATA(account_len) = strlen( iv_account ).
    
    IF account_len = 16.
      " 12 Xs total, leaving the last 4 characters clear and distributed
      rv_masked = |XX-XXXX-XXXXXX{ iv_account+12(1) }-{ iv_account+13(3) }|.
    ELSE.
      " Scenario Fallback
      rv_masked = mask_fallback( iv_account = iv_account ).
    ENDIF.
  ENDMETHOD.

  METHOD mask_fallback.
    " Core IS Requirement: Substitute 'X' for all but the final 4 characters.
    " Applies when specific layout formatting fails due to anomalous data.
    rv_masked = iv_account.
    DATA(account_len) = strlen( rv_masked ).
    
    IF account_len > 4.
      DATA(mask_len) = account_len - 4.
      rv_masked(mask_len) = repeat( val = 'X' occ = mask_len ).
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```
