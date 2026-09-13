# Architecture decisions

An ADR records an architecturally significant decision, its context, considered alternatives and consequences. Acceptance applies to the stated scope. Implementation status is recorded separately, so a selected design is not mistaken for deployed behavior.

| ID      | Decision                                                                    | Status                                           | Purpose                                                                                                                             | Public record                                                                |
| ------- | --------------------------------------------------------------------------- | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| ADR-001 | [Why I Won’t Hold Your `service_role` Key](adr-001-service-role-custody.md) | Accepted for Founder-Assisted Alpha architecture | Keep customer production credentials out of ImportFlow custody; explain the selected customer-hosted authority model and its costs. | Published September 1, 2026; current-state clarification September 13, 2026. |

Publication dates come from repository history; they are not substituted for an unrecorded decision date. Changes to availability notes preserve the accepted rationale. A later decision that supersedes an accepted choice should link to the earlier record and explain the change.

See the [current data and authority flow](overview.md) and [repository scope](../README.md).
