# Why I won't hold your `service_role` key

Every vendor that touches your database asks for a credential. Most of them ask for the easiest one. I want to explain why I decided not to, what that decision costs me, and where the thing I chose instead is worse than the alternative I turned down.

This document records the architecture decision for the person who would have to approve the integration.

## 1. What that key actually is

A Supabase secret API key (`sb_secret_…`) maps to the `service_role` Postgres role. That role carries `BYPASSRLS`.

`BYPASSRLS` is not "elevated permissions." It means row-level security is not applied at all. Say you wrote this, and you were right to:

```sql
alter table public.documents enable row level security;

create policy tenant_isolation on public.documents
  for all
  using (tenant_id = (auth.jwt() ->> 'org_id')::uuid);
```

A connection authenticated with your secret key does not fail `tenant_isolation`. It never evaluates it. Every row of `public.documents` is visible and writable, for every tenant, in one `select *`.

Two more platform facts that bound the whole problem, both from Supabase's own API-keys documentation. You can create several secret keys and revoke them individually — that is a real improvement over the single legacy key, and it matters. And: **API keys cannot be bound to a custom Postgres role.** There is no read-only secret key. There is no insert-into-one-table secret key. Every one of them is `service_role`.

So when a vendor asks for that key, the request is not "access to the table you're importing into." The request is your entire database, read and write, with your policies switched off, held on their infrastructure, for as long as the integration exists.

Their breach is your breach. That's not a risk you mitigate with encryption at rest or a scoped IAM policy on their side. There's no key shape that makes it smaller. There is only avoidance.

## 2. The architecture I rejected

The rejected design is the obvious one, and I want to be specific about how obvious it is:

You paste `sb_secret_…` into a settings field. I store it encrypted. When a customer of yours uploads a spreadsheet, my runtime validates it, connects to your project's REST API with your key, and inserts the accepted rows. If a batch times out, I query back and reconcile. Done.

That design deletes, in one stroke, essentially every hard thing on my roadmap:

- no generated SQL migration, so no SQL compiler
- no role, no grants, no policies to install in your database
- no customer-hosted component to version, deploy, or upgrade
- no signature scheme, no key registry, no rotation and revocation runbook
- no dual-approval handshake between your backend and mine
- no in-database ledger, because I'd keep the bookkeeping on my side
- no per-customer schema review before anything is allowed to run
- no uninstall or revocation path, because there's nothing installed

I won't put a number on what that cost me, because I never ran the counterfactual and any figure I gave you would be invented. But look at the list. It isn't a feature I skipped. It's most of the project. The rejected model would have shipped very substantially sooner, and it is the reason a well-funded competitor can move faster than I can on everything except this one property.

I rejected it because the first sentence of my own security posture would have been a lie. You cannot say "your data is protected by your RLS policies" and simultaneously hold the credential that switches them off.

## 3. The alternatives that were actually hard to turn down

Two of the rejected options are good. Pretending otherwise would make this post marketing.

### A scoped Postgres login

Instead of the secret key, you create a dedicated Postgres role with `INSERT` on exactly one table, give it a password, and hand me a connection string. This is genuine least privilege. It is roughly how established ELT vendors operate, and it is a defensible answer.

I rejected it as the default for three reasons, in descending order of how much they bothered me:

1. The credential still leaves your environment. Smaller blast radius, same category of promise: trust my key handling.
2. It can't be provisioned automatically. Supabase's Management API does not return your database password, so there's no path where I set this up for you. Every customer does manual role creation, a manual password handoff, and manual rotation forever after.
3. The sentence on your security questionnaire is unchanged: _the vendor has a login to our production database._

I kept the pattern in the document as a candidate for self-hosted or raw-Postgres destinations later — that's a planned direction, not a supported one — but it is not what the Supabase path uses.

### Your own endpoint

The design that beats mine on containment: I never write anything. I validate, map, and correct the file, then deliver signed, idempotency-keyed batches to an HTTP endpoint you own, and your code does the insert. My write access to your database is exactly zero, by construction, permanently.

If your constraint is absolute — no vendor code near the database, no exceptions — this is the right answer and I'll say so on a call.

