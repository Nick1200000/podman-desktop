<style>
.grid-table {
  display: grid;
}
</style>

<script lang="ts" generics="T extends { selected?: boolean; name?: string }">
/* eslint-disable import/no-duplicates */
// https://github.com/import-js/eslint-plugin-import/issues/1479
import { faChevronDown, faChevronRight } from '@fortawesome/free-solid-svg-icons';
import { afterUpdate, onMount, tick } from 'svelte';
import Fa from 'svelte-fa';

import Checkbox from '../checkbox/Checkbox.svelte';
/* eslint-enable import/no-duplicates */
import type { Column, Row } from './table';

export let kind: string;
// eslint-disable-next-line @typescript-eslint/no-explicit-any
export let columns: Column<T, any>[];
export let row: Row<T>;
export let data: T[];
export let defaultSortColumn: string | undefined = undefined;
export let collapsed: string[] = [];
/**
 * To better distinct individual row, you can provide a dedicated key method
 *
 * By default, it will use the object name property
 */
export let key: (object: T) => string = item => item.name ?? String(item);

let tableHtmlDivElement: HTMLDivElement | undefined = undefined;

// number of selected items in the list
export let selectedItemsNumber: number = 0;
$: selectedItemsNumber = row.info.selectable
  ? data.filter(object => row.info.selectable?.(object) && object.selected).length +
    data.reduce(
      (previous, current) =>
        previous +
        (row.info.children?.(current)?.filter(child => row.info.selectable?.(child) && child.selected).length ?? 0),
      0,
    )
  : 0;

// do we need to unselect all checkboxes if we don't have all items being selected ?
let selectedAllCheckboxes = false;
$: selectedAllCheckboxes = row.info.selectable
  ? data.filter(object => row.info.selectable?.(object)).every(object => object.selected) &&
    data
      .reduce(
        (accumulator, current) => {
          const children = row.info.children?.(current);
          if (children) {
            accumulator.push(...children);
          }
          return accumulator;
        },
        [] as Array<T>,
      )
      .filter(child => row.info.selectable?.(child))
      .every(child => child.selected) &&
    data.filter(object => row.info.selectable?.(object)).length > 0
  : false;

function toggleAll(e: CustomEvent<boolean>): void {
  const checked = e.detail;
  if (!row.info.selectable) {
    return;
  }
  data.filter(object => row.info.selectable?.(object)).forEach(object => (object.selected = checked));

  // toggle children
  data.forEach(object => {
    const children = row.info.children?.(object);
    if (children) {
      children.filter(child => row.info.selectable?.(child)).forEach(child => (child.selected = checked));
    }
  });
  data = [...data];
}

let sortCol: Column<T>;
let sortAscending: boolean;

$: if (data && sortCol) {
  sortImpl();
}

function sort(column: Column<T>): void {
  if (!column) {
    return;
  }

  let comparator = column.info.comparator;
  if (!comparator) {
    // column is not sortable
    return;
  }

  if (sortCol === column) {
    sortAscending = !sortAscending;
  } else {
    sortCol = column;
    sortAscending = column.info.initialOrder ? column.info.initialOrder !== 'descending' : true;
  }
  sortImpl();
}

function sortImpl(): void {
  // confirm we're sorting
  if (!sortCol) {
    return;
  }

  let comparator = sortCol.info.comparator;
  if (!comparator) {
    // column is not sortable
    return;
  }

  if (!sortAscending) {
    // we're already sorted, switch to reverse order
    let comparatorTemp = comparator;
    comparator = (a, b): number => -comparatorTemp(a, b);
  }

  // eslint-disable-next-line etc/no-assign-mutated-array
  data = data.sort(comparator);
}

onMount(async () => {
  const column: Column<T> | undefined = columns.find(column => column.title === defaultSortColumn);
  if (column?.info.comparator) {
    sortCol = column;
    sortAscending = column.info.initialOrder ? column.info.initialOrder !== 'descending' : true;
    await tick(); // Ensure all initializations are complete.
    sortImpl(); // Explicitly call sortImpl to sort the data initially.
  }
});

afterUpdate(() => {
  tick()
    .then(() => setGridColumns())
    .catch((err: unknown) => console.error('unable to refresh grid columns', err));
});

function setGridColumns(): void {
  // section and checkbox columns
  let columnWidths: string[] = ['20px'];

  if (row.info.selectable) {
    columnWidths.push('32px');
  }

  // custom columns
  columns.map(c => c.info.width ?? '1fr').forEach(w => columnWidths.push(w));

  // final spacer
  columnWidths.push('5px');

  let wid = columnWidths.join(' ');
  if (tableHtmlDivElement) {
    let grids: HTMLCollection = tableHtmlDivElement.getElementsByClassName('grid-table');
    for (const element of grids) {
      (element as HTMLElement).style.setProperty('grid-template-columns', wid);
    }
  }
}

function objectChecked(object: T): void {
  // check for children and set them to the same state
  if (row.info.children) {
    const children = row.info.children(object);
    if (children) {
      // event fires before parent changes so use '!'
      children.forEach(child => (child.selected = !object.selected));
    }
  }
}

function toggleChildren(name: string | undefined): void {
  if (!name) {
    return;
  }

  if (collapsed.includes(name)) {
    const index = collapsed.indexOf(name, 0);
    if (index > -1) {
      collapsed.splice(index, 1);
    }
  } else {
    collapsed.push(name);
  }
  // trigger Svelte update
  collapsed = collapsed;
}
</script>

