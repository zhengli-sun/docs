# 

| Author(s):  | [Gandhi Amarnadh Tadiparthi](mailto:gandhi.tadiparthi@doordash.com)[Jiachen Huang](mailto:jiachen.huang@doordash.com) |
| :---- | :---- |
| **Status:** | Closed  |
| **References:** | **[COPS Pedregal Ownership Segmentation](https://docs.google.com/document/d/139x4S-Y31X-1Sc3Pjf8-KfHKMDwZfeB_ipDznxBy2vc/edit?tab=t.0) [ Consumer Profile Data Ownership in Pedregal](https://docs.google.com/document/d/1C7KMQ-PFAP9M46gtR1CLvHJh1Jq5scSm4lz3IW6-KG0/edit?tab=t.0) [RFC - Pedregal Consumer Lifecycle](https://docs.google.com/document/d/1Yas_pySva8jfRXH2WhQdi37msYqbbZ3n7SHR356R3JM/edit?tab=t.qxjii4ymnba5)[\[RFC\] Domain ID Handling in Pedregal](https://docs.google.com/document/d/1Q4hdshz2xHk7SGS94cm8ZcxH_DDXxKEOrmPCGwsmsic/edit?pli=1&tab=t.0#heading=h.8fci0bzb9bmr)** |

| Reviewer | Area | Status | Notes |
| :---- | :---- | :---- | :---- |
| [Ankit Agarwal](mailto:a.agarwal@doordash.com) | Storefront | Reviewed/Approved | Some fields like catering\_contact\_email are used as of today. We need a replacement either from Storefront or others until then we cannot deprecate existing support from Cops. [Colin Bartels](mailto:colin.bartels@doordash.com)Is looking into this from Storefront side |
| [Jordan Blumenthal](mailto:jordan.blumenthal@doordash.com) | Drive | In progress |  |
| [Qi Guo](mailto:qi.guo@doordash.com)[Sahana Madhusudan Achar](mailto:sahana.m@doordash.com) | COPS | Reviewed/Approved |  |
| [\[Login-X\]Kevin Chen](mailto:kevin.chen@doordash.com) | Identity Experience | Reviewed/Approved |  |
| [Leon Liang](mailto:leon.liang@doordash.com) |  | In progress |  |
| [Henri Tunkkari](mailto:henri.tunkkari@wolt.com)[Jukka Ala-Fossi](mailto:jukka.ala.fossi@wolt.com) | Wolt Consumer | Reviewed/Approved |  |
| [Paipo Tang](mailto:paipo.tang@wolt.com) | Wolt SF | Reviewed/Approved |  |

## **Introduction**

Pedregal Consumer defines the minimal representation of a consumer in Pedregal, owned by Consumer Growth. This RFC only refers to the Consumer Entity and its migration. Consumer Profile is no longer supported and is being handed off to individual domain owners ( Refer to **[COPS Pedregal Ownership Segmentation](https://docs.google.com/document/d/139x4S-Y31X-1Sc3Pjf8-KfHKMDwZfeB_ipDznxBy2vc/edit?tab=t.0))**

It will replace both:

* DoorDash COPS (Consumer Profile Service, owned by Identity Platform), which is feature-frozen and scheduled for deprecation by EOH1 2026\.  
* Wolt Consumer, which today serves as the source of truth for Wolt consumers but will be retired as part of the Pedregal migration.

Key points:

* **Consumer Entity**: Pedregal Consumer is a minimal entity where each consumer has a consumer\_id (legacy preserved, UUIDv5 for Pedregal-native) tied to an actor\_id (account\_id for registered users, guest\_id for guests). It models only the relationship to Account/Guest and persona, while all other profile attributes are owned by other Pedregal sub domains (Geo, Storefront, DDFB etc..)  
* **Personas**: Only two personas are supported — default and caviar. All other legacy COPS experiences (Drive, Storefront, dd\_pos, third\_party, self\_kiosk, white\_labeled) map into default. Wolt does not have the concept of experiences and will be mapped to default.  
* **Guests**: Guests are not differentiated in Pedregal Consumer. Guest identity resides in the Identity as subdomain. Legacy guest consumers will be migrated after real users, aligned with Identity’s guest\_id model.  
* **APIs**: Pedregal Consumer provides minimal lifecycle APIs (get, list, create). No profile attributes, guest-specific, or domain-specific APIs are included and thus no update API

## **Pedregal Consumer**

Pedregal Consumer is a lightweight entity that consolidates consumer lifecycle across DoorDash and Wolt into Pedregal. It holds the relation to the Account/Guest based on the actor\_id. Supposed to be a relationship or mapping layer with little to no information about Consumer.

Responsibilities:

* Unify consumer lifecycle for both DoorDash and Wolt under Pedregal.  
* Preserve existing consumer identifiers (consumer\_id) to maintain backward compatibility.  
* Generate new Pedregal-native identifiers for new users going forward.  
* Support limited personas (default, caviar).

Not in scope: managing address, email, delivery\_customer\_pii\_id, or [other attributes](?tab=t.uzc99ibkrjzk#heading=h.xwmxp0ocv7f3) currently held in COPS/Wolt. These are delegated to other domains (e.g., Drive, Storefront, Geo).

No RTF support needed as no PII is being mapped here and Identity Account RTF strategy obsoletes the need for RTF at domain level

**Consumer Entity Schema**

Because Taulu is query-first and does not support secondary indexes yet, joins, or cross-table transactions, the Pedregal Consumer schema is implemented using two tables:

* consumers\_by\_id → keyed by consumer\_id, optimized for legacy compatibility (getConsumerById, listConsumersByIds).  
* consumers\_by\_actor\_id → keyed by (actor\_id, persona), optimized for account/guest-scoped queries (listConsumersByAccountId, getConsumer).

All writes must update both tables with idempotent upserts to maintain consistency. This ensures Pedregal Consumer supports both legacy and Pedregal-native access patterns while staying within Taulu’s constraints.

Consumer table 

| consumer\_id (String) | actor\_id | created\_at | persona |
| :---- | :---- | :---- | :---- |
| Partition |  |  | Caviar | default |

Query Supported:

* getConsumerById(consumer\_id)  
* listConsumersByIds(List\<consumer\_id\>)

consumer\_by\_actor\_id table 

| actor\_id | persona | consumer\_id |
| :---- | :---- | :---- |
| partition | sort |  |

Query Supported:

* listConsumersByActorId(actor\_id)  
* getConsumerByIdActorAndPersona(actor\_id, persona)

### **Pedregal Consumer Persona**

Pedregal Consumer introduces a simplified persona model to represent different consumer contexts while avoiding the complexity of legacy “experiences” in COPS. (Wolt does not have an “experience” concept.) An authenticated user/account can seamlessly be allowed to switch consumer persona/experience on the same client. Persona track account as a pseudo consumer (split on everything relying on consumerId)

Supported Personas

* default  
  * Represents all standard consumer identities across DoorDash and Wolt.  
  * Also covers legacy consumers from experiences that do not map to a distinct persona (e.g., Storefront, Drive, Hippa, etc..).  
* caviar  
  * Represents consumers in the Caviar context.  
  * Allows the same account to have a distinct consumer role in Caviar, separate from their default consumer role.

Key Principles

* An actor\_id (either an account\_id for registered users or a guest\_id for guests) may have multiple consumer personas.  
* Each persona is modeled as a separate consumer record, keyed by (actor\_id, persona).  
* Legacy experiences (Storefront, Drive, Hippa, etc) are not modeled as separate personas.   
* Consumers created before Caviar acquisition are represented now with experience \= null, these will be migrated as default persona as well.

Persona selection is driven by IDX\* (Identity Experience) based on Identity Client Configuration. Persona switching will not be supported now but can be supported by IDX \+ IML.

### **Pedregal Consumer Interfaces** {#pedregal-consumer-interfaces}

Pedregal Consumer exposes a minimal set of APIs to manage consumer lifecycle. These interfaces are limited to consumer entity operations and do not handle profile attributes (which belong to other domains).

### **Read APIs**

* getConsumerById(consumer\_id)  
  * Returns a single Consumer entity by consumer\_id.

| message getConsumerByIdRequest {  // Unique identifier for the consumer  string consumer\_id \= 1; } rpc getConsumerById(getConsumerByIdRequest) returns Consumer |
| :---- |

* listConsumersByIds(List\<consumer\_id\>)  
  * Batch fetch by consumer IDs.

| message listConsumersByIdsRequest {  // Unique identifier for the consumer repeated  string consumer\_ids \= 1; } rpc listConsumersByIds(listConsumersByIdsRequest) returns List\<Consumer\> |
| :---- |

* listConsumersByActorId(actor\_id)  
  * Returns all consumers tied to the same account/guest (e.g., default, caviar).

| message listConsumersByActorIdRequest {  // Unique identifier for the account/guest  string actor\_id \= 1; } rpc listConsumersByActorId(listConsumersByActorIdRequest) returns List\<Consumer\> |
| :---- |

* getConsumerByIdActorAndPersona(actor\_id, persona=default)  
  * Returns the consumer record for a specific actor\_id and persona. If persona is not provided

| message getConsumerByIdActorAndPersonaRequest {  // Unique identifier for the consumer string actor\_id \= 1;  string persona \= 2; } rpc getConsumerByIdActorAndPersona(getConsumerByIdActorAndPersonaRequest) returns Consumer |
| :---- |

### 

### **Write APIs**

* getOrCreateAuthenticatedConsumer(actor\_id, persona) → Returns the Consumer for the given persona, creating one if it does not exist. This will be called by IDX\* during signup/login if IML data is non existent for the actor\_id.

| message getOrCreateAuthenticatedConsumerRequest {  // Unique identifier for the consumer string actor\_id \= 1;  string persona \= 2; } rpc getOrCreateAuthenticatedConsumer(getOrCreateAuthenticatedConsumerRequest) returns Consumer |
| :---- |

![][image1]

Consumer Object Structure

All APIs return the Consumer object:

| syntax \= "proto3";package pedregal.consumer.v1;message Consumer {  // Unique identifier for the consumer  string consumer\_id \= 1;  // Consumer persona type (default, caviar)  enum Persona {     UNKNOWN \= 0;    DEFAULT \= 1;    CAVIAR \= 2;  }  Persona persona \= 2;   // Timestamp when the consumer record was created  string created\_at \= 3; // ISO-8601 format   // Either account\_id (for registered users) or guest\_id (for guests)  string actor\_id \= 4; } |
| :---- |
|  |

### **Guest related APIs**

In Pedregal, the guest concept is a subdomain of Identity. Pedregal Consumer does not differentiate “guest” consumers from registered consumers.

* All Consumer APIs operate on actor\_id, which may be either an account\_id (for registered users) or a guest\_id (for guests).  
* The resolution of whether an actor\_id represents a guest or an account is handled by Identity API ( GetAccountOrGuest ), not by Pedregal Consumer.  
* Therefore, Pedregal Consumer does not expose any dedicated Guest APIs.  
* [Pedregal Guest proposal](https://docs.google.com/document/d/10YkCO3YyxtrZkRUbTgnhR5z3-wrGQaKcjfuU9O47y5w/edit?tab=t.0#heading=h.8jlzy09dyxly)\* covers Guest lifecycle, Allowing Consumer to deprecate  
  * createLiteGuestConsumer  
  * getLiteGuestConsumer  
  * createFullGuestConsumer  
  * getFullGuestConsumer  
  * syncExternalConsumer  
* Based on [Pedregal Guest proposal](https://docs.google.com/document/d/10YkCO3YyxtrZkRUbTgnhR5z3-wrGQaKcjfuU9O47y5w/edit?tab=t.0#heading=h.8jlzy09dyxly)\* a guestPromotion API would be implemented by Consumer (called by UG) and this would be responsible for downstream Guest Consumer promotion to Authenticated Consumer as well, while deprecating  
  * convertToFullGuestConsumer  
  * convertGuestToAuthenticatedConsumer

### **API Principles**

* Backward compatibility: Legacy consumer\_id values are preserved and queryable.  
* Minimal scope: APIs only cover consumer lifecycle; profile attributes are explicitly out of scope.  
* No legacy proxying: Pedregal Consumer APIs do not proxy to COPS or Wolt; if upstream services need legacy-specific attributes, they must query those systems directly during migration.

## **Pedregal Guest Consumer**

In Pedregal, the concept of guest identity resides in the Identity domain, not in Pedregal Consumer.

* Pedregal Consumer does not differentiate guest vs. registered users. All are represented through actor\_id.  
* TBD on whether Guest ID can share more than one persona.  
* No dedicated Guest APIs are required — existing Consumer APIs will work once guest\_id resolution is available.

## **Doordash COPS**

Segment profile attributes that it holds based on [segmentation plan](https://docs.google.com/document/d/139x4S-Y31X-1Sc3Pjf8-KfHKMDwZfeB_ipDznxBy2vc/edit?tab=t.0)..  
Consumer domain-specific attributes will be owned and managed by their respective domains ([Consumer attributes migration decision](?tab=t.uzc99ibkrjzk)). Terms of Service will be deprecated; the consumer\_tos entity and its related APIs will be removed. 

### **Legacy Consumer Experience(DoorDash \+ Wolt)**

**Principle:**

* Only two personas exist in Pedregal → default and caviar.  
* All COPS experiences except caviar collapse into default.  
* Wolt does not have “experience” concept and will be migrated as default persona.  
* null values migrate as **default** persona.  
* Legacy Experience usage should be migrated for Account/Guest\* Namespace


| Legacy Source | Legacy Experience | User Type | Pedregal Persona | Account NS | Guest\* NS |
| :---- | :---- | :---- | :---- | :---- | :---- |
| COPS | null | Guest \+ Account | default | Doordash | Marketplace |
| COPS | doordash | Guest \+ Account | default | Doordash | Marketplace  |
| COPS | caviar | Guest \+ Account | caviar | Doordash | Marketplace  |
| COPS | drive | Guest | default | NA | Drive  |
| COPSWolt ? | Storefront\- | Guest \+ AccountGuest? \+ Account | default | SF (merged)? | TBD-SF/Marketplace (merge)? |
| COPS | dd\_pos | Guest | default | NA  | POS  |
| COPS | third\_party | Guest | default | NA | ThirdParty  |
| COPS | self\_kiosk | Guest | default | NA | Kiosk  |
| COPS | white\_labeled | Guest | default | NA | TBD Marketplace?  |
| COPS | hipaa | Guest | default | NA | Hipaa |
| Wolt | (no experience) | Account  | default | Wolt | NA |

[These Doordash COPS experiences will not be migrated and being deprecated](?tab=t.1h386gjlx1r0)

**Wolt Consumer**  
Consumer\_id is the same as Wolt user\_id per tenant. These tenants are being onboarded as Namespaces on accounts. All current Wolt users have a consumer profile at creation ( unsure about Wolt Guest)

## **Plan**

### [**Milestone 1 : Pedregal Consumer Setup**](?tab=t.g530dycyq6ni) 

Goals: Implement Consumer Graph in Pedregal & Taulu with backfill write API.

* Pedregal consumer\_id will be the same as DD COPS or Wolt Consumer ID stored as string   
* Backfill API Request: legacy user\_id, consumer\_id, experience, tenant\_id, dd\_or\_wolt identifier, created\_at. Logic to filter what to backfill is held here by querying Pedregal GetAccount ( GetAccountOrGuest\* ) 

### **Milestone 2: Async Write ([DD](?tab=t.jcdlz4twlyah), Wolt)**

Goals: Implement async write for both DD and Wolt

1) Marketplace Users  
2) Storefront Users

|  | legacy | pedregal |
| :---- | :---- | :---- |
| read | Primary | No |
| write | Origin | Async \- write |

### **Milestone 3: Realtime Backfill ([DD](?tab=t.jcdlz4twlyah), Wolt)**

Goal: Backfill the existing real consumer from DD and Wolt to Pedregal

Not all doordash users have consumer\_id. But all Wolt users have a consumer profile ( TBD Wolt SF Guest )

1. Marketplace Users  
2. Storefront Users

### [**Milestone 4: Pedregal Read**](?tab=t.g530dycyq6ni) **( ETA mid Q1 26\)**

Build [lookup](#pedregal-consumer-interfaces) interfaces for Pedregal Consumer. These will not proxy to legacy. From this milestone Segmentation plan in legacy Consumer can be kicked off.

|  | legacy | pedregal |
| :---- | :---- | :---- |
| read | Primary | Primary |
| write | Origin | Replica |

 

### **Milestone 5: DD COPS & Wolt Non Guest Consumer Segmentation strategy**

Goals: Coordinate with DoorDash and Wolt domain owners to segment and deprecate legacy attributes.

| Platform | Experiences | DRI | Notes |
| :---- | :---- | :---- | :---- |
| DD | null, doordash, caviar |  |  |
| DD | storefront |  |  |
| Wolt | \- |  |  |

### **Milestone 6: Guest consumer backfill**

Goal: Backfill the existing guest consumer from DD and Wolt to Pedregal

### **Milestone 7: DD COPS Guest Consumer Segmentation strategy**

Goals: Coordinate with DoorDash and Wolt domain owners to segment and deprecate legacy attributes.

| Platform | Experience | DRI | Notes |
| :---- | :---- | :---- | :---- |
| DD | null, doordash, caviar |  |  |
| DD | drive |  |  |
| DD | storefront |  |  |
| DD | dd\_pos |  |  |
| DD | third\_party |  |  |
| DD | self\_kiosk |  |  |
| DD | white\_labeled |  |  |
| DD | hipaa |  |  |
| Wolt (Storefront) | \- |  |  |

### **Milestone 8: Double Write**

Goal: Consumer created in DD COPS and Wolt will be replicated to Pedregal Consumer in real time

DD and Wolt Consumer double writes to pedregal. (TBD using the same backfill api). If Wolt consumers always exist for users, Consumer write can rely on Account create event for the Wolt namespace as well

|  | legacy | pedregal |
| :---- | :---- | :---- |
| read | Primary | No |
| write | Origin | Replica |

### **Milestone 9: Pedregal Write GA** 

Goal: Build interface getOrCreateAuthenticatedConsumer in Pedregal

The prerequisite for this step is legacy attributes be segmented/deprecated off fully and all legacy experience wise handling is decided. This point would take several months and not be controlled by this working group.

The write interface here generates a UUID for consumer\_id. Pedregal Consumer calls DD COPS and Wolt Consumer Write based on AccountId Namespace. Write GA can be allowed on special use cases with no support to replicate on Legacy

|  | legacy | pedregal |
| :---- | :---- | :---- |
| read | Proxy | Primary  |
| write | Replica | Origin |

### **Milestone 10: DD COPS and Wolt Consumer Cut off**

Goal: Deprecate Read to DD COPS and Wolt Consumer

|  | legacy | pedregal |
| :---- | :---- | :---- |
| read | No | Primary |
| write | No | Origin |

### 

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAzAAAAD/CAYAAAAuXjPnAABcF0lEQVR4Xu2dWewc1Zn25x5x4RuuuOKCCy64IMJCQkgICSHGshBCIISCQCBHyYggAw7BEatNgLAkKIEsrApkwshAFuADsQQwi4gngMJqBBNCmMEQdsw6bPV9v8N3mu7z9vL87XJ1VfXzk179+1/9dNXT/Z7qOm+fU1X/Uo3hf//3f1PMQtXB//zP/5SLAur66taB/UVUHdhfRNWB/UVUHdhfRNWB/UVUHdhfRNWB/UVUHdhfRNWB/UVUHTTt7z//8z+r//qv/6r+pXwClBWAqoOm3yCoOrC/iKoD+4uoOrC/iKoD+4uoOrC/iKoD+4uoOrC/iKoD+4uoOrC/iKqDpv25gCmwv4iqA/uLqDqwv4iqA/uLqDqwv4iqA/uLqDqwv4iqA/uLqDqwv4iqg6b9DQqYLB6OTz75JEW5vAxVR/z3f/93WFaGur66dYT9xVB1hP3FUHWE/cVQdYT9xVB1hP3FUHWE/cVQdYT9xVB1hP3FUHWE/cVQdUST/lzAjAn7i6HqCPuLoeoI+4uh6gj7i6HqCPuLoeoI+4uh6gj7i6HqCPuLoeoI+4uh6ogm/Y0UMMPDMplh4TRUHTQ9xASqDuwvourA/iKqDuwvourA/iKqDuwvourA/iKqDuwvourA/iKqDuwvouqgaX8uYArsL6LqwP4iqg7sL6LqwP4iqg7sL6LqwP4iqg7sL6LqwP4iqg7sL6LqoGl/LmAK7C+i6sD+IqoO7C+i6sD+IqoO7C+i6sD+IqoO7C+i6sD+IqoO7C+i6qBpfy5gCuwvourA/iKqDuwvourA/iKqDuwvourA/iKqDuwvourA/iKqDuwvouqgaX8uYArsL6LqwP4iqg7sL6LqwP4iqg7sL6LqwP4iqg7sL6LqwP4iqg7sL6LqoGl/LmAK7C+i6sD+IqoO7C+i6sD+IqoO7C+i6sD+IqoO7C+i6sD+IqoO7C+i6qBpfy5gCuwvourA/iKqDuwvourA/iKqDuwvourA/iKqDuwvourA/iKqDuwvouqgaX8uYArsL6LqwP4iqg7sL6LqwP4iqg7sL6LqwP4iqg7sL6LqwP4iqg7sL6LqoGl/LmAK7C+i6sD+IqoO7C+i6sD+IqoO7C+i6sD+IqoO7C+i6sD+IqoO7C+i6qBpf4MChg2XwV01iXJ5GaqO2Lx5c1hWhrq+unWE/cVQdYT9xVB1hP3FUHWE/cVQdYT9xVB1hP3FUHWE/cVQdYT9xVB1hP3FUHVEk/5GCpiisEkoFRCoOsgbnoa6vrp1YH8RVQf2F1F1YH8RVQf2F1F1YH8RVQf2F1F1YH8RVQf2F1F1YH8RVQdN+3MBU2B/EVUH9hdRdWB/EVUH9hdRdWB/EVUH9hdRdWB/EVUH9hdRdWB/EVUHTftzAVNgfxFVB/YXUXVgfxFVB/YXUXVgfxFVB/YXUXVgfxFVB/YXUXVgfxFVB037cwFTYH8RVQf2F1F1YH8RVQf2F1F1YH8RVQf2F1F1YH8RVQf2F1F1YH8RVQdN+3MBU2B/EVUH9hdRdWB/EVUH9hdRdWB/EVUH9hdRdWB/EVUH9hdRdWB/EVUHTftzAVNgfxFVB/YXUXVgfxFVB/YXUXVgfxFVB/YXUXVgfxFVB/YXUXVgfxFVB037cwFTYH8RVQf2F1F1YH8RVQf2F1F1YH8RVQf2F1F1YH8RVQf2F1F1YH8RVQdN+3MBU2B/EVUH9hdRdWB/EVUH9jfKv254q3p8y8dJ9+Trn6X/p1GXv6VsV1lfZpa/HbFdRcd22B7+mt6u+n5h1ucHynahbh3YX0TVgf1FVB3YX0TVQdP+BgVMFg/HJ598kqJcXoaqI7gxTbmsDHV9desI+4uh6gj7i6HqCPuLoeoI+xsNOrd0aq9/6oORzu6kqMvfUrarrC/HLH87YruKLm/3qkffnst2lfdLzPr8CGW7O0JH2F8MVUfYXwxVR9hfDFVHNOnPBcyYsL8Yqo6wvxiqjrC/GKqOsL8YuXPL3+HlfPGvXbu2Wr58ebXTTjtV//Iv/1JrHHnBhrRd/pbP7cjo43bJD3kiX+RNye+4mEf7U3WE/cVQdYT9xVB1hP3FUHVEk/5GCpjhYZnMsHAaqg6aHmICVQf2F1F1YH8RVQf2F1F1YH+j5GlFuZPL/7Bu3bpq2bJl1Zo1a6qNGzdWW7duLV65feTt/vszH41sd0fT1+2SH/JEvsgb+YNJ+Z1E0+0PVB3YX0TVgf1FVB3YX0TVQdP+XMAU2F9E1YH9RVQd2F9E1YH9jTI8rSh3ds8777xq1apVtRctwwx3ppVzM+piEbZL3sjfihUrxuZ3Gk23P1B1YH8RVQf2F1F1YH8RVQdN+3MBU2B/EVUH9hdRdWB/EVUH9hcZ1lG8rF+/vlCYLkIeKWLUdgDzbn+zsL+IqgP7i6g6sL+IqoOm/bmAKbC/iKoD+4uoOrC/iKoD+4tk3V133ZV+uTf9gXyeddZZUjuAebY/BfuLqDqwv4iqA/uLqDpo2p8LmAL7i6g6sL+IqgP7i6g6sL9I1nHuxI6cNmaah3ySVw7iCvNsfwr2F1F1YH8RVQf2F1F10LQ/FzAF9hdRdWB/EVUH9hdRdWB/ETRcvYoTwE3/IK/kV2Fe7U/Rgf1FVB3YX0TVgf1FVB007c8FTIH9RVQd2F9E1YH9RVQd2F8EDZfg5SpWpn+QV/KrMK/2p+jA/iKqDuwvourA/iKqDpr25wKmwP4iqg7sL6LqwP4iqg7sL4KG+4h4+lg/Ia/kV2Fe7U/Rgf1FVB3YX0TVgf1FVB007c83sizC/mKoOsL+Yqg6wv5iqDrC/mKg4WaIpr+Q3zLv42Je7U/REfYXQ9UR9hdD1RH2F0PVEU36AxcwRdhfDFVH2F8MVUfYXwxVR9hfDDQuYPqNC5gYdesI+4uh6gj7i6HqCPv7OsBTyArsL6LqwP4iqg7sL6LqwP4iaFzA9Bs1v/Nqf4oO7C+i6sD+IqoO7C+i6qBpfy5gCuwvourA/iKqDuwvourA/iJo1A6u6SZqfufV/hQd2F9E1YH9RVQd2F9E1UHT/lzAFNhfRNWB/UVUHdhfRNWB/Y3yrxveqh7f8nHq4D75+mfpf9MfyCd5VfPbdPsDVQf2F1F1YH8RVQf2F1F10LQ/FzAF9hdRdWB/EVUH9hdRdWB/o+RO7ZEXbBh0dk1/WGp+m25/oOrA/iKqDuwvourA/iKqDpr25wKmwP4iqg7sL6LqwP4iqg7sL3L9Ux+kzu2/P/NR+ZTpAeRVze882p+qA/uLqDqwv4iqA/uLqDpo2p8LmAL7i6g6sL+IqgP7i6g6sL9RlvoLvekWS81v0+0PVB3YX0TVgf1FVB3YX0TVQdP+XMAU2F9E1YH9RVQd2F9E1YH9jeJzYPqNz4GZTN06sL+IqgP7i6g6sL9RXMAU2F9E1YH9RVQd2F9E1YH9RdCoV6ky3UTN77zan6ID+4uoOrC/iKoD+4uoOmjan29kWYT9xVB1hP3FUHWE/cVQdYT9xUCjdnBNN/GNLGPUrSPsL4aqI+wvhqoj7O/rgIUuYHjja9eurZYvX17ttNNO6SDgaGeQH/JEvsibkt9x0ab2Ny7sL4aqI+wvBhr2IdNfyG+Z93Exr/an6Aj7i6HqCPuLoeoI+4uh6ogm/cHCTiFbt25dtWzZsmrNmjXVxo0bq61bt448b9oF+SFP5Iu8kb/MuPxOoi3tbxL2F1F1YH8RNC5g+o2a33m1P0UH9hdRdWB/EVUH9hdRddC0v4UrYLZs2VKtWLGiWrVqlYuWjkLeyB95JJ9qO4B5t79Z2F9E1YH9RdCoHVzTTdT8zqv9KTqwv4iqA/uLqDqwv4iqg6b9LVwBQ6d3/fr1hcJ0EfJIPtV2APNuf7Owv4iqA/uLoFE7uKabqPmdV/tTdGB/EVUH9hdRdWB/EVUHTftbqALmrrvuSr/cm/5APs866yypHcA825+C/UVUHdhfBI3awa2bzz77rHr++efT3OgvvviifHqH8uGHH1bPPvts9cYbb5RPbTOM/vKe2oaa33m1P0UH9hdRdWB/EVUH9hdRddC0v4UqYDh3wtPG+gX5JK80YoV5tj8F+4uoOrC/CBq1g1sXdPI5T234Qhy77bZb9fDDD6fnH3zwwWqXXXZJj/kB4phjjhl69dJ46aWXqg0bNgz+f/vtt6sjjjhiZNsHHXRQ9corrwy9aul861vfSut66qmnknfOyWsLan7n1f4UHdhfRNWB/UVUHdhfRNVB0/4WpoDh6lWcAG76B3klvwrzan+KDuwvourA/iJo1A5uXZx99tmpk//AAw9UX375ZfXqq69WJ5xwQrXzzjtX//znP9NyHsOLL75Ybd68uViDzi233JKKI2Bb++23X7XvvvtWf/vb39Ky5557Lv2/zz77DL9sSXz66afpM8wF2COPPFK98847hWp+qPmdV/tTdGB/EVUH9hdRdWB/EVUHTftbmAKGS/C26RczUx/klfwqzKv9KTqwv4iqA/uLoFE7uHXwwQcfpO1dc801I8uZ0nXSSSdVTzzxxEgBc9VVV1Xnnntuekyhw+gJxc+BBx5YPf7442n5pk2b0vJzzjmn2nXXXau99torFREUPxQvbO+www6r7r///vSYomWYp59+uvrud7+bvH3++efp/DnWw3bw9NFHH1Uff/xxtffeeyc/e+yxR1rvlVdemV5/6KGHpvWy3RdeeKE65JBDqr/+9a/puZtvvjktp0i65JJLks+mUfM7r/an6MD+IqoO7C+i6sD+IqoOmva3MAUM9xHx9LF+Ql7Jr8K82p+iA/uLqDqwv1H+dcNb1eNbPk4d3Cdf/yz9v6NhihXbo6M/ieEC5swzz6yOPvroNHpCAUHhcvfdd1ennXZaWg8jHfzPYwoJHlMsMNJCUXTeeeelQuSxxx6rfvGLXwympk3iiiuuSNu+7LLL0o8fFDKnn356Whfb2H333as777yzOv7449P/FDaMvPD4P/7jP5IuTyHj/BqWX3jhhalgY3l+X01APsmrmt+m2x+oOrC/iKoD+4uoOrC/iKqDpv0tTAGj/kJluoma33m1P0UH9hdRdWB/o+RO7ZEXbBh0dnc0Dz30UNofX3vttfKpAeMKmEcffTS97u9//3taTkFDQXDTTTcNCpj3338/PXfHHXcMXj88hYxihgJkGhRJFCwZRlkoYnIBc++996bl/DDC/0xv44ZqPM7T0nIBc8MNN6QrIWYuvvjiRguYpea36fYHqg7sL6LqwP4iqg7sL6LqoGl/gwKGDZfBlWOIcnkZqo7gYFAuK0Nd31J0agfXdBPyW+Z9XMyr/Sk6wv5iqDrC/mJc9ejbqXP77898VO42OwQ8sT/eeuut5VNp9KOcQpYLGE7E53VlMKpCATM8spJHRGC4gKGgYPlbb42ORLz33nvVRRddlJaz3T/84Q+D5+655570mlzADI8c8T9TxSYVMMcdd1x16qmnDvTD76spyCv5vfqxt0Puy5hH+1N1hP3FUHWE/cVQdYT9xVB1RJP+wCMwpheo+Z1X+1N0YH8RVQf2N8pSf6GvAy6XTCd+9erVI8vTgeb/7adcNWxcAcO0LZ5n5IZigvjLX/5Svf7666mAYZQkM6mAodhg+e9+97uBFn79618nDd7233//NH0s8/Of/zxdpSwXMJxXk5lVwHznO9+pDj/88IH+xhtvbLSAWWp+m25/oOrA/iKqDuwvourA/iKqDpr25wLG9AI1v/Nqf4oO7C+i6sD+RpnHOTBAUcA2GVXhxHlGNbgKGCe/w7gCJp9PwkgJ7+u2225L/3MC/rQCBh0FRb4/CwUFxQon/nP+St7WpZdemp7nL9PIKEbYDufS/OQnP9mmAua6665L68YPFyDgZP4mCxifAzOZunVgfxFVB/YXUXVgf6O4gDG9QM3vvNqfogP7i6g6sL8IGnX/qAuKCaZWsd0cnJzPlADgPjDDBUy+DwwnyQ+/hnNaYFoBw/1dKCjyuS+M4KxcuXJkPSeeeGK60hhQaOy5556D5yhmuLTzpAKGKW/5M8zP5QKG98mV0fif5xnJmXURgR1B/ixmMa/2p+jA/iKqDuwvourA/iKqDpr25wLG9AI1v/Nqf4oO7C+i6sD+ImjU/aNuKAoYQeHmkiqM2HAls6W8hkKivMrkm2++mbaNhxIupcyo0D/+8Y90sYBthcs133fffWn7rPPaa6+tDj744FK2w1HzO6/2p+jA/iKqDuwvourA/iKqDpr25wLG9AI1v/Nqf4oO7C+i6sD+ImjU/cMsDUZk+Gy5qhlT0xh9YVpZ06j5nVf7U3RgfxFVB/YXUXVgfxFVB037cwFjeoGa33m1P0UH9hdRdWB/ETTq/mGWDgdRppF9//vfTxcimAdqfufV/hQd2F9E1YH9RVQd2F9E1UHT/lzAmF6g5nde7U/Rgf1FVB3YXwSNun+YbqLmd17tT9GB/UVUHdhfRNWB/UVUHTTtzwWM6QVqfufV/hQd2F9E1YH9RdCo+4fpJmp+59X+FB3YX0TVgf1FVB3YX0TVQdP+BgVMFg8Hl44kyuVlqDqCq9CUy8pQ17cUnfoFb7oJ+S3zPi7m1f4UHWF/MVQdYX8x0Pj7r9/4+y9G3TrC/mKoOsL+Yqg6wv6+DnABY3qBD+Ax6tYR9hdD1RHz8ufvv37j778YdesI+4uh6gj7i6HqCPv7OsBTyEwvUPM7r/an6MD+IqoO7C+CRt0/TDdR8zuv9qfowP4iqg7sL6LqwP4iqg6a9ucCxvQCNb/zan+KDuwvourA/iJo1P3DdBM1v/Nqf4oO7C+i6sD+IqoO7C+i6qBpfy5gTC9Q8zuv9qfowP4iqg7sb5R/3fBW9fiWj9P+8eTrn6X/TX8gn+RVzW/T7Q9UHdhfRNWB/UVUHdhfRNVB0/5cwJheoOZ3Xu1P0YH9RVQd2N8ouVN75AUbBp1d0x+Wmt+m2x+oOrC/iKoD+4uoOrC/iKqDpv25gGk53Nn5+uuvLxdvEx9++GH6HF544YXyqc6j5nde7U/Rgf1FVB3YX+T6pz5Indt/f+aj8inTA8irmt95tD9VB/YXUXVgfxFVB/YXUXXQtD8XMC2HAua6664rF28TX3zxRbVx48ZUyPQNNb/zan+KDuwvourA/kZZ6i/0plssNb9Ntz9QdWB/EVUH9hdRdWB/EVUHTftzAdNyphUwf/zjH6s99tij2nnnnatDDjkkXcIOPv/882rdunXV7rvvnpafe+651fnnn58uSbfffvsl3aZNm6ojjjiiOuecc6pdd9212muvvapHHnmk2EJ3UPM7r/an6MD+IqoO7G8UnwPTb3wOzGTq1oH9RVQd2F9E1YH9jeICpuVMKmCeeeaZ9J5Wr15d/fnPf65WrFhR7bvvvtWXX35Z3XLLLamo2bBhQypk0B111FGDKWTPP/98dffdd6fHhx56aHrMayluuoqa33m1P0UH9hdRdWB/ETTq/mG6iZrfebU/RQf2F1F1YH8RVQf2F1F10LQ/38iy5UwqYNauXVvts88+g/83b96c3uM//vGP6tvf/nZ1ySWXDJ6juJlUwLz//vtJc8cdd6Sip6vwXsq8j4t5tT9FR9hfDFVH2F8MNF39/jMa/v6LUbeOsL8Yqo6wvxiqjrC/rwNcwLScSQUM079OOumkwf8fffRReo+PPvpoes1tt902eI4pZOMKGHSZhx9+uLOfEeDd0e8o9+txMa/vF0VHzMsfn5/pL94/YtStI+wvhqoj7C+GqiPs7+sATyFrOZMKmPXr11eHHXbY4P8nn3wyvUcSvttuu1WXX3754LkTTjhhbAHDuS+ZPhQwCvNqf4oO7C+CxvmNLEWnfn6mm6j5nVf7U3RgfxFVB/YXUXVgfxFVB037cwHTcihgOI/liSeeGAQJe+yxx9KUrwcffDDpTjvttGrlypXp8XHHHVftvffe1YsvvphGZNC5gPmKebU/RQf2F0Hj/EaWottpp52qrVu3lk+ZHkBeya/CvNqfogP7i6g6sL+IqgP7i6g6aNqfC5iWQwGD9+E48MAD08n6Rx55ZPqfAgUdVxaDLVu2pPNgeI7lnKB/7LHHjtwHxgXMZJbSrurUgf1F0Di/kaXoli9fni6hbvoHeSW/CvNqf4oO7C+i6sD+IqoO7C+i6qBpfy5gOs4rr7xSPffcc9Vnn319/f+HHnqoevrpp9N0MjjmmGOqiy66aPB8H1HzO6/2p+jA/iJonN/IUnRc9GPNmjXlU6YHkFfyqzCv9qfowP4iqg7sL6LqwP4iqg6a9ucCpofccMMNaXTl4osvrk4//fQ0QkNB02fU/M6r/Sk6sL8IGuc3shQdX/LLli3zNLKeQT7JK/lVmFf7U3RgfxFVB/YXUXVgfxFVB037cwHTQ7744ovq1ltvrU455ZR0BTJO8O87an7n1f4UHdhfBI3zG1mqjnPpVq1aVT5tOgz5POuss6R2APNsfwr2F1F1YH8RVQf2F1F10LQ/FzCmF6j5nVf7U3RgfxE0zm9kW3TcE4orGJruQx7Jp9oOYN7tbxb2F1F1YH8RVQf2F1F10LQ/FzCmF6j5nVf7U3RgfxE0zm9E0f3rhreqx7d8nHRPvv5Z+p9OL7/c78jpZGyH7UHebhMswnbJG/kjj+PyO42m2x+oOrC/iKoD+4uoOrC/iKqDpv35RpamF5DfMu/jYl7tT9ER9hcDjfMbQ9HRuaVTe/1TH4x0dpl2xLkTnADOVazqLmZyZ/rfn/lopHO/o+nrdskPeSJf5C1PG5uU30nRdPtbio6wvxiqjrC/GKqOsL8Yqo5o0h+4gDG9wB3cGHXriHn5c35jqLrcueXv8HK++Ll6FZfg5T4ifMZ1xpEXbEjb5W/53I6MPm6X/JAn8kXelPyOi3m0P1VH2F8MVUfYXwxVR9hfDFVHNOkPPIXM9AI1v/Nqf4oO7O8rzj777MFjNGV+77vvvpH/M035G0bVQdP+8shA7uTOGhmoy99StqusLzPL347YrqLL2736sbfnsl3l/cKszw+U7ULdOrC/iKoD+4uoOrC/iKqDpv25gDG9QM3vvNqfogP7+woKGHLKXzQ5vxQu+++//0iBM0xT/oZRddC0v+FpRbmzO426/C1lu8r6MrP87YjtKrpcPOCv6e2q7xdmfX6gbBfq1oH9RVQd2F9E1YH9RVQdNO3PBYzpBWp+59X+FB3Y39eU02goXPLjSTTpL6PqwP4iqg7sL6LqwP4iqg7sL6LqwP4iqg7sbxQXMKYXqPmdV/tTdGB/X5NHYcqYNPoCTfrLqDqwv4iqgzr85emHs3QZVQd1+MvUrQP7i6g6sL+IqgP7i6g6sL9RXMCYXqDmd17tT9GB/Y1SFi+z8ty0P1B1YH8RVQd1+MttaJYuo+qgDn+ZunVgfxFVB/YXUXVgfxFVB/Y3igsY0wvU/M6r/Sk6sL9RylGYaaMv0LQ/UHVgfxFVB9vrrzy3apJuGFUH2+tvmLp1YH8RVQf2F1F1YH8RVQf2N4oLGNML1PzOq/0pOrC/iDr6AvPwp+rA/iKqDrbX33BbmqYbRtXB9vobpm4d2F9E1YH9RVQd2F9E1YH9jeICxvQCNb/zan+KDuwvcsYZZ0ijLzAPf6oO7C+i6mB7/JWjebSrcbqSSesbx/b4K6lbB/YXUXVgfxFVB/YXUXVgf6O4gDG9QM3vvNqfogP7i6BxfiN166DP/oaLl+FRmFlMWt84tsdfSd06sL+IqgP7i6g6sL+IqgP7G2VQwLDhMrirJlEuL0PVEZs3bw7LylDXtxSd2gEy3YT8lnkfF/Nqf4qO2B5/mzZtqlavXl194xvf2CF3VnfUF+SHPJEv8qbkt4y6dcT2tL8y6tYR2+rv5JNPDjkgWF6+voxx65sU2+pvXNStI+wvhqoj7C+GqiOa8OfjYHdie46D4BEY0wvU/M6r/Sk62FZ/69atq5YtW1atWbOm2rhxY7V169aR5027ID/kiXyRN/KXGZffcdStg21tf+OoWwfb6i8fMIfvJZRjFuPWN4lt9TeOunVgfxFVB/YXUXWwo/35ONgttvc4uDAFDJWeG3M/Ia/kV2Fe7U/RwVL9bdmypVqxYkW1atUqt++OQt7IH3kkn2p7qVsHS21/06hbB9vij3NfKFzy/V9y0aKeW1Wubxrb4m8SdevA/iKqDuwvoupgR/nzcbD7bMtxcGEKmOXLl6dKz/QP8kp+FebV/hQdLNUfO/v69esLheki5JF8qu2lbh0stf1No24dbIu/XLhkcgGTdS5gRlHXV7cO7C+i6mBR/fk42B+WchxcmAJm7dq1aZjK9A/ySn4V5tX+FB0sxd9dd92VfrEw/YF8nnXWWVJ7UduVqoOltL9Z1K2DOvyVBcwsVB3U4S9Ttw7sL6LqwP4iqg52hD8fB/uHehxcmAKGN8kcOw8v9gvySV7Jr8K82p+ig6X4c3vuH0tpz2q7UnWwlPY3i7p1UIc/FzDTUddXtw7sL6LqYBH9+TjYP9Tj4MIUMAQnCLlS7xdqpZ6ZZ/tTUP15RLG/qCOKartSdaC2P2V9deugDn8uYKajrq9uHdhfRNXBovnzcbC/KMfBhSpgwHMl+8NS5kpm5t3+ZqH68zld/UU9p0ttV6oO1PanrK9uHdThzwXMdNT11a0D+4uoOlg0fz4O9hflOLhwBYyvVtF9tuVqFZl5t79ZqP58Vb3+ol5VT21Xqg6U9me6i5Jftb3UrQP7i6g6WDR/Pg72F+U4OChgcqMZjk8++SRFubwMVUdwY5pyWRnq+rZHx7QjXy+8O5Cf4euF52ljk/I7KdrS/iaF6k+5f4XpLuS3zHsZartSdUTZ/haF999/v/rss8/KxZ2kzOm0/I4Ltb3UrSPsL4aqIxbNn4+D/WbacRAWtoAheOPMsWOYyndsbXeQH/JEvsibkt9x0ab2Ny5Uf3wmpr+Q3zLvZajtStURZfvbFq666qrqzTffTI+POuqokf14zz33rG644YbiFfVy4oknVueff365eCLnnHNO8vbcc8+VT20zp556avXDH/6weu2119K6v/zyy1Iyk6effrq67bbbBv+znsceeyzdzXz4M915553T55w/8zKn0/I7LtT2UreOsL8Yqo5YNH8+DvabacdBWLgpZLOwv4iqA/uLqDpQ/fmLu98o+VXblaoDpf3NYpdddqmeeuqp9JiO9emnn1598MEH1T//+c/qN7/5TXpvt956a/Gq+njppZeqV155pVw8kd1337264447ysXbBQXMueeem0Z1OMhuC9dee211xBFHDP7nc3v00UcHBczbb7+dRqYpdJhOu9dee80slJT8qu2lbh3YX0TVwaL5U74nTXeZlV8XMAX2F1F1YH8RVQeqv1k7tuk2Sn7VdqXqoGx/dIi5euNuu+1WHXzwwdW3v/3t6o9//GN67s9//nO1zz77pILl2GOPrd55553Uccc7+jfeeCMVMLx+mKOPPro64IAD0mMKGQoI1nHGGWekTjkceuih1SmnnJKe23fffavrrrsuddB33XXX1LEHiiQ67oxAHHTQQdUTTzyRll988cVJ89ZbbyV/p512Wlr/3nvvHUZZ8ugL2/nb3/6WioP99tsvrfP444+vnn322aSjkOCiIfl9DfP73/8+LWcbJ510UvrMcgGDB7bLMn4x/t73vpfeA++F18EFF1yQlufP8swzz6xeeOGF9Bgf5513XtKVBcynn3468MD7YtmDDz44WDaOMr/jUNtL3Tqwv4iqg0Xzp3xPmu4yK78uYArsL6LqwP4iqg5Uf7N2bNNtlPyq7UrVQdn+/vKXv6TOOR35W265Jfn61a9+lZ5j+ZVXXpmeo5A47rjjqg8//DB1vB9++OHUaR9XwFBc0DGnA85fRmXogFOEXHjhhUlDQUGRw/JDDjkkbfeRRx5JhQzrB/RM02J0h04+RQ/kKWR5+haFEe9r5cqVqSgaBr+8j/vuu6/6/PPP07bQvPzyy+kvRRtQcOyxxx7VnXfeOXKuTH4Pd999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qTpQ25+yvrp10GV/n3/+eTrYbt26tXxqu+CO35999lm5uDPw2X744Yfl4iXT9c+hbajtXtXBtP0jo64PjfI9abrLrPy6gCmwv4iqA/uLqDpQ/c3asU23UfKrtitVB2r7U9ZXtw666u/aa6+tdt555xTkdp999qleeOGFUha46qqrqjfffLNcPMLvf//76tBDDy0X18aJJ55YnX/++eXiES655JLqkEMOSY+POuqo9B5z7LnnntUNN9xQvOIrbrvttvSZXHfddeVTS6YNn0PXufPOO6snnngiPVbbvaqDSfvHMOr60Cjfk6a7zMqvC5gC+4uoOrC/iKoD1d+sHdt0GyW/artSdaC2P2V9deugi/7uvvvulE/+whtvvFEdccQRqWM/i1122aV66qmnysUj7OiO+0svvVS98sor5eIRKGBWrlyZHlPAnH766dUHH3xQ/fOf/6x+85vfpPd/6623Fq/6qihYu3ZtuXibaMPn0HWOOeaY6he/+EV6rLZ7VQfj9o8SdX1olO9J011m5XdQwORGMxyffPJJinJ5GaqO4OZD5bIy1PXVrSPsL4aqI+wvhqojVH+zdmzTbchvmfcy1Hal6gi1/Snrq1tHdNHf/vvvXx1++OEj+aVjf9ZZZw2mk11zzTXV7rvvXu22227Vueeem6ZDnXrqqakdsIyiZ5jf/e53qQAiKIZyx50bBe63335pVOP444+vnn322bSc50855ZS0jX333TeNeOy1117VrrvumkaHAC/HHntsKpr23nvvasOGDWn5xRdfnDRvvfVWGjk67bTTBprnnnsuacoCZt26delx5uijj64OOOCAkWW33HJL8sm6eD+TvPP+1q9f34nPYRyM2jz22GPl4gF8VvghzjzzzLTsyy+/rM4+++zkf4899kjtA5555pm0bZbTpl577bWk5fPBC5/Dz3/+87SM53hvmd/+9rdpW5P88z5zPu644w653as6Ytz+UYa6PjQ+DvabacdBWOgChjfOrz/Lly+vdtppp5Fhb0e7gvyQJ/JF3pT8jos2tb9xofrjMzH9hfyWeS9DbVeqjlDbn7K+unVEF/3RKWQUApgOdvvttw/i9ddfr15++eWkeeihh6r7778/ddTpWHNeCJ3Jhx9+OHVIM4xs0D7QPP7446mDmzvuTOOig846+XvwwQen5XSOKSDoqKLh9Y888khaB9uA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>