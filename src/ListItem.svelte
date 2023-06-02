<!-- ListItem.svelte -->
<script>
    import { createEventDispatcher, tick } from 'svelte';
    
    export let labelColumn;
    export let valueColumn;
    export let fieldApi;
    export let disabled;
    export let option;
    export let selectedLabels;
    export let selectedValues;
    export let type;

    const dispatch = createEventDispatcher();
    // create a reactive declaration to update the `isSelected` variable
    $: isSelected = selectedLabels.some(item => item.label === option[labelColumn]);
    
    function selectedItem(label, value) {
        const index = selectedLabels.findIndex(item => item.label === label);
        if (index === -1) {
            if (type == "string") {
                // clear arrays so only one item can be within
                selectedLabels.splice(0, selectedLabels.length); 
                selectedValues.splice(0, selectedValues.length); 
                fieldApi.setValue(value); 
            }
            // Label not yet selected, add it to the array
            selectedLabels.push({ label });
            selectedValues.push({ value });
        } else {
            // Label already selected, remove it from the array
            selectedLabels.splice(index, 1);
            selectedValues.splice(index, 1);
            if (type == "string") {
                fieldApi.setValue(null); // remove if string type and removed for values.
            }
        }
        // Remove duplicates from selectedLabels and selectedValues
        const uniqueSelectedLabels = [...new Set(selectedLabels.map(item => item.label))];
        const uniqueSelectedValues = [...new Set(selectedValues.map(item => JSON.stringify(item.value)))];
        selectedLabels = uniqueSelectedLabels.map(label => ({ label }));
        selectedValues = uniqueSelectedValues.map(value => ({ value: JSON.parse(value) }));
        if (type == "array") {
            fieldApi.setValue(selectedValues.map(item => item.value[0])); // set the values on the field api
        } else if(type == "relationship") {
            fieldApi.setValue(selectedValues.map(item => item.value)); 
        }
        selectedLabels = selectedLabels; // update selected labels allows isSelected to change
        // Dispatch the selectedLabels back to the parent component
        dispatch('selectedLabels', selectedLabels);
        dispatch('selectedValues', selectedValues);
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