<script>
  import ListItem from './ListItem.svelte';

  import { getContext, onDestroy } from "svelte"
  import { fly } from "svelte/transition"
  import clickOutside from "./click_outside"

  const { API } = getContext("sdk");
  const component = getContext("component")

  const debounce = (callback, minDelay = 1000) => {
    let timeout
    return async (...params) => {
      return new Promise(resolve => {
        if (timeout) {
          clearTimeout(timeout)
        }
        timeout = setTimeout(async () => {
          resolve(await callback(...params))
        }, minDelay)
      })
    }
  }

  export let dataSource
  export let label;
  export let placeholder;
  export let defaultValue
  export let minSearchLength = 1;
  
  export let limitResults;
  export let optionsType;
  export let field;
  export let array;
  export let relationship;
  let bindedField = optionsType === 'relationship' ? relationship : optionsType === 'fields' ? field : optionsType === 'array' ? array : null;
  let type = optionsType === 'relationship' ? 'relationship' : optionsType === 'array' ? 'array' : 'string';

  export let searchOptionsType;
  export let fieldFilters;
  export let searchRelationship;
  export let searchField;
  export let searchArray;
  export let labelColumn;
  export let valueColumn;

  export let disabled;
  export let sort;
  export let onChange;
  export let validation;

  let searching = false;
  let searchString = '';
  let searchFieldState;
  let fieldApi;
  let fieldState;
  let results = [];
  let searchCache = new Map();
  let abortController = null;
  let open = false;
  let highlightedIndex = -1;
  let popoverElement;

  $: selectedLabels = [];
  $: selectedValues = [];

  const { styleable } = getContext("sdk");
  // form variables
  const formContext = getContext("form")
  const formStepContext = getContext("form-step")
  const fieldGroupContext = getContext("field-group")
  // Register field with form
  const formApi = formContext?.formApi;
  const labelPos = fieldGroupContext?.labelPosition || "above";
  $: formStep = formStepContext ? $formStepContext || 1 : 1
  $: formField = formApi?.registerField(
    bindedField,
    optionsType,
    defaultValue,
    disabled,
    validation,
    formStep
  )
  $: labelClass = labelPos === "above" ? "" : `spectrum-FieldLabel--${labelPos}`
  // Update form properties in parent component on every store change
  $: unsubscribe = formField?.subscribe(value => {
    fieldState = value?.fieldState
    fieldApi = value?.fieldApi
  })
  // destroy field if deregistered
  onDestroy(() => {
    fieldApi?.deregister()
    unsubscribe?.()
  })
  // field type select
  searchFieldState = searchOptionsType === 'relationship' ? searchRelationship : searchOptionsType === 'searchFields' ? searchField : searchOptionsType === 'searchArray' ? searchArray : null;
  $: debouncedSearch(searchString);

  // Initialise selected values if fieldState?.value is present
  $: if (fieldState?.value) {
    if (type === 'string') {
      selectedLabels = [{ label: fieldState.value }];
      selectedValues = [{ value: fieldState.value }];
    } else {
      selectedLabels = fieldState.value.map(item => ({ label: item[labelColumn] }));
      selectedValues = fieldState.value.map(item => ({ value: item[valueColumn] }));
    }
  }

  const search = async (searchString) => {
    if (!searchString || searchString.length < minSearchLength) {
      results = [];
      return;
    }

    // Check cache first
    const cacheKey = `${searchString}-${searchFieldState}-${fieldFilters}`;
    if (searchCache.has(cacheKey)) {
      results = searchCache.get(cacheKey);
      return;
    }

    // Cancel previous request
    if (abortController) {
      abortController.abort();
    }
    abortController = new AbortController();

    searching = true;
    
    try {
      let queryParam = {
        [fieldFilters]: {
          [searchFieldState]: searchString,
        },
      };

      if (searchOptionsType === 'searchArray') {
        queryParam = {
          string: {
            [searchFieldState]: [searchString],
          },
        };
      }

      const searchResults = await API.searchTable(dataSource.tableId, {
        paginate: false,
        limit: limitResults,
        query: queryParam,
      });

      if (searchResults?.rows) {
        let processedResults = searchResults.rows.map(row => ({
          _id: row._id,
          [labelColumn]: row[labelColumn],
          [valueColumn]: row[valueColumn]
        }));

        if (sort && processedResults.length > 1) {
          processedResults.sort((a, b) => {
            const labelA = (a[labelColumn] || '').toString().toLowerCase();
            const labelB = (b[labelColumn] || '').toString().toLowerCase();
            return labelA.localeCompare(labelB);
          });
        }

        results = processedResults;
        searchCache.set(cacheKey, results);

        // Limit cache size
        if (searchCache.size > 50) {
          const firstKey = searchCache.keys().next().value;
          searchCache.delete(firstKey);
        }
      } else {
        results = [];
      }
    } catch (error) {
      if (error.name !== 'AbortError') {
        console.error('Search failed:', error);
        results = [];
      }
    } finally {
      searching = false;
      abortController = null;
    }
  }
  const debouncedSearch = debounce(search, 750);

  const onClick = () => {
    open = true;
    highlightedIndex = -1;
  }
  const handleOutsideClick = event => {
    if (open) {
      event.stopPropagation();
      open = false;
      highlightedIndex = -1;
    }
  };

  const handleKeyDown = (event) => {
    if (!open) {
      if (event.key === 'Enter' || event.key === 'ArrowDown') {
        event.preventDefault();
        onClick();
      }
      return;
    }

    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        highlightedIndex = Math.min(highlightedIndex + 1, results.length - 1);
        break;
      case 'ArrowUp':
        event.preventDefault();
        highlightedIndex = Math.max(highlightedIndex - 1, -1);
        break;
      case 'Enter':
        event.preventDefault();
        if (highlightedIndex >= 0 && highlightedIndex < results.length) {
          const option = results[highlightedIndex];
          selectOption(option);
        }
        break;
      case 'Escape':
        event.preventDefault();
        open = false;
        highlightedIndex = -1;
        break;
    }
  };

  const selectOption = (option) => {
    // This will be handled by the ListItem component
    // but we need to close the dropdown and reset highlight
    open = false;
    highlightedIndex = -1;
  };
  // reset filters
  function clearSelectedResults() {
    if (fieldApi && typeof fieldApi.setValue === 'function') {
      fieldApi.setValue(null);
    }
    selectedLabels = [];
    selectedValues = [];
    results = [];
    searchString = '';
    searchCache.clear();
  }
  function handleSelectedLabels(event) {
    selectedLabels = event.detail;
  }
  function handleSelectedValues(event) {
    selectedValues = event.detail;
    if (onChange) { // apply onchange here related to the value
      if (type == "string") {
        onChange({ value: event.detail[0].value });
      }else {
        onChange({ value: event.detail});
      }
    }
  }
