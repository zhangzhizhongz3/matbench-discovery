<script lang="ts">
  import type { ChemicalElement, ElementSymbol } from 'elementari'
  import { pretty_num } from 'elementari'

  interface Props {
    element: ChemicalElement
    elem_counts: number[] | Record<ElementSymbol, number>
    style?: string | null
    show_percent?: boolean
    unit?: string
  }
  let {
    element,
    elem_counts,
    style = null,
    show_percent = true,
    unit = ``,
  }: Props = $props()

  let value = $derived(
    Array.isArray(elem_counts)
      ? (elem_counts[element?.number - 1] ?? null)
      : (elem_counts[element?.symbol] ?? null),
  )
  let total = $derived(
    (Array.isArray(elem_counts) ? elem_counts : Object.values(elem_counts)).reduce(
      (total, count) => total + count,
      0,
    ),
  )
</script>

<strong {style}>
  {#if element?.name}
    {element?.name}: {pretty_num(value)}
    {@html unit}
    <!-- compute percent of total -->
    {#if show_percent && total > 0}
      ({pretty_num((100 * value) / total)}%)
    {/if}
  {/if}
</strong>

<style>
  strong {
    display: block;
    text-align: center;
    min-height: 18pt;
  }
</style>
