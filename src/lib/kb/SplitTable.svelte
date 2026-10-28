<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // Counted the desk's way (CauseTable): every report under the cause it's filed by, and the desk's
  // one-day moves (desk-notes.json) as their own problem, counted again from the cause they sit under.
  type Problem = 'parser' | 'hint' | 'shift' | 'unplaced';
  let { ticket, issues, labels }: { ticket: number; issues: Partial<Record<Problem, string>>; labels: Partial<Record<Problem, string>> } =
    $props();

  const rows = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(rows));
  const groups = $derived(groupByCause(merged, tenantOf));
  const moves = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const table = $derived([
    ...groups.map((g) => ({ problem: g.cause as Problem, count: g.reports.length, tenants: g.tenants, firstSeen: g.firstSeen, lastSeen: g.lastSeen })),
    {
      problem: 'shift' as Problem,
      count: moves.length,
      tenants: new Set(moves.map((r) => r.tenant)).size,
      firstSeen: moves.map((r) => r.receivedAt.slice(0, 10)).sort()[0],
      lastSeen: moves.map((r) => r.receivedAt.slice(0, 10)).sort().at(-1),
    },
  ]);
  const mistagged = $derived(merged.filter((r) => r.meta.intakeTag !== groups.find((g) => g.reports.includes(r))?.cause).length);
  const num = (href: string) => `#${href.split('/').pop()}`;
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ·
  <strong>{table.length}</strong>
  {table.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each table as g (g.problem)}
      <tr class:open={!issues[g.problem]}>
        <td><code>{g.problem}</code> {labels[g.problem] ?? ''}</td>
        <td>{g.count}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>{#if issues[g.problem]}<a href={issues[g.problem]}>{num(issues[g.problem]!)}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#each groups as g (g.cause)}
  {@const moved = readAs(g, notes.oneDayMove)}
  {#if moved}
    <p class="note">{moved} of the <code>{g.cause}</code> reports read as a one-day move: counted under <code>{g.cause}</code>, where they're filed, and again under <code>shift</code>.</p>
  {/if}
{/each}

<p class="note">
  The export has {rows.length} rows for #{ticket}; {rows.length - merged.length} are the same report twice, so they are
  counted once. Intake's tag disagrees with the cause on {mistagged} reports.
</p>

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
</style>
