<script>
  import { getContext } from "svelte";
  import { dataFilters } from "@budibase/shared-core";
  import ActionButton from "../node_modules/@budibase/bbui/src/ActionButton/ActionButton.svelte";
  import ActionGroup from "../node_modules/@budibase/bbui/src/ActionGroup/ActionGroup.svelte";
  import SuperCell from "../bb-component-SuperTableCell/lib/SuperTableCell/SuperCell.svelte";

  export let dataProvider;
  export let field;
  export let filterType = "input";
  export let showLabel = true;
  export let label;
  export let labelPos = "above";
  export let limit;
  export let optionsSource = "schema";
  export let options;

  const { styleable, ActionTypes, getAction, API } = getContext("sdk");
  const component = getContext("component");

  let value = null;
  let reqLimit = limit;

  let results;
  let fieldSchema;
  let tableSchema;
  let primaryDisplayField;
  let loaded = false;
  let schemaLoaded = false;
  let cellState;

  const getOperator = ( type ) => {
    if ( type == "string") return "fuzzy"
    if ( type == "array") return "contains"
    if ( type == "options") return "equal"
    if ( type == "link") return "equal"
  }

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
    loaded = true;
  };

  const setFilter = (newValue) => {
    if ( !newValue || newValue == "" ) {
      console.log("Clearing")
      clearFilter();
      return
    }

    let filterObj = {
      field: isRelationship ? fieldSchema.foreignKey : field,
      operator: getOperator (fieldSchema.type),
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

  $: fieldSchema = dataProvider?.schema[field] ?? {};
  $: isOptions = fieldSchema.type == "array" || fieldSchema.type == "options";
  $: isRelationship = fieldSchema.type == "link";
  $: valid = isOptions || isRelationship || optionsSource == "custom";
  $: dataProviderId = dataProvider?.id;
  $: if (isRelationship && limit) {
    if (!loaded) {
      loadTableSchema(fieldSchema.tableId);
      loadTable(fieldSchema.tableId, limit);
    } else if (limit != reqLimit) {
      loadTable(fieldSchema.tableId, limit);
      reqLimit = limit;
    }
  }

  $: addExtension = getAction(
    dataProviderId,
    ActionTypes.AddDataProviderQueryExtension
  );

  $: removeExtension = getAction(
    dataProviderId,
    ActionTypes.RemoveDataProviderQueryExtension
  );

  $: $component.styles = {
    ...$component.styles,
    normal: {
      ...$component.styles.normal,
      display: "flex",
      "flex-direction": labelPos == "above" ? "column" : "row",
      "justify-content": "flex-start",
      gap: "0.5rem",
      "--label-max-width": labelPos == "left" ? "8rem" : "auto",
    },
  };
</script>

<div use:styleable={$component.styles}>
  {#if showLabel}
    <div
      class="fieldLabel"
      style:color={value ? "#378ef0" : "var(--spectrum-global-color-gray-600)"}
      style:font-weight={value ? 600 : 500 }
    >
      <div class="icon" style:color={value ? "#378ef0" : "var(--spectrum-global-color-gray-500)"}>
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="1.25"
          stroke-linecap="round"
          stroke-linejoin="round"
          class="lucide lucide-search"
          ><circle cx="11" cy="11" r="8" /><path d="m21 21-4.3-4.3" /></svg
        >
      </div>
      {label?.toUpperCase() || field?.toUpperCase()}
    </div>
  {/if}

  {#if filterType == "input"}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <div class="inputWrapper" on:click={cellState.focus}>
      {#if !showLabel}
        <div
          class="icon"
          style:color={value
            ? "#378ef0"
            : "var(--spectrum-global-color-gray-500)"}
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="20"
            height="20"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="1.25"
            stroke-linecap="round"
            stroke-linejoin="round"
            class="lucide lucide-search"
            ><circle cx="11" cy="11" r="8" /><path d="m21 21-4.3-4.3" /></svg
          >
        </div>
      {/if}

      <SuperCell
        bind:cellState
        cellOptions={{ padding: "0.5rem" }}
        multi={false}
        {value}
        {fieldSchema}
        editable={true}
        on:change={(e) => setFilter(e.detail)}
      />

      {#if value}
      <div
        on:click={clearFilter}
        class="icon"
        style:color={value
          ? "var(--spectrum-global-color-red-500)"
          : "var(--spectrum-global-color-gray-700)"}
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="1.25"
          stroke-linecap="round"
          stroke-linejoin="round"
          class="lucide lucide-x"
          ><path d="M18 6 6 18" /><path d="m6 6 12 12" /></svg
        >
      </div>
      {/if}
    </div>
  {/if}

  {#if filterType == "buttons"}
    {#key limit}
      <ActionGroup>
        {#if optionsSource == "schema" && isOptions}
          <ActionButton selected={!value} on:click={clearFilter}>
            ALL
          </ActionButton>
          {#each fieldSchema.constraints.inclusion as option, idx}
            {#if idx < limit}
              <ActionButton
                selected={option == value}
                emphasized
                on:click={() => setFilter(option)}
              >
                {option}
              </ActionButton>
            {/if}
          {/each}
        {/if}

        {#if optionsSource == "schema" && isRelationship && loaded}
          <ActionButton selected={!value} on:click={clearFilter}>
            ALL
          </ActionButton>
          {#each results?.rows as option, idx}
            <ActionButton
              selected={option[fieldSchema.fieldName] == value}
              emphasized
              on:click={() => setFilter(option[fieldSchema.fieldName])}
            >
              {option[primaryDisplayField]}
            </ActionButton>
          {/each}
        {/if}

        {#if optionsSource == "custom" && options}
          <ActionButton selected={!value} on:click={clearFilter}>
            ALL
          </ActionButton>
          {#each options as option, idx}
            <ActionButton
              selected={option.value == value}
              emphasized
              on:click={() => setFilter(option.value)}
            >
              {option.label}
            </ActionButton>
          {/each}
        {/if}
      </ActionGroup>
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
    gap: 0.5rem;
  }

  .icon {
    display: flex;
    align-items: center;
  }

  .fieldLabel {
    min-width: 5rem;
    max-width: var(--label-max-width);
    display: flex;
    font-size: 14px;
    align-items: center;
    line-height: 2rem;
    font-weight: 500;
    transition: all 1300ms;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
    gap: 0.5rem;
  }
</style>
