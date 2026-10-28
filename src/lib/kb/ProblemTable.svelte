<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem. A report the desk read as a one-day move (desk-notes.json) is its own
  // problem: nothing is swapped, so neither the parser nor the hint explains it.
  type Problem = { key: string; what: string; issue: string };
  let { ticket, problems }: { ticket: number; problems: Problem[] } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const problemOf = (r: Report) =>
    notes.oneDayMove.includes(r.id) ? 'one-day' : (causeOf(r, tenantOf(r)) ?? 'unplaced');

  const rows = $derived(
    [...new Set([...problems.map((p) => p.key), ...merged.map(problemOf)])].map((key) => {
      const rs = merged.filter((r) => problemOf(r) === key);
      const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
      return {
        key,
        p: problems.find((p) => p.key === key),
        count: rs.length,
        tenants: new Set(rs.map((r) => r.tenant)).size,
        // Intake guessed a different cause from the reporter's browser.
        mistagged: rs.filter((r) => r.meta.intakeTag !== key).length,
        seen: days.length ? `${days[0]} → ${days[days.length - 1]}` : '—',
      };
    }),
  );
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.filter((r) => r.count).length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>intake tagged otherwise</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.key)}
      <tr class:wrong={!r.p}>
        <td><code>{r.key}</code>{#if r.p}<br /><span class="what">{r.p.what}</span>{/if}</td>
        <td>{r.count}</td>
        <td>{r.tenants}</td>
        <td>{r.mistagged}</td>
        <td>{r.seen}</td>
        <td>{#if r.p}<a href={r.p.issue}>{r.p.issue.replace(/^.*\/issues\//, '#')}</a>{:else}no ticket yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .what { color: var(--muted); font-size: 0.85em; }
  .wrong td { background: var(--warn-bg); }
</style>
