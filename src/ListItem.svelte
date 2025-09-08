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
    export let type;
    export let highlighted = false;
    export let displayMode = 'simple';
    export let imageColumn = null;
    export let emailColumn = null;

    const dispatch = createEventDispatcher();
    // create a reactive declaration to update the `isSelected` variable
    $: isSelected = selectedLabels.some(item => {
        const itemLabel = String(item.label || '');
        const optionLabel = String(option[labelColumn] || '');
        return itemLabel === optionLabel;
    });
    
    // Helper to get image URL
    $: imageUrl = displayMode === 'rich' && imageColumn && option[imageColumn] ? 
        (Array.isArray(option[imageColumn]) ? option[imageColumn][0]?.url : option[imageColumn]) : null;
    
    // Helper to generate initials
    function getInitials(name) {
        if (!name) return '?';
        return name.split(' ')
            .map(word => word.charAt(0))
            .join('')
            .toUpperCase()
            .substring(0, 2);
    }
    
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
            // Store objects with both label and value for proper display
            const labelsAndValues = selectedLabels.map((labelItem, index) => ({
                label: labelItem.label,
                value: selectedValues[index]?.value
            }));
            fieldApi.setValue(labelsAndValues);
        } else if(type == "relationship") {
            // Store objects with both label and value for proper display
            const labelsAndValues = selectedLabels.map((labelItem, index) => ({
                label: labelItem.label,
                value: selectedValues[index]?.value
            }));
            fieldApi.setValue(labelsAndValues);
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
        class:rich-item={displayMode === 'rich'}
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
        {#if displayMode === 'rich'}
            <div class="item-content">
                <div class="avatar-container">
                    {#if imageUrl}
                        <img src={imageUrl} alt={option[labelColumn]} class="avatar" />
                    {:else}
                        <div class="avatar-placeholder">
                            {getInitials(option[labelColumn])}
                        </div>
                    {/if}
                </div>
                <div class="text-content">
                    <span class="primary-text">{option[labelColumn]}</span>
                    {#if emailColumn && option[emailColumn]}
                        <span class="secondary-text">{option[emailColumn]}</span>
                    {/if}
                </div>
            </div>
        {:else}
            <span class="spectrum-Menu-itemLabel">
                {option[labelColumn]}
            </span>
        {/if}
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 256 256" class="spectrum-Icon spectrum-UIIcon-Checkmark100 spectrum-Menu-checkmark spectrum-Menu-itemIcon" focusable="false" aria-hidden="true"><path d="M229.66,77.66l-128,128a8,8,0,0,1-11.32,0l-56-56a8,8,0,0,1,11.32-11.32L96,188.69,218.34,66.34a8,8,0,0,1,11.32,11.32Z"></path></svg>
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

    /* Ensure checkmark is visible for selected items */
    .spectrum-Menu-item.is-selected .spectrum-Menu-checkmark {
        opacity: 1;
        visibility: visible;
        fill: var(--spectrum-global-color-static-blue-400);
    }
    .spectrum-Menu-item:not(.is-selected) .spectrum-Menu-checkmark {
        opacity: 0;
        visibility: hidden;
    }
    .spectrum-Menu-item.rich-item {
        padding: 0.75rem 1rem;
        min-height: auto;
    }

    .item-content {
        display: flex;
        align-items: center;
        gap: 0.75rem;
        width: 100%;
    }

    .avatar-container {
        flex-shrink: 0;
    }

    .avatar {
        width: 2.5rem;
        height: 2.5rem;
        border-radius: 50%;
        object-fit: cover;
        border: 2px solid var(--spectrum-global-color-gray-300);
    }

    .avatar-placeholder {
        width: 2.5rem;
        height: 2.5rem;
        border-radius: 50%;
        background: var(--spectrum-global-color-gray-400);
        color: white;
        display: flex;
        align-items: center;
        justify-content: center;
        font-weight: 600;
        font-size: 0.875rem;
        border: 2px solid var(--spectrum-global-color-gray-300);
    }

    .text-content {
        flex: 1;
        min-width: 0;
        display: flex;
        flex-direction: column;
        gap: 0.125rem;
    }

    .primary-text {
        font-weight: 500;
        color: var(--spectrum-global-color-gray-900);
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }

    .secondary-text {
        font-size: 0.875rem;
        color: var(--spectrum-global-color-gray-600);
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }
</style>