<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem. A one-day move (desk-notes.json) is not a swap, so it is its own problem,
  // not a footnote under the cause its tenant would give it.
  type Problem = 'parser' | 'hint' | 'one-day' | 'unplaced';
  let { ticket, issues }: { ticket: number; issues: Partial<Record<Problem, string>> } = $props();

  const all = $derived(reports as Report[]);
  const merged = $derived(uniqueById(all).filter((r) => r.mergedInto === ticket));
  const problemOf = (r: Report): Problem =>
    notes.oneDayMove.includes(r.id) ? 'one-day' : (causeOf(r, tenantOf(r)) ?? 'unplaced');

  const rows = $derived.by(() => {
    const by = new Map<Problem, Report[]>();
    for (const r of merged) by.set(problemOf(r), [...(by.get(problemOf(r)) ?? []), r]);
    return [...by.entries()]
      .map(([problem, rs]) => {
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          problem,
          count: rs.length,
          mistagged: rs.filter((r) => r.meta.intakeTag !== problem).length,
          tenants: new Set(rs.map((r) => r.tenant)).size,
          firstSeen: days[0],
          lastSeen: days[days.length - 1],
        };
      })
      .sort((a, b) => b.count - a.count);
  });
</script>

<p class="sum">
  <strong>{all.length}</strong> rows exported · <strong>{all.length - merged.length}</strong> duplicate rows dropped ·
  <strong>{merged.length}</strong> reports in #{ticket} · <strong>{rows.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>intake tag differs</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code></td>
        <td><strong>{r.count}</strong></td>
        <td>{r.mistagged}</td>
        <td>{r.tenants}</td>
        <td>{r.firstSeen} → {r.lastSeen}</td>
        <td>{#if issues[r.problem]}<a href={issues[r.problem]}>{issues[r.problem]?.replace(/^.*github\.com\//, '')}</a>{:else}none{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
