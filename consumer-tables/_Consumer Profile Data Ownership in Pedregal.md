

# Consumer Profile Data Ownership in Pedregal

# Background

In the current system, the Consumer Profile Service (COPS) is owned and managed by the Identity Domain. Within the Pedregal framework, however, it is recognized that consumer profile data should no longer reside under the Identity Domain. Consequently, it is necessary to identify and assign appropriate ownership for consumer profile data currently stored in the Consumer database tables, aligning data governance more closely with the relevant domains based on data usage and context.

# Proposal

**Objective:** Deprecate the existing COPS Consumer Profile Service and clearly assign ownership of consumer profile data, currently residing in the Consumer database tables, to appropriate domains based on data relevance.

## Domain Ownership Table

| Field Name | Data Type | Active? | Proposed Domain Owner |
| ----- | ----- | ----- | ----- |
| id | INT8 | ✅ | Consumer |
| user\_id | INT8 | ✅ | Identity |
| stripe\_id | VARCHAR | ❌ | Bank |
| default\_address\_id | INT8 | ✅ | Geo |
| created\_at | TIMESTAMPTZ | ✅ | Consumer |
| first\_week | TIMESTAMPTZ | ❌ | Consumer |
| account\_credits\_deprecated | INT8 | ❌ | Money |
| receive\_text\_notifications | BOOL | ❌ | Consumer |
| default\_card\_id | INT8 | ❌ | Money |
| applied\_new\_user\_credits | BOOL | ❌ | Money |
| receive\_push\_notifications | BOOL | ❌ | Platform |
| channel | VARCHAR | ❌ | Consumer |
| referral\_code | VARCHAR | ✅ | Consumer |
| stripe\_country\_id | INT8 | ❌ | Money |
| default\_country\_id | INT8 | ✅ | Consumer |
| default\_substitution\_preference | VARCHAR | ❌ | Consumer |
| sanitized\_email | VARCHAR | ✅ | Consumer |
| last\_delivery\_time | TIMESTAMPTZ | ❌ | Consumer |
| gcm\_id | STRING | ❌ | Consumer |
| android\_version | INT8 | ❌ | Consumer |
| is\_vip | BOOL | ✅ | Consumer |
| referrer\_code | STRING | ❌ | Consumer |
| came\_from\_group\_signup | BOOL | ❌ | Consumer |
| catering\_contact\_email | VARCHAR | ✅ | Catering? NV? |
| receive\_marketing\_push\_notifications\_old | BOOL | ❌ | Consumer |
| last\_alcohol\_delivery\_time | TIMESTAMPTZ | ❌ | Consumer |
| default\_payment\_method | STRING | ❌ | Bank |
| delivery\_customer\_pii\_id | INT8 | ✅ | Consumer |
| fcm\_id | STRING | ❌ | Platform (notifications) |
| vip\_tier | INT8 | ✅ | Consumer |
| existing\_card\_found\_at | TIMESTAMPTZ | ❌ | Payments |
| existing\_phone\_found\_at | TIMESTAMPTZ | ❌ | Consumer |
| updated\_at | TIMESTAMPTZ | ✅ (Write only) | Consumer |
| forgotten\_at | TIMESTAMPTZ | ❌ | Identity |
| experience | STRING | ✅ | Consumer |
| tenant\_id | VARCHAR | ✅ | Identity |
| receive\_marketing\_push\_notifications | BOOL | ❌ | Platform (notifications) |

## Identity Mapping Layer (IML)

To support the transition from COPS, the Identity domain will implement an Identity Mapping Layer (IML). The IML will provide a centralized mechanism to translate between global `account_id` and domain-specific identifiers, including `consumer_id`. This design will significantly reduce inter-service dependencies and enhance system performance. Each domain team will be required to own their respective domain-specific identifier tables and register these with the IML. Detailed information about the IML design can be found in the [\[RFC\] Domain ID Handling in Pedregal](https://docs.google.com/document/d/1Q4hdshz2xHk7SGS94cm8ZcxH_DDXxKEOrmPCGwsmsic/edit?tab=t.0#heading=h.8fci0bzb9bmr).

