<script lang="ts">
  import { pretty_num } from 'elementari/labels'
  import type { Snippet } from 'svelte'
  import { titles_as_tooltips } from 'svelte-zoo/actions'
  import { flip } from 'svelte/animate'
  import { calc_cell_color } from './metrics'
  import type { CellSnippetArgs, CellVal, Metric, RowData } from './types'

  interface Props {
    data: RowData[]
    columns?: Metric[]
    sort_hint?: string
    style?: string | null
    class?: string
    cell?: Snippet<[CellSnippetArgs]>
    special_cells?: Record<string, Snippet<[CellSnippetArgs]>>
    controls?: Snippet
    initial_sort_column?: string
    initial_sort_direction?: `asc` | `desc`
    fixed_header?: boolean
    default_num_format?: string
    show_heatmap?: boolean
  }
  let {
    data,
    columns = [],
    sort_hint = ``,
    style = null,
    class: class_name = ``,
    cell,
    special_cells,
    controls,
    initial_sort_column,
    initial_sort_direction,
    fixed_header = false,
    default_num_format = `.3`,
    show_heatmap = $bindable(true),
  }: Props = $props()

  // Hacky helper function to detect if a string contains HTML, TODO revisit in future
  function is_html_str(val: unknown): boolean {
    if (typeof val !== `string`) return false
    // Check for common HTML patterns
    return (
      (val.includes(`<`) && val.includes(`>`)) || // Has angle brackets
      val.startsWith(`&lt;`) || // Has HTML entity for <
      val.includes(`<a `) || // Has anchor tag
      val.includes(`<span `) || // Has span tag
      val.includes(`<div `) || // Has div tag
      val.includes(`href=`) || // Has href attribute
      val.includes(`class=`) // Has class attribute
    )
  }

  // Add container reference for binding
  type SortState = { column: string; ascending: boolean }
  let sort_state = $state<SortState>({
    column: initial_sort_column || ``,
    ascending: initial_sort_direction !== `desc`,
  })

  let sorted_data = $state(data)
  $effect(() => {
    sorted_data =
      data?.filter?.((row) => Object.values(row).some((val) => val !== undefined)) ?? []
  })

  // Helper to make column IDs (needed since column labels in different groups can be repeated)
  const get_col_id = (col: Metric) =>
    col.group ? `${col.short ?? col.label} (${col.group})` : (col.short ?? col.label)

  function sort_rows(column: string, group?: string) {
    // Find the column using both label and group if provided
    const col = columns.find(
      (c) => c.label === column && (c.group === group || c.group === undefined),
    )

    if (!col) return // Skip if column not found

    // Skip sorting if column is explicitly marked as not sortable
    if (col.sortable === false) return

    const col_id = get_col_id(col)

    if (sort_state.column !== col_id) {
      sort_state.column = col_id
      sort_state.ascending = col.better === `lower`
    } else {
      sort_state.ascending = !sort_state.ascending
    }

    sorted_data = sorted_data.sort((row1, row2) => {
      const val1 = row1[col_id]
      const val2 = row2[col_id]

      if (val1 === val2) return 0
      if (val1 === null || val1 === undefined) return 1
      if (val2 === null || val2 === undefined) return -1

      const modifier = sort_state.ascending ? 1 : -1

      // Check if values are HTML strings with data-sort-value attributes
      if (typeof val1 === `string` && typeof val2 === `string`) {
        const sort_val1_match = val1.match(/data-sort-value="([^"]*)"/)
        const sort_val2_match = val2.match(/data-sort-value="([^"]*)"/)

        if (sort_val1_match && sort_val2_match) {
          const sort_val1 = sort_val1_match[1]
          const sort_val2 = sort_val2_match[1]

          // Try to convert to numbers if possible
          const num_val1 = Number(sort_val1)
          const num_val2 = Number(sort_val2)

          if (!isNaN(num_val1) && !isNaN(num_val2)) {
            return num_val1 < num_val2 ? -1 * modifier : 1 * modifier
          }

          // sort strings case-insensitively
          const [lower1, lower2] = [sort_val1.toLowerCase(), sort_val2.toLowerCase()]
          return lower1 > lower2 ? modifier : -1 * modifier
        }
      }

      return val1 < val2 ? -1 * modifier : 1 * modifier
    })
  }

  function calc_color(val: CellVal, col: Metric) {
    // Skip color calculation for null values or if color_scale is null
    if (
      val === null ||
      val === undefined ||
      col.color_scale === null ||
      typeof val !== `number` ||
      !show_heatmap // Disable heatmap colors if show_heatmap is false
    ) {
      return { bg: null, text: null }
    }

    const col_id = get_col_id(col)
    const numeric_vals = sorted_data
      .map((row) => row[col_id])
      .filter((val) => typeof val === `number`) // Type guard to ensure we only get numbers

    // Using the shared helper function for color calculation
    return calc_cell_color(
      val,
      numeric_vals,
      col.better === `higher` || col.better === `lower` ? col.better : undefined,
      col.color_scale || `interpolateViridis`,
      col.scale_type || `linear`,
    )
  }

  let visible_columns = $derived(columns.filter((col) => col.visible !== false))

  const sort_indicator = (col: Metric, sort_state: SortState) => {
    const col_id = get_col_id(col)
    if (sort_state.column === col_id) {
      // When column is sorted, show ↓ for ascending (smaller values at top)
      // and ↑ for descending (larger values at top)
      return `<span style="font-size: 0.8em;">${sort_state.ascending ? `↓` : `↑`}</span>`
    } else if (col.better) {
      // When column is not sorted, show arrow indicating which values are better:
      // ↑ for higher-is-better metrics
      // ↓ for lower-is-better metrics
      const sort_dir = col.better === `higher` ? `↑` : `↓`
      return `<span style="font-size: 0.8em;">${sort_dir}</span>`
    }
    return ``
  }