I didn't make it the default because of what it hands back to you. You would implement the idempotency ledger. You would implement replay handling for the batch that arrived twice because my retry crossed your `200`. You would implement tenant enforcement, so a `tenant_id` in the payload can't override the one in the session. You would implement outcome reporting precise enough to tell your customer which twelve rows failed and why. That is the backend this product exists to remove — and when a team gets it subtly wrong, the duplicate rows still show up in a support ticket addressed to me.

### A generated adapter running in your project

The third one is more subtle, and it's the one I spent longest on: I generate a TypeScript function, you deploy it into your own project, it holds a project-local key and does the writes. Vendor code, customer environment.

It falls apart on atomicity. A function using `supabase-js` cannot wrap N inserts and a ledger write in a single transaction. To become atomic it has to call a database function — at which point it _is_ the design I picked, plus a network hop. With a direct Postgres driver it can be atomic, but then the enforcement logic lives in regenerated vendor JavaScript holding database credentials, which is a larger and softer surface to review than SQL you can read in a migration diff. Supabase also documents 2 seconds of CPU and 256 MB per Edge Function request, which is a real ceiling for per-row work in JS on a large batch.

## 4. What I chose instead — planned, and not built yet

**Read this section as a design, not a product.** Every mechanism below is planned and not built; none of it is installed in anyone's database today. §7 lists what actually runs.

The design is a gateway that lives in _your_ project, where the enforcement is SQL you review before it runs.

One migration installs a namespaced schema. Inside it:

**A role that cannot do the dangerous thing.** The writing role is `NOLOGIN`, `NOBYPASSRLS`, and is not the owner of your tables. Its grants are column-level `INSERT` on precisely the columns named in the contract — not table-level, column-level. It has no general `SELECT` on your destination table. Under the two accepted schemas it cannot read back the IDs of the rows it inserted, which is why one of them reports counts instead of IDs.

**A write path that is a fixed list, not a language.** Each published contract compiles to a function whose SQL is static — generated once at publish time, with an explicit column list, a typed decode of the incoming rows, a pinned empty `search_path`, and a statement timeout. There is no runtime dynamic SQL anywhere. Nothing a caller sends can become a schema, table, column, or function name. The published artifact _is_ the SQL you review; if it changes, the hash it's pinned to stops matching.

**Your tenant column, filled from a place the spreadsheet can't reach.** System-bound columns are written from the job record, which was authorized before any row was parsed. And if the batch payload contains `tenant_id` at all — even carrying the correct value — the batch is rejected with an error rather than silently overwritten. A payload that includes a system-bound column is either a bug on my side or an attack on yours, and both of those deserve to stop rather than proceed correctly by accident.

Underneath that, a restrictive RLS policy on your table re-checks the tenant value against a transaction-local setting the function assigns from the job record after entry. Anything a caller set beforehand is overwritten and never read. Four independent layers have to agree before a row lands, and a forged `workspace_id` from a browser devtools console has no entry point into any of them.

**Two signatures, and neither one is enough.** Your backend authenticates the user and signs a short-lived assertion — tenant, subject, purpose, environment, mode — with a private key I never see. I independently sign a job envelope binding that assertion's digest plus the contract version and hash. The gateway verifies both and requires them to agree exactly.

The point of that is the worst case. If my signing key and my control plane are both compromised, the attacker still cannot authorize a new job against your database, because your backend's signature is missing and I can't mint it. That's the property I traded the schedule for.

**Retries that don't duplicate rows.** Each PostgREST request runs in one transaction, so the inserted rows and the ledger record commit together or not at all. The ledger is keyed on `(job_id, batch_seq)`, with a hash of the batch body stored beside it. Resend the identical batch and you get the stored outcome back with zero additional writes. Resend a _different_ body under the same key and it fails with a mismatch rather than writing anything.

And when the ledger is unreachable, the job stays in `outcome_unknown` until reconciliation succeeds. Not "probably fine." Not an optimistic retry. Unknown is a terminal state I model on purpose, because the alternative is telling you an import succeeded when I don't know that.

