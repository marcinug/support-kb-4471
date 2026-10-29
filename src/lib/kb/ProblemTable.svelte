<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket: the causes from causes.ts, plus the reports the desk read
  // as a one-day move (desk-notes.json), which are not a flipped date and get a row of their own.
  let { ticket, issues = {} }: { ticket: number; issues?: Record<string, string> } = $props();

  const ONE_DAY = 'one-day move';
  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const isMove = (r: Report) => notes.oneDayMove.includes(r.id);

  const rows = $derived.by(() => {
    const span = (rs: Report[]) => {
      const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
      return `${days[0]} → ${days[days.length - 1]}`;
    };
    const moved = merged.filter(isMove);
    return [
      ...groupByCause(merged.filter((r) => !isMove(r)), tenantOf).map((g) => ({
        problem: g.cause as string,
        reports: g.reports.length,
        tenants: g.tenants,
        seen: `${g.firstSeen} → ${g.lastSeen}`,
      })),
      ...(moved.length ? [{ problem: ONE_DAY, reports: moved.length, tenants: new Set(moved.map((r) => r.tenant)).size, seen: span(moved) }] : []),
    ];
  });
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
        <td>{#if issues[row.problem]}<a href={issues[row.problem]}>{issues[row.problem].replace(/^.*\/issues\//, '#')}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
