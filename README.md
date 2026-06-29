# Parsertest

DATA adjusted_date TYPE d.
    DATA adjusted_time TYPE t.

    " Retrieve current precise UTC timestamp and adjust to the configured time zone
    GET TIME STAMP FIELD DATA(current_timestamp).
    CONVERT TIME STAMP current_timestamp TIME ZONE configured_timezone
            INTO DATE adjusted_date TIME adjusted_time.

    " Define a dynamic mapping structure for format tokens
    TYPES: BEGIN OF ty_mapping,
             token TYPE string,
             value TYPE string,
           END OF ty_mapping.

    " Populate the mapping table inline with extracted date/time components
    DATA(mappings) = VALUE STANDARD TABLE OF ty_mapping(
      ( token = 'YYYY' value = CONV string( adjusted_date(4) ) )
      ( token = 'MM'   value = CONV string( adjusted_date+4(2) ) )
      ( token = 'DD'   value = CONV string( adjusted_date+6(2) ) )
      ( token = 'hh'   value = CONV string( adjusted_time(2) ) )
      ( token = 'mm'   value = CONV string( adjusted_time+2(2) ) )
      ( token = 'ss'   value = CONV string( adjusted_time+4(2) ) )
    ).

    " Functionally fold the string replacements using the REDUCE operator
    formatted_output = REDUCE string(
      INIT result = format_string
      FOR mapping IN mappings
      NEXT result = replace( val = result sub = mapping-token with = mapping-value occ = 0 )
    ).
