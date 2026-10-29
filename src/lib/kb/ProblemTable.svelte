<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket, counted the desk's way: each cause from causes.ts, then the
  // reports the desk read as a one-day move (desk-notes.json), counted where they're filed and again here.
  let { ticket, issues = {} }: { ticket: number; issues?: Record<string, string> } = $props();

  const ONE_DAY = 'one-day move';
  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));
  const moved = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));

  const rows = $derived.by(() => {
    const days = moved.map((r) => r.receivedAt.slice(0, 10)).sort();
    return [
      ...groups.map((g) => ({ problem: g.cause as string, reports: g.reports.length, tenants: g.tenants, seen: `${g.firstSeen} → ${g.lastSeen}` })),
      ...(moved.length
        ? [{ problem: ONE_DAY, reports: moved.length, tenants: new Set(moved.map((r) => r.tenant)).size, seen: `${days[0]} → ${days[days.length - 1]}` }]
        : []),
    ];
  });
  const link = (problem: string) => issues[problem];
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong>
  {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as row (row.problem)}
      <tr>
        <td><code>{row.problem}</code></td>
        <td>{row.reports}</td>
        <td>{row.tenants}</td>
        <td>{row.seen}</td>
        <td>{#if link(row.problem)}<a href={link(row.problem)}>{link(row.problem).replace(/^.*\/issues\//, '#')}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#each groups as g (g.cause)}
  {@const n = readAs(g, notes.oneDayMove)}
  {#if n}
    <p class="note">{n} of the <code>{g.cause}</code> reports read as a one-day move: counted under <code>{g.cause}</code> by tenant, and under <code>{ONE_DAY}</code>.</p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
</style>
