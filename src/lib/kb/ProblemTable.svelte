<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem, counted the desk's way (CauseTable): by cause, from the tenant. Reports the
  // desk read as a one-day move (desk-notes.json) stay counted where they're filed and get their own
  // row too: nothing in them is swapped, so they are a problem of their own.
  type Problem = { key: string; what: string; issue: string; state: string };
  let { ticket, problems }: { ticket: number; problems: Problem[] } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));

  const rows = $derived([
    ...groups.map((g) => ({ key: g.cause, reports: g.reports, oneDay: readAs(g, notes.oneDayMove) })),
    { key: 'one-day', reports: oneDay, oneDay: 0 },
  ]
    .filter((r) => r.reports.length)
    .map((r) => {
      const days = r.reports.map((x) => x.receivedAt.slice(0, 10)).sort();
      return {
        ...r,
        p: problems.find((p) => p.key === r.key),
        tenants: new Set(r.reports.map((x) => x.tenant)).size,
        seen: `${days[0]} → ${days[days.length - 1]}`,
      };
    }));
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th><th>state</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.key)}
      <tr class:wrong={!r.p}>
        <td><code>{r.key}</code>{#if r.p}<br /><span class="what">{r.p.what}</span>{/if}</td>
        <td>{r.reports.length}</td>
        <td>{r.tenants}</td>
        <td>{r.seen}</td>
        <td>{#if r.p}<a href={r.p.issue}>{r.p.issue.replace(/^.*\/issues\//, '#')}</a>{:else}no ticket yet{/if}</td>
        <td class:open={r.p?.state === 'open'}>{r.p?.state ?? '—'}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#each rows as r (r.key)}
  {#if r.oneDay}
    <p class="note">{r.oneDay} of the <code>{r.key}</code> reports read as a one-day move: counted under <code>{r.key}</code> by tenant, and again under <code>one-day</code>.</p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .what { color: var(--muted); font-size: 0.85em; }
  .note { color: var(--muted); font-size: 0.85em; }
  .wrong td { background: var(--warn-bg); }
  .open { color: var(--warn); font-weight: 600; }
</style>
