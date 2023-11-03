<script>
  import { getContext } from "svelte"
  import { dataFilters } from "@budibase/shared-core"
  import ActionButton from "../node_modules/@budibase/bbui/src/ActionButton/ActionButton.svelte"
  import ActionGroup from "../node_modules/@budibase/bbui/src/ActionGroup/ActionGroup.svelte"

  export let dataProvider
  export let field
  export let mode
  export let limit

  const { styleable, ActionTypes, getAction, API } = getContext("sdk")
  const component = getContext("component")

  let value
  let filters = []

  let results
  let fieldSchema
  let tableSchema 
  let primaryDisplay

  const loadTableSchema = async ( tableId ) => {
        tableSchema = await API.fetchTableDefinition(tableId);
        primaryDisplay = tableSchema.primaryDisplay
    }

  const loadTable = async ( tableId, limit ) => {
      results = await API.searchTable({
        paginate: false,
        tableId: tableId,
        limit: limit,
        query: { }
      });
    };

  $: fieldSchema = dataProvider?.schema[field] ?? {}
  $: isOptions = fieldSchema.type == "array" || fieldSchema.type == "options"
  $: isRelationship = fieldSchema.type == "link"
  $: valid = isOptions || isRelationship   
  $: dataProviderId = dataProvider?.id
  $: if ( isRelationship && limit ) {
    loadTableSchema(fieldSchema.tableId)
    loadTable(fieldSchema.tableId, limit )
  }

  $: addExtension = getAction(
    dataProviderId,
    ActionTypes.AddDataProviderQueryExtension
  )

  $: removeExtension = getAction(
    dataProviderId,
    ActionTypes.RemoveDataProviderQueryExtension
  )

  $: filterObj = {
            field: isOptions ? field : fieldSchema.foreignKey,
            operator: fieldSchema.type == "array" ? "contains" :  "equal",
            value: value,
            valueType: "Value",
          }

  $: {
    if ( value ) {
      const queryExtension = dataFilters.buildLuceneQuery([filterObj])
      addExtension?.($component.id, queryExtension)
    } else {
      removeExtension?.($component.id)
    }
  }
</script>

<div use:styleable={$component.styles}>

  {#if !valid}
    <p> Unsupported Column Type </p>
  {/if}

  {#if isOptions}
    <ActionGroup>
      <ActionButton selected={ !value } on:click={ () => { value = null } }> ALL </ActionButton>
      {#each fieldSchema.constraints.inclusion as option}
        <ActionButton
          selected={option == value}
          emphasized
          on:click={ () => { value = option } }
          > 
          {option} 
        </ActionButton>
      {/each}
    </ActionGroup>
  {/if}
  
  {#if isRelationship && results?.rows }
    <ActionGroup>
      <ActionButton selected={ !value } on:click={ () => { value = null } }> ALL </ActionButton>
      {#each results?.rows as option}
        <ActionButton
          selected={option[fieldSchema.fieldName] == value}
          emphasized 
          on:click={ () => { value = option[fieldSchema.fieldName] } }
          > 
          {option[primaryDisplay]} 
        </ActionButton>
      {/each}
      </ActionGroup>
  {/if}

</div>