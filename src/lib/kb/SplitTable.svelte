<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import type { Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';
  import { splitTicket, type Problem } from '#lib/kb/split';

  // issues: where each problem is tracked now, problem → link.
  let { ticket, issues, labels }: { ticket: number; issues: Partial<Record<Problem, string>>; labels: Partial<Record<Problem, string>> } =
    $props();

  const split = $derived(splitTicket(reports as Report[], ticket, tenantOf, notes.oneDayMove));
  const mistagged = $derived(split.groups.reduce((n, g) => n + (g.problem === 'shift' ? 0 : g.mistagged), 0));
  const num = (href: string) => `#${href.split('/').pop()}`;
</script>

<p class="sum">
  <strong>{split.unique}</strong> reports in #{ticket} ·
  <strong>{split.groups.length}</strong>
  {split.groups.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each split.groups as g (g.problem)}
      <tr class:open={!issues[g.problem]}>
        <td><code>{g.problem}</code> {labels[g.problem] ?? ''}</td>
        <td>{g.reports.length}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>{#if issues[g.problem]}<a href={issues[g.problem]}>{num(issues[g.problem]!)}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<p class="note">
  The export has {split.rows} rows for #{ticket}; {split.rows - split.unique} are the same report twice, so they are
  counted once. Intake's tag disagrees with the problem on {mistagged} of the parser and hint reports.
</p>

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
</style>
