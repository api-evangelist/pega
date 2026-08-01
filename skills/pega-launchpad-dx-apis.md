---
name: Drive a Launchpad case with the Pega DX API
description: Create a Pega/Launchpad case (or data object) from an external system, then advance it by running an assignment action — over REST with OAuth 2.0.
api: DX API (https://<appURL>/dx/api/application/v2)
operations: [createCase, getCase, createDataObject, runAssignmentAction]
source: https://github.com/pegasystems/pega-launchpad-agent-skills/blob/master/skills/launchpad-dx-apis/SKILL.md
generated: '2026-07-20'
method: generated
---

# Drive a Launchpad case with the Pega DX API

Pega Digital Experience (DX) APIs are the REST surface an external system uses to
create and drive Pega Platform / Launchpad **cases** and **data objects**. This
skill covers the create → read → act happy path. Ground every call in the
conventions in `conventions/pega-conventions.yml` and the errors in
`errors/pega-problem-types.yml`.

## Prerequisites (from the environment owner)

Ask the team hosting the Pega/Launchpad environment for four values, provisioned
through an OAuth 2.0 **Client Registration** rule whose **Persona** grants the
access roles you need:

1. Application URL (the `<appURL>` base)
2. Access Token URL
3. Client ID
4. Client Secret

## 1. Authenticate

Exchange the Client ID/Secret at the Access Token URL for an OAuth 2.0 access
token (Client Credentials grant). See `authentication/pega-authentication.yml`.
Send the token as `Authorization: Bearer <token>` on every DX API call.

## 2. Create a case

```
POST https://<appURL>/dx/api/application/v2/cases
{
  "caseTypeID": "ExpenseRequest",
  "content": { "Name": "...", "Category": "Travel", "TotalAmount": 1100 }
}
```

Only fields configured as **Allowed Fields** on the case type may be written;
supplying an unknown field returns an invalid-data error (400). The response
returns `data.caseInfo.ID` (opaque) and `businessID` (e.g. `ROT-715ABE`). Store
the `ID` — every later call uses it.

To create a **data object** instead, POST to `/objects` with `objectTypeID` +
`content`.

## 3. Read the case

```
GET https://<appURL>/dx/api/application/v2/cases/{ID}
```

Use this to pull the latest case state (and its available actions) before acting.

## 4. Run an assignment action (HATEOAS)

The create/read response embeds available actions as links, e.g.
`links["submit:Approve"]` with an `href` and a `type` naming the HTTP method
(`PATCH`). Invoke it verbatim:

```
PATCH https://<appURL>/dx/api/application/v2/assignments/{assignmentID}/actions/ReviewExpenseRequest?outcome=Approve
{ "content": { ... } }
```

Do not construct action URLs by hand — follow the `href` returned in the response.

## Errors

- **403** — the Persona on the Client Registration lacks access to that case/object; fix the Client Registration.
- **400 invalid data** — a field is not an Allowed Field, or a required Allowed Field is missing.
- **401** — missing/expired token; re-authenticate at the Access Token URL.
