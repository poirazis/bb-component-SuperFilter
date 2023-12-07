<script>
  import { getContext } from "svelte";
  import { dataFilters } from "@budibase/shared-core";
  import ActionButton from "../node_modules/@budibase/bbui/src/ActionButton/ActionButton.svelte";
	import ActionGroup from "../node_modules/@budibase/bbui/src/ActionGroup/ActionGroup.svelte"
  import SuperCell from "../bb_super_components_shared/src/lib/SuperCell/SuperCell.svelte"

	const { styleable, ActionTypes, getAction, API } = getContext("sdk");
  const component = getContext("component");

  export let dataProvider;
  export let field;
  export let filterType
  export let showLabel = true;
  export let label;
  export let labelPos = "above";
	export let labelWidth = "8rem"
  export let limit = 10
  export let optionsSource
  export let customOptions = []

  let value = null;
  let reqLimit = limit;
	let options 

  let results;
  let fieldSchema;
  let tableSchema;
  let primaryDisplayField;
  let loaded = false;
  let schemaLoaded = false;
  let cellState;

  const getOperator = (type) => {
    if (type == "string") return "fuzzy";
    if (type == "array") return "contains";
    if (type == "options") return "equal";
    if (type == "link") return "equal";
  };

  const loadTableSchema = async (tableId) => {
    tableSchema = await API.fetchTableDefinition(tableId);
    primaryDisplayField = tableSchema.primaryDisplay;
    schemaLoaded = true;
  };

  const loadTable = async (tableId, newLimit) => {
    results = await API.searchTable({
      paginate: false,
      tableId: tableId,
      limit: limit,
      query: {},
    });
	  options = results.rows.map( (r) => { return { label: r[tableSchema.primaryDisplay], value: fieldSchema.foreignKey ?? r["_id"]} })
    loaded = true;
  };

  const setFilter = (newValue) => {
    if (!newValue || newValue == "") {
      console.log("Clearing");
      clearFilter();
      return;
    }

    let filterObj = {
      field: fieldSchema.type == "link" ? fieldSchema.foreignKey : field,
      operator: getOperator(fieldSchema.type),
      value: fieldSchema.type == "options" ? newValue[0] : newValue,
      valueType: "Value",
    };

    const queryExtension = dataFilters.buildLuceneQuery([filterObj]);
    addExtension?.($component.id, queryExtension);
    value = newValue;
  };

  const clearFilter = () => {
    removeExtension?.($component.id);
    value = null;
  };

	const initializeOptions = () => {
		options = []
		if ( filterType == "input" ) return;

		if ( optionsSource == "custom" ) 
			options = customOptions
		else if ( ["array", "options", "link"].includes(fieldSchema?.type) ) {
			if ( fieldSchema.type == "link" ) {
				loadTableSchema( fieldSchema.tableId)
				loadTable(fieldSchema.tableId, limit)
			} else {
				options = fieldSchema?.constraints?.inclusion.map( ( x ) => { return { label:x, value: x } })
      }
		}	
	}

  $: fieldSchema = dataProvider?.schema[field] ?? {};
  $: dataProviderId = dataProvider?.id;

  $: addExtension = getAction(
    dataProviderId,
    ActionTypes.AddDataProviderQueryExtension
  );

  $: removeExtension = getAction(
    dataProviderId,
    ActionTypes.RemoveDataProviderQueryExtension
  );

	$: initializeOptions( filterType, field, optionsSource, limit )

  $: $component.styles = {
    ...$component.styles,
    normal: {
      ...$component.styles.normal,
			"min-height" : "2rem",
      "display": "flex",
      "flex-direction": labelPos == "above" && showLabel ? "column" : "row",
      "align-items": labelPos == "above" && showLabel ? "stretch" : "center",
      "gap": labelPos == "above" ? "0.25rem" : "0.85rem",
      "--label-width": labelPos == "left" ? labelWidth ?? "8rem" : "auto",
			"min-width" : 0,
    },
  };

  $: console.log(fieldSchema)
</script>

<div use:styleable={$component.styles}>
  {#if showLabel}
    <div class="fieldLabel">
      {label || field.toUpperCase() }
    </div>
  {/if}

  {#if filterType == "input" }
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <div class="inputWrapper" on:click={cellState.focus}>
      <SuperCell
        bind:cellState
        cellOptions={{
          padding: "0.5rem",
          clearValueIcon: true,
          iconFront: "ri-search-line",
          placeholder: "Search " + field,
          debounce: 500,
        }}
        multi={false}
        initialState={"Editing"}
        lockState
        {value}
        {fieldSchema}
        editable={true}
        on:change={(e) => setFilter(e.detail)}
      />
    </div>
  {/if}
	

  {#if filterType == "options"}
		<ActionGroup compact quiet>
			<ActionButton size="S" quiet selected={!value} on:click={clearFilter}>
				<svg xmlns="http://www.w3.org/2000/svg" 
					width="14" height="14" viewBox="0 0 24 24" 
					fill="none" stroke={ value ? "var(--spectrum-global-color-blue-500)" : "var(--spectrum-global-color-gray-500)"} stroke-width="2" stroke-linecap="round" 
					stroke-linejoin="round" class="lucide lucide-circle-off">
					<path d="m2 2 20 20"/><path d="M8.35 2.69A10 10 0 0 1 21.3 15.65"/><path d="M19.08 19.08A10 10 0 1 1 4.92 4.92"/>
				</svg>			
			</ActionButton>
			{#if options?.length}
				{#each options as option, idx}
					{#if idx <= limit}
						<ActionButton
							selected={option.value == value}
							quiet
							fullwidth
							size="S"
							on:click={() => setFilter(option.value)}
						>
              {option.label}
						</ActionButton>
					{/if}
				{/each}
			{/if}
		</ActionGroup>
  {/if}

</div>

<style>
  .inputWrapper {
    min-width: 180px;
    height: 2rem;
    width: 100%;
    display: flex;
    justify-items: stretch;
    align-items: stretch;
    gap: 0.85rem;
  }
  .fieldLabel {
		width: var(--label-width);
    font-size: 14px;
    align-items: center;
    line-height: 1.75rem;
    font-weight: 500;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
  }
</style>
