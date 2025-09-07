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
    export let highlighted = false;

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
        // Remove duplicates efficiently using Map for complex objects
        const labelMap = new Map();
        const valueMap = new Map();
        
        selectedLabels.forEach(item => {
            if (item && item.label != null) {
                labelMap.set(item.label, { label: item.label });
            }
        });
        
        selectedValues.forEach(item => {
            if (item && item.value != null) {
                const key = typeof item.value === 'object' ? JSON.stringify(item.value) : String(item.value);
                valueMap.set(key, item);
            }
        });
        
        selectedLabels = Array.from(labelMap.values());
        selectedValues = Array.from(valueMap.values());
        if (type == "array") {
            fieldApi.setValue(selectedValues.map(item => item.value[0])); // set the values on the field api
        } else if(type == "relationship") {
            fieldApi.setValue(selectedValues.map(item => item.value)); 
        }
        selectedLabels = selectedLabels; // update selected labels allows isSelected to change
        // Dispatch the selectedLabels back to the parent component
        dispatch('selectedLabels', selectedLabels);
        dispatch('selectedValues', selectedValues);
        dispatch('select');
    }
</script>
    <li
        class="spectrum-Menu-item"
        class:is-selected={isSelected}
        class:is-highlighted={highlighted}
        role="option"
        aria-selected={isSelected}
        tabindex="-1"
        on:click={disabled ? null : () => selectedItem(option[labelColumn], option[valueColumn])}
        on:keydown={(event) => {
            if (event.key === 'Enter' || event.key === ' ') {
                event.preventDefault();
                selectedItem(option[labelColumn], option[valueColumn]);
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
    .spectrum-Menu-item.is-highlighted {
        background-color: var(--spectrum-global-color-gray-200);
        cursor: pointer;
    }

    .spectrum-Menu-item.is-highlighted:not(.is-selected) {
        background-color: var(--spectrum-alias-highlight-hover);
    }

    .spectrum-Menu-item:focus-visible {
        outline: 2px solid var(--spectrum-global-color-blue-400);
        outline-offset: -2px;
    }
</style>