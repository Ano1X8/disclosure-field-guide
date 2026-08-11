# Vulnerability Disclosure: A Field Guide to the Channels

Notes from filing security reports through nine different channels — what each one wants, what each one
rejects, how long they take, and the mistakes that cost me time.

Most disclosure advice is written from the coordinator's side. This is written from the reporter's side,
after enough rejections to see the patterns. **Every rejection described here is one of mine.**

> **Scope.** This is about *process*, not findings. No unpublished vulnerability is described, and no case
> is identified before its advisory ships. Where a channel rejected something, the rejection reasoning is
> given because that is the useful part.

---

## The thing that took longest to learn

**Channel choice affects outcome more than finding quality does.**

The clearest evidence is a natural experiment that played out publicly on a single repository. Four
memory-safety bugs were reported there as **public GitHub issues**. All four were fixed. All four produced
**zero advisories and zero CVEs** — the fixes landed as ordinary commits and the reporters got a changelog
line.

In the same window, on the same repository, with the same maintainer, another researcher reported four bugs
through **Private Vulnerability Reporting**. They received **four CVEs with named credit**.

Same repo, same maintainer, same bug class. The only variable was the channel.

If a public artifact matters to you — and it should, because it is the only durable record that you found
something — then the channel is not an administrative detail. It is the decision.

---

## Choosing a channel

| Situation | Channel | Why |
|---|---|---|
| Project on GitHub with PVR enabled | **GitHub PVR** | Best outcome available. Maintainer publishes a GHSA, GitHub acts as CNA, credit is named and public. |
| GitHub project, PVR disabled, has `SECURITY.md` | **Email the contact** | Slower and less certain, but still private. |
| GitHub project, no PVR, no `SECURITY.md`, dormant | **Judgment call** — see below | There is no private channel to wait on. |
| Vendor with its own PSIRT | **Vendor PSIRT** | They will route it. Expect weeks to first contact. |
| Vendor is a CNA | **Vendor, and ask about CNA scope** | Prevents a duplicate CVE ID. See the pitfalls section. |
| Nothing above applies | **MITRE CNA-LR** | The fallback. Slow. |

**Check PVR before anything else.** It is one unauthenticated request:

```bash
curl -s https://api.github.com/repos/OWNER/REPO/private-vulnerability-reporting
```

`{"enabled":true}` means the best channel is available. Use it.

### When there is no private channel

If PVR is off, `SECURITY.md` is absent, and the maintainer is unresponsive, there is nothing to coordinate
*with*. Waiting silently protects nobody. The approach I settled on is a **calibrated public issue**:
vulnerability class, file, line, and a one-line fix — with the proof-of-concept withheld.

That gives maintainers and downstream users enough to act without handing over a working exploit. It is a
defensible position, but state your reasoning in the issue, because "filed a public vulnerability issue"
reads very differently depending on whether the reader knows you checked for a private channel first.

---

## Channel notes

### GitHub PVR

The best channel that exists, when it exists. Private until the maintainer publishes, GitHub issues the CVE
as a CNA, and credit is named on a permanent public page.

Two things to know:

**Draft advisories are invisible, and that is normal.** A filed report sits as an unpublished draft
visible only to you and the maintainers. To confirm from outside that a report is actually filed:

| Anonymous fetch | Authenticated API | Meaning |
|---|---|---|
| 404 | 403 *"must have the repository security advisories scope"* | **Filed, unpublished draft** |
| 404 | 404 | Never sent |
| 200 | 200 | Published |

A permissions error is not a not-found. That distinction resolved two reports I could not otherwise account
for.

**The API cannot tell you who was credited.** `credits[].user` is `null` on *both* the list endpoint and the
per-advisory endpoint. Only the rendered HTML page shows logins. If you are auditing your own record, scrape
the page or read it logged in — an empty API response means nothing.

### Vendor PSIRTs

Expect **weeks** to first contact, and expect that contact to be an acknowledgement rather than a verdict.
One report took 33 days to a first reply that only confirmed a ticket had been opened.

They will usually ask about your disclosure plans. Answer concretely: a specific embargo date, an explicit
offer to extend if a fix is in progress, and what would cause you to publish early. Vague good intentions
invite vague timelines.

**Ask whether the affected code is inside their CNA scope.** A large vendor's CNA scope often covers its
*products* but not its research organisation's repositories. If you assume wrong and file with MITRE too,
you get duplicate IDs for one bug. One sentence in your first reply prevents it.

### Microsoft (MSRC)

Microsoft evaluates whether a **trust boundary** is crossed, and the bar is specific. Five of my reports
closed on essentially the same reasoning:

> a local tool processing files the developer deliberately selected, with no automated or multi-tenant
> ingestion boundary — safer deserialization here is defense-in-depth.

That is a coherent position, not an unfair one. The lesson is to **name the boundary before you file**. If
the attack requires the victim to deliberately choose the attacker's file, and nothing ingests files
automatically, expect this outcome regardless of how clean the exploit is.

### Managed bug bounty platforms

Two structural constraints that are easy to miss:

