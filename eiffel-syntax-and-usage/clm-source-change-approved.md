<!---
   Copyright 2025 Ericsson AB.
   For a full list of individual contributors, please see the commit history.

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
--->

# Source Change Approval Using Confidence Levels

_Source change approval_ is the process of signaling when a source change has
been reviewed and approved according to project requirements. This document
describes how to represent source change approval states
using [EiffelConfidenceLevelModifiedEvent][CLM] events, providing a standardized
way to communicate approval status using the Eiffel protocol.

Source change approvals can be used as a trigger to execute certain pipelines. Those
pipelines could either consume serious amounts of hardware/network resources and
thus should not run unless the source change is deemed interesting enough, or
they could run in sensitive environments and should therefore not be executed
unless the reviewer(s) say it's safe to run. By modeling approval as
confidence levels, teams can create flexible approval workflows that integrate
naturally with existing Eiffel-based CI/CD pipelines.

## Confidence Level as Approval State

The [EiffelConfidenceLevelModifiedEvent][CLM] provides a natural way to
represent approval states by treating approval as a confidence level of the
source change. This approach offers several advantages:

- **Semantic alignment**: Approval represents confidence in the quality and readiness of a change
- **Standard tooling**: Existing Eiffel tools understand confidence level events
- **Flexible names**: Supports different approval schemes and requirements

## Ideas for Approval State Representations

To exemplify how the CLM event could be used, we show some alternatives.

### Simple Approval States

The most straightforward approach uses discrete approval states:

```json
{
  "data": {
    "name": "readyToMerge",
    "value": "SUCCESS"
  }
}
```

**Ideas for Confidence Level Names:**

- `automatedChecksPass`: The automatic checks has passed
- `codeReviewed`: Code has been reviewed by at least one team member and no blocking issues have been identified. This
   indicates the change has received initial review attention, but does not constitute approval for merging
- `codeReviewApproved`: Code has received formal approval from authorized reviewer(s) according to project policy.
- `approvalPolicyMet`: Custom approval policy fulfilled. For example, some code might need architects or guardians to
   approve before merging
- `readyToMerge`: Automated test have passed, code reviewed, and all required approvals received

### Numeric Confidence Scores

For organizations using approval scoring systems:

```json
{
  "data": {
    "name": "approvalScore:85",
    "value": "SUCCESS"
  }
}
```

The numeric value can represent:

- Percentage of required approvals received
- Weighted approval score based on reviewer expertise
- Composite score including automated and human reviews

## Event Structure and Linking

### Basic Event Structure

```json
{
  "meta": {
    "type": "EiffelConfidenceLevelModifiedEvent",
    "version": "4.1.0",
    "time": 1234567890,
    "id": "aaaaaaaa-bbbb-5ccc-8ddd-eeeeeeeeeee0"
  },
  "data": {
    "name": "readyToMerge",
    "value": "SUCCESS",
    "issuer": {
      "name": "Review Management System",
      "email": "reviews@example.com"
    }
  },
  "links": [
    {
      "type": "SUBJECT",
      "target": "<source-change-created-event-id>"
    }
  ]
}
```

### Link Types Involved

This section describes the link types most relevant for source change approval
workflows. The CLM event supports additional link types (such as
CONFIDENCE_BASIS, CONTEXT, FLOW_CONTEXT, and SUB_CONFIDENCE_LEVEL) that may be
useful for more complex scenarios — see
the [EiffelConfidenceLevelModifiedEvent specification][CLM] for complete
details.

#### SUBJECT

Identifies the source change that this confidence level applies to. This link
connects the approval event to the
relevant [EiffelSourceChangeCreatedEvent][SCC]
or [EiffelSourceChangeSubmittedEvent][SCS].

**Required:** Yes  
**Legal targets:** [EiffelSourceChangeCreatedEvent][SCC], [EiffelSourceChangeSubmittedEvent][SCS]  
**Multiple allowed:** Yes  

#### CAUSE

Identifies what triggered this confidence level change. This could link to
events representing individual reviews, automated analysis results, or policy
changes.

**Required:** No  
**Legal targets:** Any  
**Multiple allowed:** Yes  

#### PREDECESSOR

Identifies a previous confidence level event that the current event outdates or otherwise replaces.

**Required:** No  
**Legal targets:** [EiffelConfidenceLevelModifiedEvent][CLM]  
**Multiple allowed:** Yes

## Override Examples

The following example shows one way to model multiple reviews and their dependence:

1. A developer pushes code for review (`SCC 1`).
1. The first reviewer gives a `+1` as they think it looks good which generates a `codeReviewed`:`SUCCESS`  (`CLM 1`).
1. The second reviewer spots a serious mistake and gives a `-2` which generates a `codeReviewApproved`:`FAILURE` (`CLM 2`).
1. The developer updates the code and pushes the new code for review (`SCC 2`).
1. The second reviewer accepts the changes by giving a `+2` which generates a `codeReviewApproved`:`SUCCESS` (`CLM 3`).

![Override example](./predecessor-simple.png)

CLM 2 could look like this

```json
{
  "meta": {
    "type": "EiffelConfidenceLevelModifiedEvent",
    "version": "4.1.0",
    "time": 1234567890,
    "id": "aaaaaaaa-bbbb-5ccc-8ddd-eeeeeeeeeee0"
  },
  "data": {
    "name": "codeReviewApproved",
    "value": "FAILURE",
    "issuer": {
      "name": "Bob Jones",
      "email": "bob.jones@example.com"
    }
  },
  "links": [
    {
      "type": "SUBJECT",
      "target": "<SCC1-event-id>"
    },
    {
      "type": "PREDECESSOR",
      "target": "<CLM1-event-id>"
    }
  ]
}
```

This example models a code review system where an approval from one user does not cancel the rejection from another user,
hence CLM 3 does not replace CLM 2. A consumer that tracks the approval state of source changes should list both CLM 2
and CLM 3 as current.

### Value of the PREDECESSOR Link

With the `PREDECESSOR` link you can express which events superseded each other
without relying on conventions that might not be known to all readers.

## Integration with Pipeline Workflows

### Pre-merge Pipeline Gating

Pipeline activities can wait for appropriate confidence levels before proceeding as shown with this pseudocode:

```json
{
  "pipelineGate": {
    "waitFor": {
      "eventType": "EiffelConfidenceLevelModifiedEvent",
      "conditions": {
        "data.name": "readyToMerge",
        "data.value": "SUCCESS"
      },
      "linkType": "SUBJECT",
      "linkTarget": "<source-change-created-event-id>"
    }
  }
}
```

### Progressive Approval Workflows

Different pipeline stages can trigger based on different confidence levels:

1. **Basic CI triggers** on `automatedChecksPass`
2. **Integration tests** trigger on `codeReviewed`  
3. **Deployment pipeline** triggers on `readyToMerge`

<!-- Bookmarks section -->

[CLM]: ../eiffel-vocabulary/EiffelConfidenceLevelModifiedEvent.md
[SCC]: ../eiffel-vocabulary/EiffelSourceChangeCreatedEvent.md
[SCS]: ../eiffel-vocabulary/EiffelSourceChangeSubmittedEvent.md