</script>
asdasd
  <div class="spectrum-Form-item {labelClass === "above" ? "flexCol" : ""}" use:styleable={$component.styles}>
    {#if !formContext}
      <div class="placeholder">Form components need to be wrapped in a form</div>
    {:else}
      {#if !searchField}
        <div class="error">Please select a field</div>
      {/if}
      {#if fieldState?.error}
        <div class="error">{fieldState.error}</div>
      {/if}
      <label
        class:hidden={!label}
        for={fieldState?.fieldId}
        class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel svelte-tta2b5 ${labelClass}`}
      >
        {label || " "}
      </label>
      <div class="spectrum-Form-itemField typehead">
          <button 
            class="spectrum-Picker w-full spectrum-Picker--sizeM"
            on:click={disabled ? null : onClick}
            on:keydown={disabled ? null : handleKeyDown}
            class:disabled={disabled}
            aria-haspopup="listbox"
            aria-expanded={open}
            aria-label={label || 'Type ahead selector'}
            id="typeahead-button-{fieldState?.fieldId}"
          >
            {#if selectedLabels.length}
              <span class="spectrum-Picker-label is-placeholder">
                {#each selectedLabels.slice(0, 4) as label, idx}
                  {#if idx > 0}, {/if}
                  {label.label}
                {/each}
                <strong>{#if selectedLabels.length > 4} + {selectedLabels.length - 4}{/if}</strong>
              </span>
            {:else}
               <span class="spectrum-Picker-label is-placeholder">{ placeholder ? placeholder : 'Choose some options' }</span>
            {/if}
            <svg class="spectrum-Icon spectrum-UIIcon-ChevronDown100 spectrum-Picker-menuIcon" focusable="false" aria-hidden="true"><use xlink:href="#spectrum-css-icon-Chevron100"></use></svg>
          </button>
          {#if open}
            <div
              bind:this={popoverElement}
              use:clickOutside={handleOutsideClick}
              transition:fly|local={{ y: -20, duration: 200 }}
              class="spectrum-Popover spectrum-Popover--bottom spectrum-Picker-popover is-open w-full"
              role="listbox"
              aria-labelledby="typeahead-button-{fieldState?.fieldId}"
            >
              <div class="spectrum-Textfield w-full">
                <svg class="spectrum-Icon spectrum-Icon--sizeM spectrum-Textfield-icon" focusable="false" aria-hidden="true"><use xlink:href="#spectrum-icon-18-Magnify"></use></svg>
                <input 
                  type="search"
                  bind:value={searchString}
                  on:keydown={handleKeyDown}
                  class="spectrum-Textfield-input spectrum-Search-input"
                  placeholder="Search"
                  aria-label="Search options"
                  autocomplete="off"
                  spellcheck="false"
                />
                <button type="reset" on:click={() => clearSelectedResults()} class="spectrum-ClearButton spectrum-Search-clearButton"><svg class="spectrum-Icon spectrum-UIIcon-Cross75" focusable="false" aria-hidden="true"><use xlink:href="#spectrum-css-icon-Cross75"></use></svg></button>
              </div>
              {#if labelColumn != null }
                {#if searching}
                  <span class="spinner spinner-large"></span>
                {:else}
                   <ul class="spectrum-Menu" role="listbox" aria-label="Search results">
                    {#if searching == false && results.length < 1}
                      <span class="no-results">No results found</span>
                    {:else}
                       <!-- else content here -->
                       {#each results as option, index}
                        <ListItem
                          labelColumn={labelColumn}
                          valueColumn={valueColumn}
                          disabled={disabled}
                          option={option}
                          fieldApi={fieldApi}
                          selectedLabels={selectedLabels}
                          selectedValues={selectedValues}
                          on:selectedLabels={handleSelectedLabels}
                          on:selectedValues={handleSelectedValues}
                          type={type}
                          highlighted={highlightedIndex === index}
                          on:select={() => selectOption(option)}
                        />
                       {/each}
                    {/if}
                   </ul>
                {/if}
              {:else}
                <p>Sorry but you haven't assigned a label.</p>
              {/if}
            </div>
          {/if}
      </div>
    {/if}
  </div>
<style>
  .flexCol {
    display: flex;
    flex-direction: column;
  }
  .disabled {
    background: var(--spectrum-global-color-gray-200);
    color: var(--spectrum-global-color-gray-500);
  }
  .typehead .spectrum-Popover {
    z-index: 99;
  }
  .spectrum-Textfield.w-full {
    border-bottom: 1px solid var(--spectrum-textfield-m-border-color, var(--spectrum-alias-border-color));
  }
  ul.spectrum-Menu {
    max-height: 250px;
  }
  .spectrum-Textfield input {
    border: none!important;
  }
  .w-full {
    width: 100%;
  }
 .placeholder {
    color: var(--spectrum-textfield-m-border-color);
  }
  label {
    white-space: nowrap;
  }
  label.hidden {
    padding: 0;
  }
  .spectrum-Form-itemField {
    position: relative;
    width: 100%;
  }
  .error {
    color: var(
      --spectrum-semantic-negative-color-default,
      var(--spectrum-global-color-red-500)
    );
    font-size: var(--spectrum-global-dimension-font-size-75);
    margin-top: var(--spectrum-global-dimension-size-75);
  }
  .spinner {
    width: 1.5rem;
    height: 1.5rem;
    border-top-color: #444;
    border-left-color: #444;
    animation: spinner 400ms linear infinite;
    border-bottom-color: transparent;
    border-right-color: transparent;
    border-style: solid;
    border-width: 2px;
    border-radius: 50%;  
    box-sizing: border-box;
    display: inline-block;
    vertical-align: middle;
  }
  .spinner-large {
    width: 2.5rem;
    height: 2.5rem;
    border-width: 6px;
  }
  @keyframes spinner {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  .customSvgColour {
    fill: var(--spectrum-global-color-gray-900);
  }

  /* Loading state improvements */
  .spinner {
    margin: 1rem auto;
    display: block;
  }

  /* Better visual feedback */
  .spectrum-Picker:focus-visible {
    outline: 2px solid var(--spectrum-global-color-blue-400);
    outline-offset: 2px;
  }

  .spectrum-Search-input:focus {
    outline: none;
    border-color: var(--spectrum-global-color-blue-400) !important;
  }

  /* No results message styling */
  .no-results {
    display: block;
    padding: 0.75rem 1rem;
    color: var(--spectrum-global-color-gray-600);
    font-style: italic;
    text-align: center;
  }
</style>