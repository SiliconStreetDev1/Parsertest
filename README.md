```
DATA(pattern) = `(?i)\b([a-zA-Z0-9_]+)\s*:\s*redirected\s+to\s+(?:composition\s+parent\s+|parent\s+|[a-zA-Z0-9_])`.
CLASS zcl_vdm_diagram_xco_adp DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.
    INTERFACES zif_vdm_diagram_xco_adapter .

    ALIASES get_associations FOR zif_vdm_diagram_xco_adapter~get_associations .
    ALIASES get_cardinality  FOR zif_vdm_diagram_xco_adapter~get_cardinality .
    ALIASES get_cds_name_by_ddl FOR zif_vdm_diagram_xco_adapter~get_cds_name_from_ddl .
    ALIASES get_cds_type     FOR zif_vdm_diagram_xco_adapter~get_cds_type .
    ALIASES get_compositions FOR zif_vdm_diagram_xco_adapter~get_compositions .
    ALIASES get_fields       FOR zif_vdm_diagram_xco_adapter~get_fields .
    ALIASES get_sources      FOR zif_vdm_diagram_xco_adapter~get_sources .

  PROTECTED SECTION.
    "! Reads the raw DDL source code from the database using standard DDIC APIs
    METHODS get_ddl_source
      IMPORTING
        cds_name           TYPE sxco_cds_object_name
      RETURNING
        VALUE(source_code) TYPE string.

    "! Executes a cached PCRE regular expression against a target string
    METHODS extract_regex_matches
      IMPORTING
        pattern        TYPE string
        text           TYPE string
      RETURNING
        VALUE(matches) TYPE match_result_tab.

    "! Parses DDL to find missed redirected compositions and appends native XCO handles
    METHODS enrich_redirected_compositions
      IMPORTING
        cds_name     TYPE sxco_cds_object_name
      CHANGING
        compositions TYPE sxco_t_cds_compositions.

    "! Parses DDL to find missed redirected associations/parents and appends native XCO handles
    METHODS enrich_redirected_associations
      IMPORTING
        cds_name     TYPE sxco_cds_object_name
      CHANGING
        associations TYPE sxco_t_cds_associations.

  PRIVATE SECTION.
    TYPES: BEGIN OF ddl_cache_entry,
             name   TYPE sxco_cds_object_name,
             source TYPE string,
           END OF ddl_cache_entry.

    TYPES: BEGIN OF ty_regex_cache,
             pattern TYPE string,
             regex   TYPE REF TO cl_abap_regex,
           END OF ty_regex_cache.

    TYPES: BEGIN OF ty_type_cache,
             cds_name TYPE sxco_cds_object_name,
             type     TYPE zvdm_diagram_cds_type,
           END OF ty_type_cache.

    DATA ddl_cache   TYPE HASHED TABLE OF ddl_cache_entry WITH UNIQUE KEY name.
    DATA regex_cache TYPE HASHED TABLE OF ty_regex_cache WITH UNIQUE KEY pattern.
    DATA type_cache  TYPE HASHED TABLE OF ty_type_cache WITH UNIQUE KEY cds_name.
ENDCLASS.

CLASS zcl_vdm_diagram_xco_adp IMPLEMENTATION.

  METHOD extract_regex_matches.
    DATA regex_object TYPE REF TO cl_abap_regex.

    ASSIGN regex_cache[ pattern = pattern ] TO FIELD-SYMBOL(<cached_regex>).
    IF sy-subrc = 0.
      regex_object = <cached_regex>-regex.
    ELSE.
      TRY.
          regex_object = cl_abap_regex=>create_pcre( pattern = pattern ).
          INSERT VALUE #( pattern = pattern regex = regex_object ) INTO TABLE regex_cache.
        CATCH cx_sy_regex.
          CLEAR matches.
          RETURN.
      ENDTRY.
    ENDIF.

    TRY.
        DATA(matcher) = regex_object->create_matcher( text = text ).
        matches       = matcher->find_all( ).
      CATCH cx_sy_matcher.
        CLEAR matches.
    ENDTRY.
  ENDMETHOD.


  METHOD get_ddl_source.
    DATA(normalized_name) = to_upper( cds_name ).

    ASSIGN ddl_cache[ name = normalized_name ] TO FIELD-SYMBOL(<cache_entry>).
    IF sy-subrc = 0.
      source_code = <cache_entry>-source.
      RETURN.
    ENDIF.

    TRY.
        cl_dd_ddl_handler_factory=>create( )->read( EXPORTING name         = CONV ddlname( normalized_name )
                                                    IMPORTING ddddlsrcv_wa = DATA(source_wa) ).
        source_code = source_wa-source.
      CATCH cx_dd_ddl_read.
        source_code = ''.
    ENDTRY.

    INSERT VALUE #( name   = normalized_name
                    source = source_code ) INTO TABLE ddl_cache.
  ENDMETHOD.


  METHOD enrich_redirected_compositions.
    TRY.
        DATA(source_code) = get_ddl_source( cds_name ).
        IF source_code IS INITIAL.
          RETURN.
        ENDIF.

        " Find aliases explicitly redirected to composition child
        DATA(pattern) = |(?i)\\b([a-zA-Z0-9_]+)\\s*:\\s*redirected\\s+to\\s+composition\\s+child|.
        DATA(matches) = extract_regex_matches( pattern = pattern text = source_code ).

        LOOP AT matches INTO DATA(match).
          IF lines( match-submatches ) >= 1.
            DATA(alias_match) = match-submatches[ 1 ].
            DATA(alias_name)  = substring( val = source_code off = alias_match-offset len = alias_match-length ).

            DATA(is_duplicate) = abap_false.
            LOOP AT compositions INTO DATA(existing_comp).
              IF to_upper( existing_comp->content( )->get_alias( ) ) = to_upper( alias_name ).
                is_duplicate = abap_true.
                EXIT.
              ENDIF.
            ENDLOOP.

            " Explicitly request the projection view handle to enforce native projection targeting (No Ghost Lines)
            IF is_duplicate = abap_false.
              IF me->get_cds_type( cds_name ) = 'P'. 
                APPEND xco_cds=>projection_view( cds_name )->composition( CONV #( alias_name ) ) TO compositions.
              ENDIF.
            ENDIF.
          ENDIF.
        ENDLOOP.
      CATCH cx_root.
        RETURN.
    ENDTRY.
  ENDMETHOD.


  METHOD enrich_redirected_associations.
    TRY.
        DATA(source_code) = get_ddl_source( cds_name ).
        IF source_code IS INITIAL.
          RETURN.
        ENDIF.

        " Find aliases redirected to parent or composition parent or standard redirected
        DATA(pattern) = |(?i)\\b([a-zA-Z0-9_]+)\\s*:\\s*redirected\\s+to\\s+(?:composition\\s+parent\\s+|parent\\s+|[a-zA-Z0-9_])|.
        DATA(matches) = extract_regex_matches( pattern = pattern text = source_code ).

        LOOP AT matches INTO DATA(match).
          IF lines( match-submatches ) >= 1.
            DATA(alias_match) = match-submatches[ 1 ].
            DATA(alias_name)  = substring( val = source_code off = alias_match-offset len = alias_match-length ).

            DATA(is_duplicate) = abap_false.
            LOOP AT associations INTO DATA(existing_assoc).
              IF to_upper( existing_assoc->content( )->get_alias( ) ) = to_upper( alias_name ).
                is_duplicate = abap_true.
                EXIT.
              ENDIF.
            ENDLOOP.

            " Explicitly request the projection view handle to enforce native projection targeting
            IF is_duplicate = abap_false.
              IF me->get_cds_type( cds_name ) = 'P'. 
                APPEND xco_cds=>projection_view( cds_name )->association( CONV #( alias_name ) ) TO associations.
              ENDIF.
            ENDIF.
          ENDIF.
        ENDLOOP.
      CATCH cx_root.
        RETURN.
    ENDTRY.
  ENDMETHOD.


  METHOD zif_vdm_diagram_xco_adapter~get_cds_name_from_ddl.
    IF cds_name IS INITIAL.
      ddlcds_name = cds_name.
      RETURN.
    ENDIF.

    DATA(source_code) = get_ddl_source( cds_name ).
    IF source_code IS INITIAL.
      ddlcds_name = cds_name.
      RETURN.
    ENDIF.

    DATA(keyword) = SWITCH string( me->get_cds_type( cds_name )
      WHEN 'W' THEN `view entity`
      WHEN 'V' THEN `define view`
      WHEN 'P' THEN `projection view`
      ELSE `` ).

    IF keyword IS NOT INITIAL.
      DATA(lower_source) = to_lower( source_code ).
      DATA(text_after) = substring_after( val = lower_source sub = keyword ).

      IF text_after IS NOT INITIAL.
        DATA(offset) = find( val = lower_source sub = keyword ) + strlen( keyword ).
        DATA(original_text) = substring( val = source_code off = offset ).

        DATA(found_name) = segment( val = condense( original_text ) index = 1 sep = ` ` ).
        found_name = replace( val = found_name sub = `;` with = `` ).

        FIND FIRST OCCURRENCE OF found_name IN source_code IGNORING CASE MATCH OFFSET DATA(match_offset).
        IF sy-subrc = 0.
          ddlcds_name = substring( val = source_code off = match_offset len = strlen( cds_name ) ).
        ENDIF.
      ENDIF.
    ENDIF.

    IF ddlcds_name IS INITIAL OR to_upper( ddlcds_name ) <> to_upper( cds_name ).
      ddlcds_name = cds_name.
    ENDIF.
  ENDMETHOD.


  METHOD zif_vdm_diagram_xco_adapter~get_cds_type.
    DATA(normalized_cds_name) = CONV sxco_cds_object_name( to_upper( cds_name ) ).

    ASSIGN type_cache[ cds_name = normalized_cds_name ] TO FIELD-SYMBOL(<cache_entry>).
    IF sy-subrc = 0.
      type = <cache_entry>-type.
      RETURN.
    ENDIF.

    DATA(object_name_filter) = xco_abap_repository=>object_name->get_filter(
                                 xco_abap_sql=>constraint->equal( normalized_cds_name ) ).

    DATA(type_definitions) = xco_abap_repository=>objects->ddls->where(
                               VALUE #( ( object_name_filter ) ) )->in( xco_abap=>repository )->get( ).

    IF lines( type_definitions ) = 1.
      type = type_definitions[ 1 ]->get_type( )->value.
    ENDIF.

    INSERT VALUE #( cds_name = normalized_cds_name type = type ) INTO TABLE type_cache.
  ENDMETHOD.


  METHOD zif_vdm_diagram_xco_adapter~get_sources.
    TRY.
        CASE me->get_cds_type( cds_name ).
          WHEN 'V'. APPEND xco_cds=>view( cds_name )->content( )->get_data_source( )-entity TO sources.
          WHEN 'W'. APPEND xco_cds=>view_entity( cds_name )->content( )->get_data_source( )-view_entity TO sources.
          WHEN 'P'. APPEND xco_cds=>projection_view( cds_name )->content( )->get_data_source( )-view_entity TO sources.
          WHEN OTHERS. CLEAR sources.
        ENDCASE.
      CATCH cx_xco_runtime_exception.
        DATA(source_code) = get_ddl_source( cds_name ).
        IF source_code IS NOT INITIAL.
          DATA(regex_matches) = extract_regex_matches( pattern = `(?i)\b(?:from|join)\s+([a-zA-Z0-9_/]+)` text = source_code ).
          sources = VALUE #( FOR match IN regex_matches WHERE ( submatches IS NOT INITIAL )
                             LET match_sub = match-submatches[ 1 ] IN
                             ( substring( val = source_code off = match_sub-offset len = match_sub-length ) ) ).
        ENDIF.
    ENDTRY.
  ENDMETHOD.


  METHOD zif_vdm_diagram_xco_adapter~search_for_cds.
    IF cds_search_string IS INITIAL.
      RETURN.
    ENDIF.

    DATA(search_pattern) = to_upper( cds_search_string ).
    search_pattern = replace( val = search_pattern sub = '*' with = '%' occ = 0 ).

    DATA(name_filter) = xco_abap_repository=>object_name->get_filter(
      xco_abap_sql=>constraint->contains_pattern( search_pattern )
    ).

    DATA(ddl_objects) = xco_abap_repository=>objects->ddls->where(
      VALUE #( ( name_filter ) )
    )->in( xco_abap=>repository )->get( ).

    cds_names = VALUE #( FOR ddl_object IN ddl_objects ( ddl_object->name ) ).
  ENDMETHOD.


  METHOD zif_vdm_diagram_xco_adapter~get_fields.
    fields = xco_cds=>entity( cds_name )->fields->all->get( ).
  ENDMETHOD.


  METHOD zif_vdm_diagram_xco_adapter~get_cardinality.
    cardinality = currentcardinality.

    IF hasparent = abap_true. 
      cardinality-min = 1.
      cardinality-max = 1.
      RETURN.
    ENDIF.

    IF NOT ( cardinality-max = 1 AND cardinality-min = 0 ).
      RETURN.
    ENDIF.

    DATA(source) = get_ddl_source( cds_name ).
    IF source IS INITIAL OR assocname IS INITIAL.
      RETURN.
    ENDIF.

    FIND FIRST OCCURRENCE OF assocname IN source IGNORING CASE MATCH OFFSET DATA(name_off).
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

    DATA(prefix) = substring( val = source len = name_off ).
    DATA(start_off) = find( val = to_upper( prefix ) sub = 'ASSOCIATION' occ = -1 ).

    IF start_off < 0.
      RETURN.
    ENDIF.

    DATA(line) = substring( val = source off = start_off len = name_off - start_off + strlen( assocname ) ).
    DATA(esc_name) = replace( val = assocname sub = '/' with = '\/' occ = 0 ).
    DATA(pattern) = `(?i)association\s*(?:\[([^\]]*)\])?[^;]*?\bas\s+` && esc_name && `\b`.

    DATA(matches) = extract_regex_matches( pattern = pattern text = line ).

    IF lines( matches ) > 0.
      DATA(match) = matches[ 1 ].

      IF lines( match-submatches ) > 0 AND match-submatches[ 1 ]-length > 0.
        DATA(sub_match) = match-submatches[ 1 ].
        DATA(val) = condense( val = substring( val = line off = sub_match-offset len = sub_match-length ) from = ` ` to = `` ).

        IF val = '1'.
          cardinality-min = 1.
          cardinality-max = 1.
        ENDIF.
      ENDIF.
    ENDIF.
  ENDMETHOD.


  METHOD zif_vdm_diagram_xco_adapter~get_associations.
    CASE me->get_cds_type( cds_name ).
      WHEN 'V'. associations = xco_cds=>view( cds_name )->associations->all->get( ).
      WHEN 'W'. associations = xco_cds=>view_entity( cds_name )->associations->all->get( ).
      WHEN 'P'. associations = xco_cds=>projection_view( cds_name )->associations->all->get( ).
      WHEN OTHERS. CLEAR associations.
    ENDCASE.

    enrich_redirected_associations(
      EXPORTING cds_name     = cds_name
      CHANGING  associations = associations ).
  ENDMETHOD.


  METHOD zif_vdm_diagram_xco_adapter~get_compositions.
    CASE me->get_cds_type( cds_name ).
      WHEN 'V'. compositions = xco_cds=>view( cds_name )->compositions->all->get( ).
      WHEN 'W'. compositions = xco_cds=>view_entity( cds_name )->compositions->all->get( ).
      WHEN 'P'. compositions = xco_cds=>projection_view( cds_name )->compositions->all->get( ).
      WHEN OTHERS. CLEAR compositions.
    ENDCASE.

    enrich_redirected_compositions(
      EXPORTING cds_name     = cds_name
      CHANGING  compositions = compositions ).
  ENDMETHOD.

ENDCLASS.
```
