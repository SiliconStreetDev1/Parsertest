```
METHOD zif_data_masker~apply_mask.
    " Evaluate the length of the incoming string to determine the masking strategy
    DATA(input_length) = strlen( unmasked_value ).

    " Rule 1: 16-character strings (nnnnnnnnnnnnnnnn -> nn-nnnn-XXXXXXXX-nn)
    IF input_length = 16.
      masked_value = |{ substring( val = unmasked_value off = 0 len = 2 ) }-{ substring( val = unmasked_value off = 2 len = 4 ) }-XXXXXXXX-{ substring( val = unmasked_value off = 14 len = 2 ) }|.
    
    " Rule 2: 13-character strings (nnnnnnnnaaann -> XXXXXXXX-aaa-nn)
    ELSEIF input_length = 13.
      masked_value = |XXXXXXXX-{ substring( val = unmasked_value off = 8 len = 3 ) }-{ substring( val = unmasked_value off = 11 len = 2 ) }|.
    
    " Fallback: If the string does not match the exact expected patterns, 
    " return the original string safely to prevent runtime substring dumps.
    ELSE.
      masked_value = unmasked_value.
    ENDIF.

  ENDMETHOD.
```
