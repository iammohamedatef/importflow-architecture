# ADR-001: Why I Won't Hold Your `service_role` Key

**Status:** Accepted — architecture decision for Founder-Assisted Alpha scope
**Scope:** Supabase integration, Founder-Assisted Alpha (the current phase, in which I personally set up and review each customer integration rather than customers self-provisioning)
**Date:** <!-- fill in -->
**Author:** <!-- fill in -->

> This is not a claim of current production execution. Unbuilt mechanisms described below are planned or conditional; §7 states what exists today.

**TL;DR:** ImportFlow's default Supabase integration will not depend on holding a standing `service_role`-class customer credential. §§1–3 explain what I rejected and why. §§4–6 describe the selected architecture — a design, not yet built. §7 is the only section written in present tense, and it's the one to read first if you're deciding whether to approve this integration today.

This document records the architecture decision for the person who would have to approve the integration.

## 1. What that key actually is

A Supabase secret API key (`sb_secret_…`) maps to the `service_role` Postgres role. That role carries `BYPASSRLS`.

That distinction matters when evaluating a vendor integration.

Suppose your application has tenant isolation like this:

```sql
alter table public.documents enable row level security;

create policy tenant_isolation on public.documents
  for all
  using (tenant_id = (auth.jwt() ->> 'org_id')::uuid);
```

With the relevant Data API grants, the role's operations are not constrained by tenant RLS policies.

So a `service_role`-class credential should not be described as equivalent to access constrained by the application's normal tenant RLS boundary.

Two other platform facts shape the decision.

Supabase allows multiple secret API keys and allows them to be revoked individually. That materially improves operational revocation compared with relying on a single long-lived key.

But Supabase secret API keys cannot currently be bound to an arbitrary custom Postgres role. A project cannot, for example, turn an `sb_secret_…` key into a credential whose database authority is inherently limited to `INSERT` on one importer table.

The simplest vendor architecture, therefore, is straightforward: the customer provides a `service_role`-class credential and the vendor's infrastructure uses it to perform destination operations.

That architecture may be operationally convenient, but its database authority is broader than the importer contract I want ImportFlow to depend on.

That is the boundary I decided not to make part of ImportFlow's default Supabase architecture.

## 2. The architecture I rejected

The rejected design is simple:

You would paste an `sb_secret_…` key into a settings field. I would store it encrypted. When one of your users uploaded a spreadsheet, my runtime would validate it, call your project's Data API with that credential, and insert the accepted rows.

If a request timed out, the system would reconcile the outcome before deciding whether retrying was safe.

That design would eliminate a substantial amount of architecture:

* I would not need a customer-reviewed generated migration for the destination authority.
* I would not need a dedicated database role and contract-specific grants.
* I would not need a customer-hosted gateway component.
* I would not need the planned dual-anchor authorization mechanism.
* I would not need the planned customer key registry and its rotation/revocation lifecycle.
* I would not need an in-database idempotency ledger for the selected architecture.
* I would not need the same contract-specific publication gate.
* Revocation would primarily mean revoking the credential rather than removing installed gateway authority.

I will not attach a fabricated schedule or engineering-cost estimate to that decision. The important point is architectural: the simpler design would remove several of the mechanisms required by the selected model.

I rejected it because I do not want ImportFlow's default Supabase integration to depend on ImportFlow retaining a standing `service_role`-class customer credential.

## 3. The alternatives that were actually hard to turn down

Two alternatives remain technically credible. Rejecting them as the default does not make them bad architectures.

### A scoped Postgres login

Instead of a Supabase secret API key, a customer could create a dedicated Postgres login whose grants are restricted to the operations required by one importer and provide that connection information to ImportFlow.

That can provide genuine database-level least privilege.

I rejected it as the default Supabase architecture for three reasons.

1. The credential would still leave the customer's environment and become something ImportFlow would have to store and operate securely.

2. Provisioning would not be equivalent to retrieving an existing database password through the Supabase Management API. The Management API does not provide a mechanism for retrieving the project's existing database password. Any password-based direct-connect design would therefore require a separate customer-controlled provisioning and rotation flow rather than assuming ImportFlow could obtain the existing password automatically.

3. The resulting security-review statement would still be that a third-party service holds a login capable of connecting to the production database, even if that login has much narrower privileges than `service_role`.

