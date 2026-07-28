```
" ==============================================================================
" LOCAL TYPES 
" Place this section in the "Local Types" tab of your class in Eclipse/ADT
" ==============================================================================

" 1. Content Wrapper (Holds the Alias and Cardinality)
CLASS lcl_redirected_content DEFINITION.
  PUBLIC SECTION.
    INTERFACES if_xco_cds_composition_content.
    METHODS constructor IMPORTING alias_name  TYPE string
                                  cardinality TYPE if_xco_cds_association_content=>ts_cardinality.
  PRIVATE SECTION.
    DATA alias_string TYPE string.
    DATA comp_cardinality TYPE if_xco_cds_association_content=>ts_cardinality.
ENDCLASS.

CLASS lcl_redirected_content IMPLEMENTATION.
  METHOD constructor.
    alias_string = alias_name.
    comp_cardinality = cardinality.
  ENDMETHOD.
  
  METHOD if_xco_cds_composition_content~get_alias.
    rv_alias = alias_string.
  ENDMETHOD.
  
  METHOD if_xco_cds_composition_content~get_cardinality.
    rs_cardinality = comp_cardinality.
  ENDMETHOD.
ENDCLASS.


" 2. Composition Wrapper (Holds the Target and returns the Content)
CLASS lcl_redirected_composition DEFINITION.
  PUBLIC SECTION.
    INTERFACES if_xco_cds_composition.
    METHODS constructor IMPORTING target_name TYPE sxco_cds_object_name 
                                  alias_name  TYPE string
                                  cardinality TYPE if_xco_cds_association_content=>ts_cardinality.
  PRIVATE SECTION.
    DATA content_ref TYPE REF TO if_xco_cds_composition_content.
ENDCLASS.

CLASS lcl_redirected_composition IMPLEMENTATION.
  METHOD constructor.
    if_xco_cds_composition~target = target_name.
    content_ref = NEW lcl_redirected_content( alias_name  = alias_name 
                                              cardinality = cardinality ).
  ENDMETHOD.
  
  METHOD if_xco_cds_composition~content.
    ro_content = content_ref.
  ENDMETHOD.
ENDCLASS.


" ==============================================================================
" GLOBAL CLASS DEFINITION & IMPLEMENTATION
" ==============================================================================
CLASS zcl_vdm_diagram_xco_adp DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.
    INTERFACES zif_vdm_diagram_xco_adapter .

    ALIASES get_associations
      FOR zif_vdm_diagram_xco_adapter~get_associations .
    ALIASES get_cardinality
      FOR zif_vdm_diagram_xco_adapter~get_cardinality .
    ALIASES get_cds_name_by_ddl
      FOR zif_vdm_diagram_xco_adapter~get_cds_name_from_ddl .
    ALIASES get_cds_type
      FOR zif_vdm_diagram_xco_adapter~get_cds_type .
    ALIASES get_compositions
      FOR zif_vdm_diagram_xco_adapter~get_compositions .
    ALIASES get_fields
      FOR zif_vdm_diagram_xco_adapter~get_fields .
    ALIASES get_sources
      FOR zif_vdm_diagram_xco_adapter~get_sources .

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

    "! Parses the DDL source to find compositions missed by XCO and appends them
    "! @parameter cds_name     | The name of the projection view
    "! @parameter compositions | The XCO compositions table to be enriched
    METHODS enrich_redirected_compositions
      IMPORTING
        cds_name     TYPE sxco_cds_object_name
      CHANGING
        compositions TYPE sxco_t_cds_compositions.

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

        " Capture Group 1: The Alias (e.g., _Item)
        " Capture Group 2: The Target Entity (e.g., ZC_Item_View)
        DATA(pattern) = |(?i)\\b([a-zA-Z0-9_]+)\\s*:\\s*redirected\\s+to\\s+composition\\s+child\\s+([a-zA-Z0-9_]+)|.
        DATA(matches) = extract_regex_matches( pattern = pattern text = source_code ).

        LOOP AT matches INTO DATA(match).
          IF lines( match-submatches ) = 2.
            
            DATA(alias_match) = match-submatches[ 1 ].
            DATA(alias_name)  = substring( val = source_code off = alias_match-offset len = alias_match-length ).

            DATA(target_match) = match-submatches[ 2 ].
            DATA(target_name)  = to_upper( substring( val = source_code off = target_match-offset len = target_match-length ) ).

            " --- DUPLICATE PREVENTION ---
            DATA(is_duplicate) = abap_false.
            LOOP AT compositions INTO DATA(existing_comp).
              IF to_upper( existing_comp->content( )->get_alias( ) ) = to_upper( alias_name ).
                is_duplicate = abap_true.
                EXIT.
              ENDIF.
            ENDLOOP.

            " --- CARRIER INSTANTIATION & CARDINALITY LOOKUP ---
            IF is_duplicate = abap_false.
              " Default cardinality if lookup fails
              DATA(true_cardinality) = VALUE if_xco_cds_association_content=>ts_cardinality( min = 0 max = 2147483647 ). 
              
              " Drop down to the Base View to extract the original developer cardinality natively
              DATA(base_sources) = me->get_sources( cds_name ).
              IF lines( base_sources ) > 0.
                DATA(base_view) = base_sources[ 1 ].
                
                " Retrieve native compositions from the underlying TP view
                DATA(base_compositions) = me->get_compositions( base_view ).
                LOOP AT base_compositions INTO DATA(base_comp).
                  IF to_upper( base_comp->content( )->get_alias( ) ) = to_upper( alias_name ).
                    true_cardinality = base_comp->content( )->get_cardinality( ).
                    EXIT.
                  ENDIF.
                ENDLOOP.
              ENDIF.

              " Wrap target, alias, and resolved base-cardinality in the local carrier class
              DATA(redirected_composition) = NEW lcl_redirected_composition( 
                                             target_name = CONV #( target_name ) 
                                             alias_name  = alias_name
                                             cardinality = true_cardinality ).
                                             
              APPEND redirected_composition TO compositions.
            ENDIF.

          ENDIF.
        ENDLOOP.

      CATCH cx_root.
        " Silently catch framework errors and rely on the original XCO results
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


  METHOD zif_vdm_diagram_xco_adapter~get_associations.
    CASE me->get_cds_type( cds_name ).
      WHEN 'V'. associations = xco_cds=>view( cds_name )->associations->all->get( ).
      WHEN 'W'. associations = xco_cds=>view_entity( cds_name )->associations->all->get( ).
      WHEN 'P'. associations = xco_cds=>projection_view( cds_name )->associations->all->get( ).
      WHEN OTHERS. CLEAR associations.
    ENDCASE.
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


  METHOD zif_vdm_diagram_xco_adapter~get_compositions.
    " Standard XCO Extraction
    CASE me->get_cds_type( cds_name ).
      WHEN 'V'. compositions = xco_cds=>view( cds_name )->compositions->all->get( ).
      WHEN 'W'. compositions = xco_cds=>view_entity( cds_name )->compositions->all->get( ).
      WHEN 'P'. compositions = xco_cds=>projection_view( cds_name )->compositions->all->get( ).
      WHEN OTHERS. CLEAR compositions.
    ENDCASE.

    " Augment the table with missing DDL redirections
    enrich_redirected_compositions(
      EXPORTING cds_name     = cds_name
      CHANGING  compositions = compositions ).
  ENDMETHOD.

ENDCLASS.


"! <p class="shorttext synchronized">VDM Cytoscape JSON Renderer (XCO Framework)</p>
"! Generates a structured JSON payload for Cytoscape.js using the modern XCO library.
"! Inherits from the standard base class to utilize the text buffer and selection state.
CLASS zcl_vdm_diagram_cytoscape DEFINITION
  PUBLIC
  INHERITING FROM zcl_vdm_diagram_base
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.
    TYPES: BEGIN OF ty_format,
             layout_algorithm TYPE string,     
             theme            TYPE string,     
             animate          TYPE abap_bool,  
             node_spacing     TYPE i,          
           END OF ty_format.

    METHODS constructor IMPORTING format TYPE ty_format OPTIONAL.

    METHODS zif_vdm_diagram_hooks~on_start REDEFINITION.
    METHODS zif_vdm_diagram_hooks~on_end REDEFINITION.
    METHODS zif_vdm_diagram_hooks~on_entity_start REDEFINITION.
    METHODS zif_vdm_diagram_hooks~on_entity_end REDEFINITION.
    METHODS zif_vdm_diagram_hooks~on_base_elements REDEFINITION.
    METHODS zif_vdm_diagram_hooks~on_fields REDEFINITION.
    METHODS zif_vdm_diagram_hooks~on_associations REDEFINITION.
    METHODS zif_vdm_diagram_hooks~on_relationship REDEFINITION.
    METHODS zif_vdm_diagram_hooks~on_legend REDEFINITION.

    TYPES: BEGIN OF ty_node_data,
             id           TYPE string,
             label        TYPE string,
             is_focal     TYPE abap_bool,
             is_union     TYPE abap_bool,
             keys         TYPE string_table,
             standard     TYPE string_table,
             base_sources TYPE string_table,
             associations TYPE string_table,
           END OF ty_node_data.

    TYPES: BEGIN OF ty_node,
             data TYPE ty_node_data,
           END OF ty_node.

    TYPES: BEGIN OF ty_edge_data,
             id            TYPE string,
             source        TYPE string,
             target        TYPE string,
             label         TYPE string,
             cardinality   TYPE string,
             relation_type TYPE string,
             color_hint    TYPE string,
             source_anchor TYPE string,
           END OF ty_edge_data.

    TYPES: BEGIN OF ty_edge,
             data TYPE ty_edge_data,
           END OF ty_edge.

    TYPES: BEGIN OF ty_graph_config,
             format TYPE ty_format,
           END OF ty_graph_config.

    TYPES: BEGIN OF ty_elements,
             config TYPE ty_graph_config,
             nodes  TYPE STANDARD TABLE OF ty_node WITH EMPTY KEY,
             edges  TYPE STANDARD TABLE OF ty_edge WITH EMPTY KEY,
           END OF ty_elements.

  PRIVATE SECTION.
    DATA format       TYPE ty_format.
    DATA elements     TYPE ty_elements.
    DATA current_node TYPE ty_node.

    METHODS serialize_to_json RETURNING VALUE(result) TYPE string.

ENDCLASS.



CLASS zcl_vdm_diagram_cytoscape IMPLEMENTATION.

  METHOD constructor.
    super->constructor( ).

    IF format IS INITIAL.
      me->format-layout_algorithm = 'cose'.
      me->format-theme            = 'fiori_light'.
      me->format-animate          = abap_true.
      me->format-node_spacing     = 100.
    ELSE.
      me->format = format.
    ENDIF.

    CLEAR: me->elements, me->current_node.
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_start.
    CLEAR me->elements.
    me->elements-config-format = me->format.
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_entity_start.
    me->current_node = VALUE #(
      data = VALUE #(
        id       = alias
        label    = alias
        is_focal = is_focal_entity
      )
    ).
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_base_elements.
    me->current_node-data-is_union = is_union_entity.
    LOOP AT base_sources INTO DATA(source).
      APPEND source TO me->current_node-data-base_sources.
    ENDLOOP.
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_fields.
    IF selection-keys = abap_true.
      LOOP AT key_fields INTO DATA(key).
        APPEND key TO me->current_node-data-keys.
      ENDLOOP.
    ENDIF.

    IF standard_fields IS NOT INITIAL AND selection-fields = abap_true.
      LOOP AT standard_fields INTO DATA(field).
        APPEND field TO me->current_node-data-standard.
      ENDLOOP.
    ENDIF.
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_associations.
    LOOP AT association_aliases INTO DATA(assoc).
      APPEND assoc TO me->current_node-data-associations.
    ENDLOOP.
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_entity_end.
    APPEND me->current_node TO me->elements-nodes.
    CLEAR me->current_node.
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_relationship.
    DATA(color_hint) = SWITCH string( relationship_type
      WHEN zcl_vdm_diagram_generator=>c_relation_type-composition THEN '#800080' " Purple
      WHEN zcl_vdm_diagram_generator=>c_relation_type-association THEN '#188918' " Green
      ELSE '#32363a' ).

    DATA(source_anchor) = COND string(
      WHEN selection-associations_fields = abap_true AND association_alias IS NOT INITIAL
      THEN association_alias
      ELSE ''
    ).

    DATA(edge) = VALUE ty_edge(
      data = VALUE #(
        id            = |{ source_alias }_to_{ target_alias }_{ association_alias }|
        source        = source_alias
        target        = target_alias
        label         = association_alias
        cardinality   = cardinality_text
        relation_type = relationship_type
        color_hint    = color_hint
        source_anchor = source_anchor
      )
    ).

    APPEND edge TO me->elements-edges.
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_legend.
  ENDMETHOD.

  METHOD zif_vdm_diagram_hooks~on_end.
    DATA(json_output) = me->serialize_to_json( ).
    add_text( json_output ).
  ENDMETHOD.

  METHOD serialize_to_json.
    result = xco_cp_json=>data->from_abap( me->elements )->apply(
      VALUE #( ( xco_cp_json=>transformation->underscore_to_camel_case ) )
    )->to_string( ).
  ENDMETHOD.

ENDCLASS.
```
