<script>
    import {onDestroy} from 'svelte'
    import { InputDataCollection } from "../api/inputdatas";
    
    // Exported variables
    export let unique_id;

    // Local state variables
    let input = null;
    let value = "";
    let selectionStart = 0;
    let selectionEnd = 0;
    let changedCounter = 0;
    let saveTimerID = 0;
    let isInitialized = false;

    // Function to save changes to the database
    const saveChange = async () => {
        changedCounter = 0;
        let savedData = InputDataCollection.findOne({ unique_id });
        if (savedData)
            await InputDataCollection.update(savedData._id, {
                unique_id,
                value,
                selectionStart,
                selectionEnd,
            });
        else
            await InputDataCollection.insert({
                unique_id,
                value,
                selectionStart,
                selectionEnd,
            });
    };

    // Function to handle input changes
    const onChange = (e) => {
        const {
            selectionStart: newSelectionStart,
            selectionEnd: newSelectionEnd,
            value: newValue,
        } = e.target;
        if (
            selectionStart != newSelectionStart ||
            selectionEnd != newSelectionEnd ||
            value != newValue
        ) {
            selectionStart = newSelectionStart;
            selectionEnd = newSelectionEnd;
            value = newValue;

            clearTimeout(saveTimerID);
            if (++changedCounter > 100) saveChange();
            else saveTimerID = setTimeout(saveChange, 100);
        }
    };

    // Initialization logic using a reactive statement
    $m: {
        if (!isInitialized) {
            let savedData = InputDataCollection.findOne({ unique_id });
            if (savedData) {
                ({ value, selectionStart, selectionEnd } = savedData);
                if (input) {
                    isInitialized = true;
                    input.value = value;
                    input.setSelectionRange(selectionStart, selectionEnd)
                    input.focus()
                }
            }
        }
    }

    // Cleanup logic
    onDestroy(saveChange)
</script>

<textarea
    class="w-full h-full rounded border border-blue-400 focus-within:outline-blue-700 focus-within:shadow-lg resize-none p-5"
    bind:this={input}
    on:input={onChange}
    on:keypress={onChange}
    on:keyup={onChange}
    on:mouseup={onChange}
/>
