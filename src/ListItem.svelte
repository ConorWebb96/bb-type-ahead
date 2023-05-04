<!-- ListItem.svelte -->
<script>
    import { createEventDispatcher } from 'svelte';
    
    export let labelColumn;
    export let valueColumn;
    export let fieldApi;
    export let disabled;
    export let option;
    export let selectedLabels;
    export let selectedValues;
    
    // create a reactive declaration to update the `isSelected` variable
    $: isSelected = selectedLabels.some(item => item.label === option[labelColumn]);
    $: console.log(`isSelected: ${isSelected}`);
    
    function selectedItem(label, value) {
      const index = selectedLabels.findIndex(item => item.label === label);
      if (index === -1) {
        // Label not yet selected, add it to the array
        selectedLabels.push({ label });
        selectedValues.push({ value });
      } else {
        // Label already selected, remove it from the array
        selectedLabels.splice(index, 1);
        selectedValues.splice(index, 1);
      }
      fieldApi.setValue(selectedValues.map(item => item.value)); // set the values on the field api
    }
</script>
    <li
        class="spectrum-Menu-item"
        class:is-selected={isSelected}
        role="option"
        aria-selected="true"
        tabindex="0"
        on:click={disabled ? null : () => selectedItem(option[labelColumn], option[valueColumn])}
        on:keydown={(event) => {
            if (event.key === 'Enter') {
            selectedItem(option[labelColumn], option[valueColumn])
            }
        }}
    >
        <span class="spectrum-Menu-itemLabel">
            {option[labelColumn]}
        </span> 
        <svg
            class="spectrum-Icon spectrum-UIIcon-Checkmark100 spectrum-Menu-checkmark spectrum-Menu-itemIcon"
            focusable="false"
            aria-hidden="true"
        >
            <use xlink:href="#spectrum-css-icon-Checkmark100" />
        </svg>
    </li>
<style>
    /* your styles go here */
</style>