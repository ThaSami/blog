---
title: "I Replaced Our Commercial Artifact Registry With a Free One After a 5× Renewal. Here's Everything That Broke."
date: 2026-05-31 10:00:00 +0400
categories: [Engineering, Infrastructure]
tags: [open-source, devops, kubernetes, postgres, migration]
description: "How I migrated our entire artifact management stack (Maven, npm, PyPI, Helm) from our commercial artifact registry to an open source registry, after the renewal came back at 5× the price."
toc: true
---

For years our build infrastructure rested on a tool almost nobody on the team thought about until it hurt: a commercial artifact registry holding every JAR, wheel, npm tarball, and Helm chart our services and mobile apps depended on. More than **1,000 microservices and CI/CD jobs** resolved their dependencies through it, across a store of **14 million+ artifacts**. It worked. Then the renewal quote came back at **five times** what we'd been paying.

That number is what turned "we should look at alternatives someday" into a project with my name on it. This is the honest version of what followed: the plan, the cutover, and the long tail of bugs that don't show up in any vendor's comparison chart.

## The background: a 5× renewal for a commodity

An artifact registry is plumbing. It stores bytes, serves them back over a handful of package manager protocols, and proxies a few public upstreams. The proprietary product we ran did this well, but it is, architecturally, a content addressed blob store with some format specific index logic on top. And the new renewal asked us to pay 5× last year's price for it.

Two things made that an easy "no." First, the **cost** was now indefensible for the marginal value. Second, **ownership**: when the registry misbehaved, our only lever was a support ticket, and we'd lost days to incidents waiting on vendor escalation. I wanted a registry where, if something broke at 2 a.m., I could read the source.

## Why artifact-keeper (and what I rejected)

I evaluated two replacements: a commercial alternative and an open source project called **artifact-keeper**, a single Rust/Axum binary backed by S3 for blobs and Postgres for metadata.

The other major commercial option was the safe choice: mature, supported, known. But it was another license, another vendor relationship, another black box. artifact-keeper was the opposite bet, MIT licensed, $0 in fees, and a codebase small enough that one engineer could understand it end to end. The tradeoff was equally clear: **no vendor SLA, and a young project**, barely three months into its 1.x line, with a single dominant maintainer.

The deciding factor was architecture. artifact-keeper mapped cleanly onto infrastructure we already ran, Kubernetes, S3, managed Postgres, pod level IAM. No JVM heap tuning, no clustered HA license tier, no per format plugin bundles. And the "young project" risk cut both ways: the same youth that made it rough meant I could *fix* the rough parts and upstream them.

My first PoC report actually recommended **holding**. I'd found three release blocking migration bugs and didn't trust the data path. What changed the call wasn't a new release. It was a leadership decision to back the risk anyway, betting that the gaps were closable.

## The plan: migrate the hot set, cut over at DNS

I avoided a big bang migration. Three pillars:

**Migrate what builds actually use.** The old Maven repos held ~300 GB, but most of it was historical versions no build had touched in months. Rather than copy all of it, I filtered the source to artifacts modified in a recent window and bulk migrated only that hot set (more below). I *did* trial a pull through cache, a remote pointed at the old system, lazy fetching on demand, but only on a separate **dev instance**, against two or three test pipelines, purely to validate client behavior. It was never the production path; the main libraries were bulk migrated for real.

| Format | Production strategy |
|---|---|
| Helm | Bulk copy (small, bounded inventory) |
| npm / PyPI | Bulk copy of local packages + a proxy remote for public ones |
| Maven / mobile libs | Bulk migrate the recently modified hot set |

**Cut over at DNS, not in 1000+ repos.** The naive plan sends every team a PR changing their build to a new URL. I didn't want to coordinate hundreds of those. Instead I pointed the *old* registry's hostname at the new system's load balancer and rewrote the URL shape at the edge (more below), so existing builds kept resolving against the URLs they already had.

**A rollback that actually rolls back.** The old registry stayed read only for 30 days after the cutover. The rollback lever was DNS, flip the record back, not a restore from backup.

## The hard parts (where the real story lives)

### The bundled migration tool quietly lied about success

The first thing I tested was the native migration worker. It ran, reported `status: completed`, every checksum matched, and **every migrated path returned 404.** Bytes were written to storage, but no database rows referenced them, and the destination repositories were never created.