An A2-style scoped-login model remains a possible future design for self-hosted or raw-Postgres environments. It is not part of the accepted Founder-Assisted Alpha Supabase path.

### Your own endpoint

A stronger containment model would be for ImportFlow never to receive database write authority at all.

ImportFlow could validate, map, and correct a dataset and then deliver signed, idempotency-keyed batches to an HTTP endpoint operated entirely by the customer. Customer code would perform the database mutation.

That architecture has a very strong property: the customer's destination credentials and database write authority never need to enter ImportFlow's environment.

I did not select it as the Alpha default because it would return several hard correctness responsibilities to the integrating developer.

The customer implementation would need to define and correctly operate:

* idempotency state;
* replay handling;
* tenant/system-field enforcement;
* durable outcome reconciliation;
* partial-result semantics;
* error reporting precise enough for the importer workflow.

That is a valid architecture for organizations that explicitly prefer customer-owned execution. It is retained as a possible future delivery model, but no generic developer-endpoint implementation is part of the accepted Alpha architecture.

### A generated adapter running in your project

Another option would be to generate a TypeScript destination adapter that the customer deploys inside its own Supabase project.

The adapter could hold project-local credentials and perform writes without those credentials ever entering ImportFlow infrastructure.

The problem is where the transactional and authorization boundary would live.

A sequence of independent Data API calls from JavaScript would not by itself provide one database transaction covering both the destination write and the durable idempotency record. The design would therefore need a database-side transactional primitive, or a direct database connection with equivalent transactional semantics.

Once the database becomes responsible for the authoritative transaction and contract enforcement, a smaller fixed gateway in front of contract-specific database operations becomes preferable to placing the full enforcement model in regenerated JavaScript.

That is the basis for the selected design.

## 4. What I chose instead — planned architecture, not current execution

**Read this section as architecture, not as a description of currently deployed product behavior.**

The selected Founder-Assisted Alpha design would place a narrow gateway inside the customer's Supabase project, with the database authority expressed through SQL that the customer can review before installation or publication.

The design requires one reviewed migration to install a namespaced gateway schema.

Inside it:

**A constrained execution role.** The design requires a `NOLOGIN`, `NOBYPASSRLS` role that would not own the customer's destination tables.

For an accepted importer contract, the role would receive only the grants required by that contract. The planned Alpha variants would avoid unrestricted destination reads, and Variant A would not require destination-ID reads.

**A fixed write surface rather than a runtime query language.** Each published contract is planned to compile into contract-specific database operations with static SQL and explicit destination columns.

Runtime input would not be allowed to become a schema name, table name, column name, function name, or arbitrary SQL fragment.

The design requires generated functions to use hardened `SECURITY DEFINER` practices, including explicit object qualification and a pinned `search_path`.

**System-bound fields would come from previously authorized job state.** Tenant identifiers and other system-owned values would be planned to come from the durable job authorization record rather than from spreadsheet input.

The design requires a batch containing a system-bound column to be rejected rather than silently trusting or overwriting that value.

A restrictive RLS policy is planned as defense in depth. The planned execution function would set transaction-local tenant context from previously authorized job state before performing the contract operation.

**Two independent authorization anchors would be required.** The customer backend is planned to authenticate its user and issue a short-lived assertion covering claims such as tenant, subject, purpose, environment, and mode.

ImportFlow is planned to issue a separate job envelope binding the customer assertion to the specific contract and job.

The customer-hosted gateway would require both authorization anchors to validate and agree. The design therefore would not treat ImportFlow's signature alone as sufficient authority to create a new destination job.

**Retries would be tied to durable idempotency state.** The design requires the destination batch mutation and its idempotency ledger entry to occur within the same database transaction.

A repeated `(job_id, batch_seq)` carrying the same batch hash would be planned to resolve to the previously stored outcome rather than perform a second mutation.

The same idempotency identity carrying a different body would be rejected.

If durable destination state could not be determined, the planned product state would remain `outcome_unknown` until reconciliation established what happened.

**Trigger and side-effect handling would be explicit.** The compatibility design is planned to inspect database metadata that can identify structural facts about triggers and functions.

That metadata would not be treated as proof of business meaning.

The design therefore requires customer acknowledgement for side effects that can be classified and would block production publication when effects remain unknown or unsupported.