**Some programs cap you at one submission in triage at a time.** Findings queue behind each other, and a
slow triage blocks everything else you have ready. Sequence deliberately: file the strongest first, not the
one you finished first.

**Programs reject whole classes, not just instances.** One program closed two model-load deserialization
reports as informative on the reasoning that loading a model artifact is equivalent to executing untrusted
code. That is a *class* verdict — the third report would have failed too. When a rejection reasoning would
apply equally to every sibling you have, stop filing siblings and re-examine the premise.

One rejection I want to describe honestly because I was wrong: I reported an unsafe-deserialization flag
without noticing that a `pickle.load` on the same data ran **two lines above** the flag I was reporting.
Patching my finding would have removed no risk. The program was right to close it.

**Before reporting a disabled safety default, verify it is load-bearing.** If the dangerous operation
happens anyway on another path, the flag is not the vulnerability.

### Broker programs (ZDI and similar)

**They buy by vendor, not by bug class.** A clean memory-corruption bug in a project whose vendor is not in
their purchasing program is worth nothing to them, regardless of quality. Vendor eligibility is the *first*
filter, not the last.

Their public *upcoming* queue is a useful dedup oracle, with a limitation: it lists the affected vendor and
nothing else. A vendor string like "PostgreSQL" does not distinguish core from an extension. A hit narrows a
candidate; it does not disqualify one.

Read the researcher agreement before submitting — particularly the exclusivity terms and what happens when
they decline.

### Project-specific security teams

Large projects with their own security process are often the best-run channel, and they are strict about
their own conventions:

- **The issue tracker may not be markdown.** One uses Jira wiki markup, where markdown pastes as an
  unreadable wall. Check before you paste.
- **Metadata fields matter.** I had two filings hand-corrected because a component field was set to a
  generic value instead of the specific one.
- **They publish a fix bar.** If they state the required fix is a specific permission check, argue your
  severity against *that* bar, not against a generic scoring rubric.

One economic note: a per-instance vulnerability class in a plugin ecosystem can be swept by another
researcher in a single publication drop. If you find a class, expect competition, and weigh whether filing
thirty instances at low severity is worth it before you start.

### MITRE CNA-LR

The fallback when no CNA covers the product. It works, and it is slow — plan in weeks.

The failure mode I hit: a batch submitted with a typo in the requester email. Those requests exist in
MITRE's system with no way to reach me, and they will not proactively find them. The fix was not chasing —
it was resubmitting through the web form, which mints new trackable request numbers.

**Check your reply-to address before submitting a batch.** It is the single cheapest thing on this page.

---

## Cross-cutting

### Dedup properly, and treat an empty result as a claim

**Advisory databases are not sufficient.** For some classes — particularly memory-safety bugs in ML and
native parsers — the most active researchers report publicly and take no CVE. None of their prior art
appears in OSV, GHSA or NVD. **The issue tracker and `git log` are the primary oracle for those classes;
the advisory databases are secondary.**

Always search:

```bash
gh search issues --repo OWNER/REPO "<parser or function name>" --state all
git log --grep "<function name>"      # silent fixes never get an advisory
```

**Validate your dedup query against a known positive before you trust a zero.** A query that returns nothing
because it is malformed looks exactly like a query that returns nothing because the finding is fresh. I have
made this mistake with a package-registry query that returned zero results for *every* input, including
things I knew were vulnerable.

**Dedup has a shelf life of hours.** Re-run it immediately before filing, not when you start writing.

### A fix at one call site predicts siblings — conditionally

When a project patches one instance of a class and leaves others, the others are often still live. But this
only holds when the call sites are **numerous and were added by different authors**. If one person who
understood the class patched all of it in a single commit, there is nothing left, and advisory volume will
mislead you into thinking otherwise.

The useful signal is a hardening commit that touches *one* site in a file containing four similar ones.

### "Credited" is not one thing

Before putting any finding on a résumé or portfolio, resolve what kind of credit it actually was:

- primary reporter on a published advisory
- co-discoverer
- duplicate, with credit granted anyway
- acknowledged privately, no public artifact

These are very different claims. I maintained a "credited" list for over a year before discovering one entry
was a duplicate where the public advisory credits someone else entirely. Internal tracking can be loose about
this. Anything a hiring manager or a program reads cannot be.

### Severity: say what it is

A denial of service is a denial of service. Inflating it to "potential remote code execution" because a
crash *might* be exploitable costs credibility that is expensive to rebuild, and triagers see the pattern
constantly. If you have not demonstrated control, do not claim it.

### Reachability before severity

A defect that no public entry point can reach is a code-quality issue. Establish the path from an attacker-
controlled input to the sink *first*, and describe severity in terms of the boundary actually crossed.

The corollary is worth stating: **a sink identified by reading source is a hypothesis, not a finding.** It
becomes a finding when a crafted input produces a sanitizer trace through the shipped code path — with the
guard that would refute it verified as absent, rather than assumed absent.

---

## Contributing

Corrections welcome, particularly from anyone with contradicting experience of these channels. Processes
change and this reflects one reporter's sample.

## License

CC BY 4.0 — use it, adapt it, credit it.