<div class="w-full mx-5" class:hidden={data.length === 0} role="table" aria-label={kind} bind:this={tableHtmlDivElement}>
  <!-- Table header -->
  <div role="rowgroup">
    <div
      class="grid grid-table gap-x-0.5 h-7 sticky top-0 text-[var(--pd-table-header-text)] uppercase z-2"
      role="row">
      <div class="whitespace-nowrap justify-self-start" role="columnheader"></div>
      {#if row.info.selectable}
        <div class="whitespace-nowrap place-self-center" role="columnheader">
          <Checkbox
            title="Toggle all"
            bind:checked={selectedAllCheckboxes}
            disabled={!row.info.selectable || data.filter(object => row.info.selectable?.(object)).length === 0}
            indeterminate={selectedItemsNumber > 0 && !selectedAllCheckboxes}
            on:click={toggleAll} />
        </div>
      {/if}
      {#each columns as column, index (index)}
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <!-- svelte-ignore a11y-interactive-supports-focus -->
        <div
          class="max-w-full overflow-hidden flex flex-row text-sm font-semibold items-center whitespace-nowrap {column
            .info.align === 'right'
            ? 'justify-self-end'
            : column.info.align === 'center'
              ? 'justify-self-center'
              : 'justify-self-start'} self-center select-none"
          class:cursor-pointer={column.info.comparator}
          on:click={sort.bind(undefined, column)}
          role="columnheader">
          <div class="overflow-hidden text-ellipsis">
            {column.title}
          </div>
          {#if column.info.comparator}<i
              class="fas pl-0.5"
              class:fa-sort={sortCol !== column}
              class:fa-sort-up={sortCol === column && sortAscending}
              class:fa-sort-down={sortCol === column && !sortAscending}
              class:text-[var(--pd-table-header-unsorted)]={sortCol !== column}
              aria-hidden="true"></i
            >{/if}
        </div>
      {/each}
    </div>
  </div>
  <!-- Table body -->
  <div role="rowgroup">
    {#each data as object (object)}
      {@const children = row.info.children?.(object) ?? []}
      {@const itemKey = key(object)}
      <div class="min-h-[48px] h-fit bg-[var(--pd-content-card-bg)] rounded-lg mb-2 border border-[var(--pd-content-table-border)]">
        <div
          class="grid grid-table gap-x-0.5 min-h-[48px] hover:bg-[var(--pd-content-card-hover-bg)]"
          class:rounded-t-lg={!collapsed.includes(itemKey) &&
            children.length > 0}
          class:rounded-lg={collapsed.includes(itemKey) ||
            children.length === 0}
          role="row"
          aria-label={object.name}>
          <div class="whitespace-nowrap place-self-center" role="cell">
            {#if children.length > 0}
              <button
                title={collapsed.includes(itemKey) ? 'Expand Row' : 'Collapse Row'}
                aria-expanded={!collapsed.includes(itemKey)}
                on:click={toggleChildren.bind(undefined, itemKey)}
              >
                <Fa
                  size="0.8x"
                  class="text-[var(--pd-table-body-text)] cursor-pointer"
                  icon={!collapsed.includes(itemKey) ? faChevronDown : faChevronRight} />
              </button>
            {/if}
          </div>
          {#if row.info.selectable}
            <div class="whitespace-nowrap place-self-center" role="cell">
              <Checkbox
                title="Toggle {kind}"
                bind:checked={object.selected}
                disabled={!row.info.selectable(object)}
                disabledTooltip={row.info.disabledText}
                on:click={objectChecked.bind(undefined, object)} />
            </div>
          {/if}
          {#each columns as column, index (index)}
            <div
              class="whitespace-nowrap {column.info.align === 'right'
                ? 'justify-self-end'
                : column.info.align === 'center'
                  ? 'justify-self-center'
                  : 'justify-self-start'} self-center {column.info.overflow === true
                ? ''
                : 'overflow-hidden'} max-w-full py-1.5"
              role="cell">
              {#if column.info.renderer}
                <svelte:component
                  this={column.info.renderer}
                  object={column.info.renderMapping ? column.info.renderMapping(object) : object} />
              {/if}
            </div>
          {/each}
        </div>

        <!-- Child objects -->
        {#if !collapsed.includes(itemKey) && children.length > 0}
          {#each children as child, i (child)}
            <div
              class="grid grid-table gap-x-0.5 hover:bg-[var(--pd-content-card-hover-bg)]"
              class:rounded-b-lg={i === children.length - 1}
              role="row"
              aria-label={child.name}>
              <div class="whitespace-nowrap justify-self-start" role="cell"></div>
              {#if row.info.selectable}
                <div class="whitespace-nowrap place-self-center" role="cell">
                  <Checkbox
                    title="Toggle {kind}"
                    bind:checked={child.selected}
                    disabled={!row.info.selectable(child)}
                    disabledTooltip={row.info.disabledText} />
                </div>
              {/if}
              {#each columns as column, index (index)}
                <div
                  class="whitespace-nowrap {column.info.align === 'right'
                    ? 'justify-self-end'
                    : column.info.align === 'center'
                      ? 'justify-self-center'
                      : 'justify-self-start'} self-center {column.info.overflow === true
                    ? ''
                    : 'overflow-hidden'} max-w-full py-1.5"
                  role="cell">
                  {#if column.info.renderer}
                    <svelte:component
                      this={column.info.renderer}
                      object={column.info.renderMapping ? column.info.renderMapping(child) : child} />
                  {/if}
                </div>
              {/each}
            </div>
          {/each}
        {/if}
      </div>
    {/each}
  </div>
</div>