**Triggers, honestly.** I can read `pg_trigger` and tell you the timing, the function, its owner, its security mode, and a hash of its definition so I notice when it changes. I cannot tell you what `notify_new_member()` _means_. If it calls `net.http_post` to your notification service, catalog introspection sees an HTTP call; it does not see 4,000 emails. So classification is your job, and an effect nobody has classified blocks production activation rather than proceeding.

This is also where I'll tell you the limit of the whole design: once a batch commits and an `AFTER INSERT` trigger has fired an outbound call, rolling back rows does not recall the email. There is no rollback of a side effect that already left the building — not mine, not anyone's. What I can do is tell you exactly what landed, and refuse to run until you've acknowledged what will fire.

## 5. What I hold, and what still has to live in your project

I hold one thing: my own signing key. It signs my half of the job authorization. It cannot decrypt anything, cannot connect to anything, and is useless without your backend's signature.

Here is the part a security review would find on its own, so I'd rather say it first.

The planned gateway component in your project calls the database through PostgREST, and it authenticates with a `service_role`-class secret key — because, as in §1, Supabase keys cannot be bound to a custom role. That key exists **only in your project's environment**. I never see it, never store it, and it never appears in my logs. But it exists, and honesty requires naming what it means: the fixed router and the `SECURITY DEFINER` boundary are load-bearing. They're what makes a full-power key behave like a narrow one — the caller can only reach a fixed set of operations, and the effective privileges inside them are the constrained role with your RLS applied.

If someone compromises your project's environment and steals that key, they have your project. That is true whether or not I exist, and it is not a threat this design claims to solve.

## 6. Where this is worse

Four places, plainly:

**It's slower to set up.** The rejected design is a paste-a-key field. Mine is a migration you read, a component you deploy, a signing key you generate, and a schema review. Today that happens with me on a call, by hand, because self-serve installation is planned and not built. I'm not going to publish a setup-time number until I've measured one on a real partner.

**It's Supabase-only.** The developer-endpoint design works against any destination on day one. Mine compiles to Supabase-specific authority. Other Postgres destinations are a later roadmap item that would need their own decision, not a config flag.

**It's more of my code between you and correctness.** A generator that emits SQL is a bigger thing to be wrong about than an HTTP POST. I've responded by having every milestone independently reviewed adversarially and keeping the findings in the repository, including the ones against me — but the surface is genuinely larger, and a reviewer who prefers less vendor machinery is not being unreasonable.

**It bought me nothing on the day I shipped, because I haven't shipped.** That's the honest cost.

## 7. What actually exists today

Present tense, and nothing beyond it.

What runs today: deterministic canonical JSON and hashing, Supabase schema fingerprinting, Postgres catalog observation, and a contract compiler that takes one provenance-bound single-table insert-only input and returns either an inert plan — a description of what _would_ happen — or a refusal with a stated reason. If your table shape isn't supported, it says so and names why, instead of guessing.

**The part that emits SQL is deliberately switched off.** Not unfinished — gated, behind a review that hasn't happened yet. No SQL bytes are generated by this codebase right now, on purpose.

Also not built, all planned: the gateway component, the installable migration, the key registry, the embeddable widget, self-serve setup, and any automated write into a customer's database.

What the design has behind it is two spikes against a disposable hosted Supabase project, plus an independent audit that reproduced the results. It found no critical issue, and no high-severity issue remains open for the tested architecture and scope — one was found earlier and closed, which is the whole reason the second signature exists. The two-signature mechanism was exercised there: missing anchors, mismatches, expiry, nonce conflicts, revoked keys, and spreadsheet-supplied tenant overrides all failed before anything mutated. That is a scoped architecture decision on evidence — not a production readiness claim, and not a customer deployment. Every open medium-severity finding is written down in the repository, and accepting the decision closed none of them.

And the free compatibility check — paste your `CREATE TABLE`, get back which columns a customer's file can safely populate and which must never be user-supplied — needs no credentials at all. Not "we encrypt them." None. It reads a text file you paste.

## 8. The question worth asking any vendor

Not "do you take security seriously." Everyone says yes.

Ask which architecture they rejected, and what rejecting it cost them. A vendor who chose the easy one will tell you about their encryption. A vendor who chose the hard one will have a document, a date, and a list of things they gave up.

Mine is ADR-001. I'll send you the whole thing, including the parts where the answer is still open.