Reading the Rust, the cause was a single `if let` that silently did nothing when the destination repo lookup failed. Two issues compounded it: a `concurrent_transfers` config field that nothing in the worker actually read (the loop was sequential `await` with a throttle delay *per item*), and a `/pause` endpoint that permanently truncated a job to `completed`. I filed all three upstream and fixed the worst, **provisioning destination repos before transfer**, in [a merged PR](https://github.com/artifact-keeper/artifact-keeper/pull/1017).

For our own migration of the main mobile library repos, ~17 million files across them, the lesson was simpler: **don't use the worker.** I wrote a direct pipeline, Python `asyncio` + `aiohttp`, one connection pooled process driving 50 concurrent transfers, fed by a query that filtered the source to only recently modified artifacts:

```
items.find({ "repo": "<repo>", "type": "file",
             "modified": { "$gt": "<cutoff>" } })
```

That single filter cut the largest repo from 14.3 million files to 600,000, the actual hot set. The rest stayed on the old system as a cold backstop. (This big run came *after* I'd fixed the production blocker in the next section. The first real production migration didn't get that far.)

### The metadata bug that broke every version, and the SQL to undo it

Even when bytes landed correctly, package managers couldn't find them. The migration code bound the *filename* to the artifact's `name` column and **never set `version` at all.** For Maven that's survivable; for PyPI, npm, and Helm, whose index endpoints are generated from those columns, it was fatal. `pip install` 404'd. `helm pull` saw an empty index.

I fixed it in three layers. Upstream, a format aware path parser so future migrations are correct ([merged](https://github.com/artifact-keeper/artifact-keeper/pull/1078), my first real contribution to the project). For rows already migrated, a set of **idempotent SQL repair scripts** that reconstructed the missing columns straight from the path, run from throwaway Kubernetes pods. The Helm one is representative:

```sql
UPDATE artifacts a
   SET name    = regexp_replace(split_part(a.path, '/', 3), '\.tgz$', ''),
       version = split_part(a.path, '/', 2)
  FROM repositories r
 WHERE a.repository_id = r.id
   AND r.format = 'helm'
   AND a.version IS NULL;
```

Every script was guarded (`WHERE version IS NULL`) so reruns were safe, and wrapped in a transaction. A few hundred charts that encoded their version only in the directory, not the filename, needed a second pass that also rewrote the path to the canonical `<chart>-<version>.tgz` shape the serve handler expects.

### The first real production migration failed on a pip timeout

The dev instance had passed its handful of test pipelines, so I ran the first real migration against the **production** instance. It failed, not with 404s, but with *timeouts*. The PyPI proxy/virtual path was adding **7 to 15 seconds of latency per upstream wheel**, while npm through the same framework stayed under a second. pip's default socket timeout is 15 seconds, so the first install of any cold upstream wheel was a coin flip, and enough of them lost the toss to sink the run.

I traced it to three things stacked: two sequential uncached upstream round trips per wheel, a fully buffered response path that read the whole artifact into memory before responding, and a cache layout that funnelled every proxy hit onto a single deep storage prefix instead of the distributed content addressed layout local repos used. I filed it [with the latency numbers](https://github.com/artifact-keeper/artifact-keeper/issues/1263); the fix landed upstream and dropped cold wheel fetches from ~15 s to under 1.5 s. *Then* the real migrations could proceed, starting with the mobile libraries above.

### "Return the first member, not the union", the same bug in two places

This pattern cost me the most debugging, because it showed up twice wearing different clothes.

First in **PyPI**: a virtual repo aggregating a local index and a public proxy index returned only the *first populated member's* `simple/<project>/` listing instead of unioning them. So a package present in both, internal fork *and* upstream, showed only one set of versions. `pip` couldn't satisfy transitive constraints it should have been able to, failing with `ResolutionImpossible` on dependencies that demonstrably existed. I fixed the resolver to splice entries across all members and dedupe, [merged upstream](https://github.com/artifact-keeper/artifact-keeper/pull/1267), closing [the issue I'd filed](https://github.com/artifact-keeper/artifact-keeper/issues/1230).

Then in **Maven**, and this time I caused it. To unblock one build I'd copied a single version of a public library into a *local* repo. Days later a different build failed pulling a *different* version of that same library through the aggregating virtual, even though the upstream proxy had it. Same root cause: the virtual binds an artifact to the first member that contains it and stops. By seeding one version locally, I'd made that member "claim" the whole coordinate and shadow every other version the proxy could serve. The fix was counterintuitive. **Delete the thing I'd added**, and let the virtual fall through to the proxy for all versions.

### A 135,000× database speedup hiding in one index

The pip timeouts had a second, independent cause hiding underneath the proxy latency, and this one hit every format, not just PyPI. The hot read on the whole system, "find this artifact by filename", did a leading wildcard `LIKE` on a path column, which Postgres can't index, so it scanned. On a 1.6 million row table that meant **~2,700 ms per lookup**, and the query routinely slammed into the 10 second statement timeout ceiling. Under concurrent load it compounded: requests queued behind those scans and read path p99 in a saturating burst climbed **past 30 seconds**, well beyond pip's 15 second default. So a slow `LIKE` in one table was quietly manufacturing client timeouts across the registry.

The fix was a functional index on the *reversed* path, turning a leading wildcard into an indexable prefix:

```sql
CREATE INDEX idx_artifacts_repo_reverse_path
  ON artifacts (repository_id, reverse(path) text_pattern_ops)
  WHERE is_deleted = false;
```

Mean lookup dropped from **2,700 ms to 0.02 ms**, about 135,000×, touching 6 buffer pages instead of 14,000, and flat regardless of repo size. The throughput effect was just as stark: a Maven read burst from inside the cluster, the one the scan had been choking, went from roughly **400 to ~2,012 requests per second across three backend pods**, at a 0.6 second p99 with zero errors. [Merged upstream](https://github.com/artifact-keeper/artifact-keeper/pull/1285), closing [the issue](https://github.com/artifact-keeper/artifact-keeper/issues/1266).

### The scary one: every new hire could mint themselves an admin key

This is the finding that made me stop and breathe. We wired up SSO so developers onboarded themselves, and every new user correctly landed without admin rights. But while reading the token issuance code, I realized the scope list on a new token was validated only for *length*, never *content*. Any signed in user without admin rights could call the token creation endpoint and request scopes like `admin`, `*`, or `delete:repositories`, and get them. I confirmed it empirically: an ordinary account minted a working token with `["admin"]`.

In other words, the moment we onboarded the whole engineering org, every one of them was a single API call away from full control of the registry, including deleting repositories. I filed it and [shipped the fix](https://github.com/artifact-keeper/artifact-keeper/pull/1261): an allowlist of admin class scopes that only an existing admin can grant, wired into both token issuance paths, with regression tests for the privilege escalation case. A [sibling fix](https://github.com/artifact-keeper/artifact-keeper/pull/1258) split the user management router so ordinary users could still manage their *own* tokens and password (which a too broad middleware had been [blocking](https://github.com/artifact-keeper/artifact-keeper/issues/1257)) without that opening the scope hole. Both merged.

Finding a privilege escalation bug in the tool you're about to put every developer behind, and *not* reporting it, isn't an option. Owning the source is what made it a pull request instead of a prayer.

### The smaller cuts

Briefly: npm served only a single `latest` dist-tag, and chose it by upload recency rather than semver, so custom tags like `next` or `canary` couldn't be installed and `latest` could resolve to a prerelease; I fixed it upstream by persisting dist-tags and deriving `latest` from semver. The official Helm chart was self labeled an "example configuration" and needed real hardening. And because storage is content addressed by hash, per repo size stats undercount, you read bucket metrics for true capacity.

## The cutover: 200 teams, zero config changes

With the mobile libraries migrated and the proxy fixed, the question was how to flip traffic without asking every team to edit their build. The catch: our old registry and artifact-keeper speak different URL shapes. The old system served `/ack-me/<repo>/...` and `/ack-me/api/pypi/<repo>/...`; artifact-keeper expects native prefixes, `/maven/<repo>/...`, `/pypi/<repo>/...`, `/npm/<repo>/...`, `/helm/<repo>/...`.

So I cut over at DNS and translated the URLs at the load balancer. I pointed the old registry's hostname at the new system's load balancer (Kubernetes Gateway API on an ALB) and added rewrite rules mapping the old shape onto the native one:

```
/ack-me/api/pypi/<repo>/...  →  /pypi/<repo>/...
/ack-me/api/npm/<repo>/...   →  /npm/<repo>/...
/ack-me/<repo>/...           →  /maven/<repo>/...
/ack-me/<helm-repo>/...      →  /helm/<helm-repo>/...
```

Every existing build kept working against the URLs it already had, no PR, no per team coordination, no flag day for 1000+ repos. Two edge constraints made it fiddly: the Gateway API caps a single route at 16 rules, so I split the set across several routes; and its rewrite filter is prefix only with no regex, so each repo needs its own explicit rule. Tedious, but the alternative was hundreds of synchronized config changes across teams I didn't control.

## Results

The headline: **a 5× renewal, sidestepped.** We were paying a license priced per node, on a pair of HA nodes, each a modest **4 core / 16 GB** box, for what is architecturally a commodity blob store. When the renewal came back at **five times** the prior year, I replaced the whole thing instead. The license dropped to **$0** (MIT), and the open source stack runs on a few thousand dollars a month of AWS for compute, Postgres, and S3, so a steeply rising license line became a small, flat cloud bill. The savings are real and recurring, and they widen every year the old contract would have escalated.

Beyond the dollars:

- Single, cloud native deployment on infrastructure we already operate, no JVM tuning, no HA license tier.
- Every bug above I could diagnose by reading code and fix by writing it, [ten of those fixes](#the-upstream-fixes) are now merged upstream, so the next team migrating inherits a better, safer tool.
- Artifact lookups dropped out of the request budget entirely, and a privilege escalation hole is closed for everyone, not just us.

## What I'd do differently

- The most expensive lesson was a migration tool reporting success while producing nothing. I now verify the *destination* independently. The real test is whether a client can resolve a migrated artifact, never the job's own status field.

- Bulk inserting metadata to "fix" missing rows created the Maven shadowing bug. In any system that aggregates or proxies, adding a record can change resolution for records you didn't touch. Test the read path after every write path repair.

- The PoC was a few weeks. The real work was the weeks *after* cutover, per build failures, each a slightly different root cause behind the same generic error. The migration isn't done when traffic flips; it's done when the failure reports stop.

- Every painful moment here would have been a support ticket on the old stack. Instead each was a pull request. For plumbing this fundamental, that was worth far more than the license savings.

## If you're considering the same move

If you're weighing this migration yourself, the point of everything above is that you don't have to start from zero. Every bug in this post is filed upstream and ten of the fixes are already merged into the mainline, so the registry you'd adopt today is materially safer and faster than the one I began with. The migration tooling, the idempotent SQL repair scripts, the DNS-plus-URL-rewrite cutover, and the habit of verifying the destination instead of trusting a "completed" status are all reusable patterns, not one-off hacks. The expensive part of my migration wasn't the cutover, it was discovering these sharp edges one failed build at a time, and that's exactly the part you get to skip.

## The upstream fixes

Everything above that says "merged upstream" is public. The fixes, all merged into [artifact-keeper](https://github.com/artifact-keeper/artifact-keeper), in case they're useful to anyone else running this registry:

- [#1017](https://github.com/artifact-keeper/artifact-keeper/pull/1017), migration: create destination repos before transferring (the "completed but empty" bug)
- [#1078](https://github.com/artifact-keeper/artifact-keeper/pull/1078), migration: populate `name`/`version` from a format aware filename parse (the broken index bug)
- [#1216](https://github.com/artifact-keeper/artifact-keeper/pull/1216), maven: virtual download for GAV grouped secondary files (`.pom`, `.module`, `-sources.jar`)
- [#1258](https://github.com/artifact-keeper/artifact-keeper/pull/1258), users: split the router so ordinary users can manage their own tokens/password (closes [#1257](https://github.com/artifact-keeper/artifact-keeper/issues/1257))
- [#1261](https://github.com/artifact-keeper/artifact-keeper/pull/1261), auth: block ordinary users from granting admin class scopes on token issuance
- [#1267](https://github.com/artifact-keeper/artifact-keeper/pull/1267), pypi: union virtual member entries on `simple/<project>/` (closes [#1230](https://github.com/artifact-keeper/artifact-keeper/issues/1230))
- [#1268](https://github.com/artifact-keeper/artifact-keeper/pull/1268), migration: correct column names + required NOT NULL columns
- [#1269](https://github.com/artifact-keeper/artifact-keeper/pull/1269), migration: raise statement/lock timeout for the migration session
- [#1285](https://github.com/artifact-keeper/artifact-keeper/pull/1285), perf: functional `reverse(path)` index to make the suffix `LIKE` indexable (closes [#1266](https://github.com/artifact-keeper/artifact-keeper/issues/1266))
- [#1557](https://github.com/artifact-keeper/artifact-keeper/pull/1557), npm: persist and serve dist-tags, deriving `latest` by semver instead of upload recency (closes [#1543](https://github.com/artifact-keeper/artifact-keeper/issues/1543))

Plus the PyPI proxy latency report ([#1263](https://github.com/artifact-keeper/artifact-keeper/issues/1263)) that drove the cold wheel speedup.
