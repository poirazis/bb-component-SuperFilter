<script>
  import { getContext } from "svelte"
  import { dataFilters } from "@budibase/shared-core"
  import ActionButton from "../node_modules/@budibase/bbui/src/ActionButton/ActionButton.svelte"
  import ActionGroup from "../node_modules/@budibase/bbui/src/ActionGroup/ActionGroup.svelte"
  import SuperCell  from "../bb-component-SuperTableCell/lib/SuperTableCell/SuperCell.svelte"

  export let dataProvider
  export let field
  export let filterType 
  export let showLabel = true
  export let label
  export let labelPos = "above"
  export let limit
  export let optionsSource
  export let options

  const { styleable, ActionTypes, getAction, API } = getContext("sdk")
  const component = getContext("component")

  let value = null 
  let reqLimit = limit

  let results
  let fieldSchema
  let tableSchema 
  let primaryDisplayField
  let loaded = false
  let schemaLoaded = false
  let cellState

  const loadTableSchema = async ( tableId ) => {
      tableSchema = await API.fetchTableDefinition(tableId);
      primaryDisplayField = tableSchema.primaryDisplay
      schemaLoaded = true
    }

  const loadTable = async ( tableId, newLimit ) => {
      results = await API.searchTable({
        paginate: false,
        tableId: tableId,
        limit: limit,
        query: { }
      });
      loaded = true;
    };

  const setFilter = ( newValue ) => {
    let filterObj = {
      field: isRelationship ? fieldSchema.foreignKey : field,
      operator: fieldSchema.type == "array" ? "contains" : "equal",
      value: fieldSchema.type == "number" ? Number (newValue) : newValue,
      valueType: "Value"
    }

    console.log(filterObj)

    const queryExtension = dataFilters.buildLuceneQuery([filterObj])
    addExtension?.($component.id, queryExtension)
    value = newValue
  }

  const clearFilter = ( ) => {
    removeExtension?.($component.id)
    value = null
  }

  $: fieldSchema = dataProvider?.schema[field] ?? {}
  $: isOptions = fieldSchema.type == "array" || fieldSchema.type == "options" 
  $: isRelationship = fieldSchema.type == "link"
  $: valid = isOptions || isRelationship || optionsSource == "custom"
  $: dataProviderId = dataProvider?.id
  $: if ( isRelationship && limit ) {
    if (!loaded) {
      loadTableSchema(fieldSchema.tableId)
      loadTable(fieldSchema.tableId, limit )
    } else if ( limit != reqLimit ) {
      loadTable(fieldSchema.tableId, limit )
      reqLimit = limit
    }
  }

  $: addExtension = getAction(
    dataProviderId,
    ActionTypes.AddDataProviderQueryExtension
  )

  $: removeExtension = getAction(
    dataProviderId,
    ActionTypes.RemoveDataProviderQueryExtension
  )

  $: $component.styles = {
    ...$component.styles,
    normal : {
      ...$component.styles.normal,
      "display" : "flex",
      "flex-direction" : labelPos == "above" ? "column" : "row",
      "justify-content" : "flex-start"
    }
  }

</script>

<div use:styleable={$component.styles}>
    {#if showLabel}
      <div class="fieldLabel" style:color={ value ? "lime" : "var(--spectrum-global-color-gray-800)"}>
        { label || field?.toUpperCase() }
      </div>
    {/if}

    {#if filterType == "input"}
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <div class="inputWrapper" on:click={cellState.focus}>
        <SuperCell
          bind:cellState 
          cellOptions={{ "padding" : "0.25rem"}}
          multi=false
          {value}
          {fieldSchema}
          editable = { true }
          on:change={ ( e ) => setFilter ( e.detail )}
        />
      </div>
    {/if}

    {#if filterType == "buttons"}
    {#key limit}
      {#if optionsSource == "schema" && isOptions}
        <ActionButton selected={ !value } on:click={ clearFilter }> ALL </ActionButton>
        {#each fieldSchema.constraints.inclusion as option, idx}
          {#if idx < limit}
            <ActionButton
              selected={option == value}
              emphasized
              on:click={ () => setFilter ( option ) }
              > 
              {option} 
            </ActionButton>
          {/if}
        {/each}
      {/if}
      
      {#if optionsSource == "schema" && isRelationship && loaded }
          <ActionButton selected={ !value } on:click={ clearFilter}> ALL </ActionButton>
          {#each results?.rows as option, idx}
            <ActionButton
              selected={option[fieldSchema.fieldName] == value}
              emphasized 
              on:click={ () => setFilter ( option[fieldSchema.fieldName] ) }
              > 
              {option[primaryDisplayField]} 
            </ActionButton>
          {/each}
      {/if}

      {#if optionsSource == "custom" && options }
        <ActionButton selected={ !value } on:click={ clearFilter}> ALL </ActionButton>
        {#each options as option, idx}
          <ActionButton
            selected={option.value == value}
            emphasized 
            on:click={ () => setFilter ( option.value ) }
            > 
            {option.label} 
          </ActionButton>
        {/each}
      {/if}
    {/key}
    {/if}
</div>

<style>
  .inputWrapper {
    flex: 1;
    min-width: 200px;
    min-height: 2rem;
    display: flex;
    justify-items: stretch;
    align-items: stretch;    
  }

  .fieldLabel {
    min-width: 5rem;
    display: flex;
    align-items: center;
    line-height: 32px;
  }
</style>