</script>

<div class="table-container {class_name}" {style} use:titles_as_tooltips>
  {#if (sort_state && sort_hint) || controls}
    <div class="table-header">
      {#if sort_state && sort_hint}
        <span class="sort-hint">{sort_hint}</span>
      {/if}
      {#if controls}
        {@render controls()}
      {/if}
    </div>
  {/if}
  <table class:fixed-header={fixed_header} class="heatmap" style="grid-column: 2;">
    <thead>
      <!-- Don't add a table row for group headers if there are none -->
      {#if visible_columns.some((col) => col.group)}
        <!-- First level headers -->
        <tr class="group-header">
          {#each visible_columns as { label, group, description } (label + group)}
            {#if !group}
              <th></th>
            {:else}
              {@const group_cols = visible_columns.filter((c) => c.group === group)}
              <!-- Only render the group header once for each group by checking if this is the first column of this group -->
              {#if visible_columns.findIndex((c) => c.group === group) === visible_columns.findIndex((c) => c.group === group && c.label === label)}
                <th title={description} colspan={group_cols.length}>{@html group}</th>
              {/if}
            {/if}
          {/each}
        </tr>
      {/if}
      <!-- Second level headers -->
      <tr>
        {#each visible_columns as col (col.label + col.group)}
          <th
            title={col.description}
            onclick={() => sort_rows(col.label, col.group)}
            style={col.style}
            class:sticky-col={col.sticky}
            class:not-sortable={col.sortable === false}
          >
            {@html col.short ?? col.label}
            {@html sort_indicator(col, sort_state)}
          </th>
        {/each}
      </tr>
    </thead>
    <tbody>
      {#each sorted_data as row (JSON.stringify(row))}
        <tr animate:flip={{ duration: 500 }} style={String(row.row_style ?? ``)}>
          {#each visible_columns as col (col.label + col.group)}
            {@const val = row[get_col_id(col)]}
            {@const color = calc_color(val, col)}
            <td
              data-col={col.label}
              data-sort-value={is_html_str(val) ? null : val}
              class:sticky-col={col.sticky}
              style:background-color={color.bg}
              style:color={color.text}
              style={col.cell_style ?? col.style}
            >
              {#if special_cells?.[col.label]}
                {@render special_cells[col.label]({ row, col, val })}
              {:else if cell}
                {@render cell({ row, col, val })}
              {:else if typeof val === `number`}
                {pretty_num(val, col.format ?? default_num_format)}
              {:else if val === undefined || val === null}
                n/a
              {:else}
                {@html val}
              {/if}
            </td>
          {/each}
        </tr>
      {/each}
    </tbody>
  </table>
</div>

<style>
  .table-container {
    overflow-x: auto;
    display: grid;
    grid-template-columns: 1fr min-content 1fr;
    font-size: var(--heatmap-font-size, 0.9em);
    /* https://stackoverflow.com/a/38994837 */
    scrollbar-width: none; /* Firefox */
    max-width: 90vw;
  }
  .table-container::-webkit-scrollbar {
    display: none; /* Safari and Chrome */
  }
  table {
    overflow: hidden;
  }
  th,
  td {
    padding: var(--heatmap-cell-padding, 1pt 5pt);
    text-align: var(--heatmap-text-align, left);
    border: var(--heatmap-cell-border, none);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  th {
    background: var(--heatmap-header-bg, var(--night));
    position: sticky;
    cursor: pointer;
  }
  th:hover {
    background: var(--heatmap-header-hover-bg, var(--night-lighter, #2a2a2a));
  }
  .sticky-col {
    position: sticky;
    left: 0;
    background: var(--heatmap-header-bg, var(--night));
    z-index: 1;
  }
  tr:nth-child(odd) td.sticky-col {
    background: var(--heatmap-row-odd-bg, rgb(15, 14, 14));
  }
  tbody tr:hover {
    filter: var(--heatmap-row-hover-filter, brightness(1.1));
  }
  td[data-sort-value] {
    cursor: default;
  }
  .group-header th {
    border-bottom: 1px solid black;
    text-align: center;
  }
  /* Styles for the table header with sort hint and controls */
  .table-header {
    grid-column: 2;
    width: 100%;
    display: flex;
    margin: 10pt auto;
    gap: 2em;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
    justify-content: space-between;
  }
  span.sort-hint {
    color: var(--text-muted, #aaa);
    margin: 0;
  }
  .not-sortable {
    cursor: default;
  }
</style>