It would also make no universal rollback claim for external effects. If a committed database trigger caused an irreversible external action, removing the inserted row afterward would not undo that external action.

## 5. What ImportFlow would hold, and what would remain in the customer project

Under the selected architecture, ImportFlow is planned to hold its own signing authority for its side of job authorization.

That signing authority would not itself be a customer database credential.

The planned customer-hosted gateway would still need a project-local mechanism for calling the project's fixed database RPC surface.

For the accepted design, that mechanism is planned to use a dedicated project-local `service_role`-class secret available only inside the customer's Supabase project environment.

ImportFlow would not be designed to receive or retain that secret.

Because the project-local key itself would still map to `service_role`, the design requires the gateway not to behave as a generic proxy.

The fixed router and the `SECURITY DEFINER` boundary would therefore be load-bearing security boundaries: caller-controlled data would be limited to the fixed accepted operation shape, while the database-side contract function would execute using the deliberately constrained role and contract-specific authority.

As established in §1, that access is not constrained by tenant RLS policies. The planned design therefore would not rely on the project-local secret's own role scope for least privilege; it would rely on keeping that credential inside the customer project and placing the effective database operation behind the fixed gateway and constrained contract execution boundary.

Compromise of the customer's own project environment and theft of a project-level secret would remain outside the problem this architecture claims to solve.

## 6. Where the selected design would be worse

There are real costs.

**Setup would be more involved.** A direct credential model could be implemented as a simple credential handoff.

The selected Alpha design would require a reviewed migration, a customer-hosted gateway artifact, customer-side authorization setup, contract review, and developer-controlled publication.

Founder-Assisted Alpha is planned to perform those steps with direct founder involvement. Self-serve provisioning is not currently implemented.

I'm not attaching a schedule or engineering-cost estimate to that setup cost, since neither has been measured yet against real partner installations.

**The accepted architecture would initially be Supabase-specific.** A developer-owned HTTP destination can target almost anything.

The selected model compiles authority around Supabase/Postgres semantics. Raw Postgres, other Postgres environments, and non-Postgres destinations would require later architecture decisions rather than being treated as configuration switches.

**The implementation surface would be larger.** A contract compiler, migration generator, gateway, ledger model, authorization protocol, publication state, and reconciliation system create more software that can contain defects than a simple outbound HTTP request.

That additional machinery would therefore require corresponding tests, security review, artifact review, and operational evidence before execution could be accepted.

**The architecture decision does not mean the execution system exists today.** Acceptance of ADR-001 means the architecture has been selected for the stated Founder-Assisted Alpha scope. It does not mean the planned production execution path has already been implemented.

## 7. What actually exists today

Present tense applies only in this section.

The current implementation includes deterministic canonical JSON and hashing, Supabase schema fingerprinting, Postgres catalog observation, and contract compilation for the supported provenance-bound, single-table, insert-only model.

The compiler can analyze a supported input and produce an inert description of the accepted contract plan or refuse the input with a stated compatibility reason.

**The current compiler produces no SQL and exposes no executable artifact path.**

There is currently no generated migration that can be installed to execute destination writes, no shipping customer-hosted gateway, no production destination execution path, no production job-signing execution flow, and no generated SQL execution surface.

Any descriptions in §§4–6 of generated SQL, gateway verification, signatures, destination writes, ledgers, publication, retries, or production activation describe the accepted architecture and requirements for future implementation. They are not descriptions of behavior currently exposed by the product.

The architecture has supporting spike and audit evidence from disposable test environments. That evidence informs the decision and its gates, but it is not a customer deployment or a claim that the production execution system currently exists.

The Compatibility Check is currently manually delivered.

A customer can provide schema material for review without providing a destination credential, and the current process can identify supported and unsupported field ownership or destination-shape conditions. The self-serve compatibility-check product experience is not yet implemented.

That is the current boundary: architecture accepted for Founder-Assisted Alpha; execution path not yet built.

## 8. The question worth asking any vendor

A useful security question is not simply whether a vendor says it takes security seriously.

Ask what authority the integration requires, where that authority lives, what mechanism constrains it, what happens when the mechanism fails, and which simpler architecture the vendor considered and rejected.

For ImportFlow, that decision is ADR-001.

It records both the architecture selected for Founder-Assisted Alpha and the limits of what that acceptance means today.
