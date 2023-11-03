<script>
  import { getContext } from "svelte"
  import { dataFilters } from "@budibase/shared-core"
  import ActionButton from "../node_modules/@budibase/bbui/src/ActionButton/ActionButton.svelte"
  import ActionGroup from "../node_modules/@budibase/bbui/src/ActionGroup/ActionGroup.svelte"

  export let dataProvider
  export let field
  export let limit

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
      field: isOptions ? field : fieldSchema.foreignKey,
      operator: fieldSchema.type == "array" ? "contains" :  "equal",
      value: newValue,
      valueType: "Value"
    }

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
  $: valid = isOptions || isRelationship   
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
</script>

<div use:styleable={$component.styles}>
  {#key limit}

    {#if !valid}
      <p> Unsupported Column Type </p>
    {/if}

    {#if isOptions}
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
    
    {#if isRelationship && loaded }
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

  {/key}
</div>