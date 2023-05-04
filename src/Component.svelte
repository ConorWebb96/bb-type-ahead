<script>
  import ListItem from './ListItem.svelte';

  import { getContext, onDestroy } from "svelte"
  import { fly } from "svelte/transition"
  import clickOutside from "./click_outside"

  const { API, notificationStore } = getContext("sdk");
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

  export let optionsType;
  export let field;
  export let array;
  let bindedField = optionsType === 'relationship' ? relationship : optionsType === 'fields' ? field : optionsType === 'array' ? array : null;

  export let searchOptionsType;
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
  let searchString;
  let optionsTypeState;
  let fieldApi;
  let fieldState;
  let results = [];
  let open = false;

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
    "string",
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
  optionsTypeState = searchOptionsType === 'relationship' ? searchRelationship : searchOptionsType === 'searchFields' ? searchField : searchOptionsType === 'searchArray' ? searchArray : null;

  $: debouncedSearch(searchString);

  const search = async (searchString) => {
    if (searchString == null || searchString.length < 1) {
      return 
    }
    searching = true; // start search loading
    let queryParam = {
      string: {
        [`${optionsTypeState}`]: searchString || "",
      },
    };
    // if (searchOptionsType === 'relationship') {
    //   queryParam = {
    //     string: {
    //       [`1:${optionsTypeState}`]: searchString || "",
    //     },
    //   };
    // }
    if (searchOptionsType === 'array') {
      // Additional logic here to convert search string to array after ,
      
      queryParam = {
        string: {
          [`${optionsTypeState}`]: [searchString] || "",
        },
      };
    }

    const searchResults = await API.searchTable({
      paginate: false,
      tableId: dataSource.tableId,
      limit: 20,
      query: queryParam,
      paginate: false,
    });

    if (sort === true) {
      if (Array.isArray(searchResults)) { // check if searchResults is an array
        searchResults.sort((a, b) => {
          const optionsTypeStateA = a.optionsTypeState.toLowerCase();
          const optionsTypeStateB = b.optionsTypeState.toLowerCase();
          if (optionsTypeStateA < optionsTypeStateB) {
            return -1;
          }
          if (optionsTypeStateA > optionsTypeStateB) {
            return 1;
          }
          return 0;
        });
      } else {
        // handle error exception here
      }
    }
    searching = false; // end searching loading
    results = searchResults.rows.map(({ _id, [labelColumn]: val, [valueColumn]: value }) => ({ _id, [labelColumn]: val, [valueColumn]: value })); // reduce array to include id and selected field
  }
  const debouncedSearch = debounce(search, 750);

  const onClick = () => {
    open = true
  }
  const handleOutsideClick = event => {
    if (open) {
      event.stopPropagation()
      open = false
    }
  }
</script>
{JSON.stringify(fieldState)}

  <div class="spectrum-Form--labelsAbove" use:styleable={$component.styles}>
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
            on:click={onClick}
          >
            <span class="spectrum-Picker-label is-placeholder">{ placeholder ? placeholder : 'Choose some options' }</span>
            <svg class="spectrum-Icon spectrum-UIIcon-ChevronDown100 spectrum-Picker-menuIcon" focusable="false" aria-hidden="true"><use xlink:href="#spectrum-css-icon-Chevron100"></use></svg>
          </button>
          {#if open}
            <div
              use:clickOutside={handleOutsideClick}
              transition:fly|local={{ y: -20, duration: 200 }}
              class="spectrum-Popover spectrum-Popover--bottom spectrum-Picker-popover is-open w-full"
            >
              <div class="spectrum-Textfield w-full">
                <svg class="spectrum-Icon spectrum-Icon--sizeM spectrum-Textfield-icon" focusable="false" aria-hidden="true"><use xlink:href="#spectrum-icon-18-Magnify"></use></svg>
                <input 
                  type="search"
                  bind:value={searchString}
                  class="spectrum-Textfield-input spectrum-Search-input"
                  placeholder="Search"
                />
                <button type="reset" on:click={() => clearSelectedResults()} class="spectrum-ClearButton spectrum-Search-clearButton"><svg class="spectrum-Icon spectrum-UIIcon-Cross75" focusable="false" aria-hidden="true"><use xlink:href="#spectrum-css-icon-Cross75"></use></svg></button>
              </div>
              {#if labelColumn != null }
                {#if searching}
                  <span class="spinner spinner-large"></span>
                {:else}
                   <ul class="spectrum-Menu" role="listbox">
                     {#each results as option, idx}
                      <ListItem
                        labelColumn={labelColumn}
                        valueColumn={valueColumn}
                        disabled={disabled}
                        option={option}
                        fieldApi={fieldApi}
                        selectedLabels={selectedLabels}
                        selectedValues={selectedValues}
                      />
                     {/each}
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
  .typehead .spectrum-Popover {
    z-index: 99;
  }
  .spectrum-Textfield.w-full {
    border-bottom: 1px solid var(--spectrum-textfield-m-border-color, var(--spectrum-alias-border-color));
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
</style>