<script lang="ts">
    import { goto } from "$app/navigation";
    import { writable } from "svelte/store";
    interface DataFrame {
        columns: string[];
        rendered: boolean[];
        rows: any[];
    }
    let df = writable<DataFrame | null>(null);
    df.subscribe((df) => {
        if (df == null) return;
        const df_rendered_str = df.rendered
            .map((r) => (r ? "1" : "0"))
            .join("");
        goto("?df_rendered=" + df_rendered_str);
    });

    let selectedRow = writable<number | null>(null);
    const selectedRowValid = () => {
        if ($df == null) return false;
        if (selectedRow == null) return false;
        if (!Number.isInteger(selectedRow)) return false;

        const num = Number(selectedRow);
        if (num < 0 || num >= $df.rows.length) return false;

        return true;
    };
    selectedRow.subscribe((selectedRow) => {
        if (selectedRowValid()) {
            goto("?selected_row=" + selectedRow);
        } else {
            goto("?selected_row=");
        }
    });

    let files: FileList | null = null;
    $: if (files) {
        const reader = new FileReader();
        reader.readAsText(files[0], "UTF-8");
        reader.onerror = (evt) => {
            console.error("error reading file");
            console.error(evt);
        };

        reader.onload = (evt) => {
            if (!evt.target) {
                console.error("Reading file but got no target.");
                return;
            }
            if (!evt.target.result) {
                console.error("Reading file but got no target.result.");
                return;
            }

            // Read and parse jsonl file
            const fileContents = String(evt.target.result);
            const lines = fileContents.split("\n");
            const jsons = lines
                .map((line) => {
                    try {
                        return JSON.parse(line);
                    } catch (e) {
                        return null;
                    }
                })
                .filter((json) => json !== null);

            // Get set of all keys
            const keySet = new Set<string>();
            jsons.forEach((json) => {
                Object.keys(json).forEach((key) => {
                    keySet.add(key);
                });
            });
            const keys = Array.from(keySet).toSorted();

            df.set({
                columns: Array.from(keys),
                rendered: Array.from(keys).map((_) => true),
                rows: jsons,
            });

            console.log(jsons.length);
            console.log(jsons[100]);
        };
    }
</script>

<title>jsonl viewer</title>
<label for="file_selector">Choose jsonl file:&emsp;</label>
<input type="file" name="file_selector" bind:files />
<br /><br />

{#if $df}
    <!-- Make a button toggle per column in the dataframe -->
    {#if $df.columns.length > 0}
        <button
            on:click={() => {
                if ($df) {
                    $df.rendered = $df.rendered.map((_) => true);
                }
            }}
        >
            Show all fields
        </button>
        <button
            on:click={() => {
                if ($df) {
                    $df.rendered = $df.rendered.map((_) => false);
                }
            }}
        >
            Hide all fields
        </button>
        <br />

        <span>Fields:</span>
        {#each $df.columns as column, i}
            <button
                class={$df.rendered[i] ? "bold" : "normal"}
                on:click={() => {
                    if ($df) $df.rendered[i] = !$df.rendered[i];
                }}
                draggable="true"
            >
                {column}
            </button>
            &ensp;
        {/each}
        <br />
    {/if}

    <!-- Input field for what row of df to visualize -->
    <label for="row_selector"
        >Select row (0 - {$df.rows.length - 1}):&emsp;</label
    >
    <input
        type="number"
        name="row_selector"
        min="0"
        max={$df.rows.length - 1}
        bind:value={selectedRow}
    />
    <br />

    <!-- Print selected row -->
    {#each $df.columns as column, i}
        {#if $df.rendered[i] && selectedRowValid() && selectedRow !== null}
            <span><b>{$df.columns[i]}:</b> </span>
            {#if String($df.rows[selectedRow][column]).includes("\n")}
                <!-- Add newline is content spans multiple lines. -->
                <br />
            {/if}
            <code class="render_with_whitespace"
                >{$df.rows[selectedRow][column]}</code
            >
            <br />
        {/if}
    {/each}
{/if}

<style>
    .bold {
        font-weight: bold;
    }
    .normal {
        font-weight: normal;
    }
    .render_with_whitespace {
        white-space: pre-wrap;
    }
</style>
