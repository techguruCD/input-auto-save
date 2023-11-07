<script>
    import { onDestroy } from "svelte";
    import { InputDataCollection } from "../api/inputdatas";

    // Exported variables
    export let unique_id;
    export let alternativeItems = [];

    // Local state variables
    let input = null;
    let alternativeContainer = null;
    let alternativeVisible = false;
    let value = "";
    let selectionStart = 0;
    let selectionEnd = 0;
    let changedCounter = 0;
    let saveTimerID = 0;
    let isInitialized = false;
    let activeIndex = 0;
    let filteredItems = [];
    let x = 0,
        y = 0;
    let replaceStartIndex = 0;
    let originReplaceValue = "";
    let alternativeReplacingStatus = -1; // -1: not showing 0: showing(just changed) 1: showing(arrow)

    // Function to save changes to the database
    const saveChange = async () => {
        changedCounter = 0;
        let savedData = InputDataCollection.findOne({ unique_id });
        const data = {
            unique_id,
            value,
            selectionStart,
            selectionEnd,
        };
        if (savedData) await InputDataCollection.update(savedData._id, data);
        else await InputDataCollection.insert(data);
    };

    const getCursorXY = (input) => {
        let selectionPoint = input.selectionStart;
        let boundingRect = input.getBoundingClientRect();
        const { offsetLeft: inputX, offsetTop: inputY } = input;
        // create a dummy element that will be a clone of our input
        const div = document.createElement("div");
        // get the computed style of the input and clone it onto the dummy element
        const copyStyle = getComputedStyle(input);
        for (const prop of copyStyle) {
            div.style[prop] = copyStyle[prop];
        }
        div.style["position"] = "fixed";
        div.style["left"] = `${boundingRect.left}px`;
        div.style["top"] = `${boundingRect.top}px`;
        div.style["width"] = `${boundingRect.width}px`;
        div.style["height"] = `${boundingRect.height}px`;
        // we need a character that will replace whitespace when filling our dummy element if it's a single line <input/>
        const swap = ".";
        const inputValue =
            input.tagName === "INPUT"
                ? input.value.replace(/ /g, swap)
                : input.value;
        // set the div content to that of the textarea up until selection
        const textContent = inputValue.substr(0, selectionPoint);
        // set the text content of the dummy element div
        div.textContent = textContent;
        if (input.tagName === "TEXTAREA") div.style.height = "auto";
        // if a single line input then the div needs to be single line and not break out like a text area
        if (input.tagName === "INPUT") div.style.width = "auto";
        // create a marker element to obtain caret position
        const span = document.createElement("span");
        // give the span the textContent of remaining content so that the recreated dummy element is as close as possible
        span.textContent = inputValue.substr(selectionPoint) || ".";
        // append the span marker to the div
        div.appendChild(span);
        // append the dummy element to the body
        document.body.appendChild(div);
        // get the marker position, this is the caret position top and left relative to the input
        const { offsetLeft: spanX, offsetTop: spanY } = span;
        // lastly, remove that dummy element
        // NOTE:: can comment this out for debugging purposes if you want to see where that span is rendered
        document.body.removeChild(div);
        // return an object with the x and y of the caret. account for input positioning so that you don't need to wrap the input
        // return {
        //     x: inputX + spanX,
        //     y: inputY + spanY,
        // };
        return {
            x: boundingRect.left + spanX,
            y: boundingRect.top + spanY,
        };
    };

    const moveAlternativeContainer = () => {
        ({ x, y } = getCursorXY(input));
        let listContainerBoundaryRect =
            alternativeContainer.getBoundingClientRect();
        y -= listContainerBoundaryRect.height;
        if (y < 0) y += listContainerBoundaryRect.height + 30;
        x -= listContainerBoundaryRect.width / 2;
        if (x < 0) x = 0;
        else if (x + listContainerBoundaryRect.width > window.innerWidth)
            x = window.innerWidth - listContainerBoundaryRect.width;
    };

    const onAternativeContainerResize = (e) => {
        moveAlternativeContainer();
    };

    // Function to handle input changes
    const onChange = (e) => {
        if (alternativeVisible) {
            if (alternativeReplacingStatus == 1) {
                alternativeReplacingStatus = 0;
                return;
            }
            if (alternativeReplacingStatus == 0) {
                alternativeVisible = false;
                alternativeReplacingStatus = -1;
            }
        }
        const {
            selectionStart: newSelectionStart,
            selectionEnd: newSelectionEnd,
            value: newValue,
        } = e.target;

        if (value != newValue) {
            if (newSelectionStart == newSelectionEnd) {
                let index = newSelectionStart - 1;
                if (newValue[newSelectionStart - 1] == "/") {
                    replaceStartIndex = newSelectionStart - 1;
                    originReplaceValue = "";
                    alternativeVisible = true;
                    alternativeReplacingStatus = -1;
                } else if (alternativeVisible) {
                    for (; index >= 0; index--) {
                        if (newValue[index] == "/") break;
                        if (newValue[index] == "\n") index = 0;
                    }
                    if (index < 0) alternativeVisible = false;
                }
                if (alternativeVisible) {
                    activeIndex = 0;
                    originReplaceValue = newValue.substring(
                        index + 1,
                        newSelectionStart
                    );
                    setTimeout(moveAlternativeContainer, 100);
                }
            }
        }

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

    const replaceValue = (value) => {
        let { selectionStart } = input;
        input.value =
            input.value.substring(0, replaceStartIndex) +
            value +
            input.value.substring(selectionStart, input.value.length);
        input.setSelectionRange(
            replaceStartIndex + value.length,
            replaceStartIndex + value.length
        );
    };

    const replaceAlternative = () => {
        if (filteredItems[activeIndex])
            replaceValue(filteredItems[activeIndex]);
    };

    const onKeyDown = (e) => {
        if (e.key == "ArrowUp" || e.key == "ArrowDown") {
            if (alternativeVisible) {
                e.preventDefault();
                activeIndex += e.key == "ArrowUp" ? -1 : 1;
                if (activeIndex < 0) activeIndex = 0;
                else if (activeIndex >= filteredItems.length)
                    activeIndex = filteredItems.length - 1;
                activeItem = alternativeContainer.children[activeIndex];
                if (activeItem) {
                    let listBoundaryRect =
                        alternativeContainer.getBoundingClientRect();
                    let itemBoundaryRect = activeItem.getBoundingClientRect();
                    if (itemBoundaryRect.bottom > listBoundaryRect.bottom) {
                        activeItem.scrollIntoView(false);
                    } else if (itemBoundaryRect.top < listBoundaryRect.top) {
                        activeItem.scrollIntoView(true);
                    }
                    alternativeReplacingStatus = 1;
                    replaceAlternative();
                }
            }
        } else if (e.key == "Enter") {
            if (alternativeVisible) {
                e.preventDefault();
                alternativeVisible = false;
                if (alternativeReplacingStatus == -1) replaceAlternative();
            }
        } else if (e.key == "Escape") {
            alternativeVisible = false;
            replaceValue("/" + originReplaceValue);
        }
    };

    $m: {
        filteredItems = alternativeItems.filter((item) =>
            item.toLowerCase().includes(originReplaceValue)
        );
        if (activeIndex >= filteredItems.length)
            activeIndex = filteredItems.length - 1;
    }

    // Initialization logic using a reactive statement
    $m: {
        if (!isInitialized) {
            let savedData = InputDataCollection.findOne({ unique_id });
            if (savedData) {
                ({ value, selectionStart, selectionEnd } = savedData);
                if (input) {
                    isInitialized = true;
                    input.value = value;
                    input.setSelectionRange(selectionStart, selectionEnd);
                    input.focus();
                }
            }
        }
    }

    // Cleanup logic
    onDestroy(saveChange);
</script>

<textarea
    class="w-full h-full rounded border border-blue-400 focus-within:outline-blue-700 focus-within:shadow-lg resize-none p-5"
    bind:this={input}
    on:input={onChange}
    on:keydown={onKeyDown}
    on:keypress={onChange}
    on:keyup={onChange}
    on:mouseup={onChange}
/>
{#if alternativeVisible}
    <ul
        class="shadow-lg bg-white border border-gray-300 rounded-md p-2 max-h-40 h-fit overflow-y-auto scroll-mr-3 w-96"
        bind:this={alternativeContainer}
        style={`position:fixed;left:${x}px; top:${y}px; display:${
            alternativeVisible ? "block" : "none"
        }`}
        on:resize={onAternativeContainerResize}
    >
        {#each filteredItems as item, index}
            <li
                class={`flex z-10 items-center gap-x-2 px-2 py-1 relative cursor-pointer hover:bg-gray-50 hover:text-gray-900 rounded-md my-1 ${
                    index == activeIndex ? "bg-blue-100" : ""
                }`}
                on:mousedown={(e) => {
                    e.preventDefault();
                    activeIndex = index;
                    alternativeVisible = false;
                    replaceAlternative();
                }}
            >
                <div class="truncate">
                    {item}
                </div>
            </li>
        {/each}
        <div
            class="bg-blue fixed inset-0 z-0"
            on:mousedown={(e) => {
                e.preventDefault();
                alternativeVisible = false;
            }}
        />
    </ul>
{/if}
