# MICP: Managed Integration & Credential Protocol

**Integrations for agents, without lending them the keys.**

Version 0.2 · 26 November 2026 · Elizeu Carlos · All rights reserved

Project page: **https://micpprotocol.com**
Build an integration in the browser: **https://micpprotocol.com/builder.html**

---

## Abstract

Software agents are now asked to act on real systems: send mail, publish video, write rows into a
spreadsheet. The usual way to grant that access is to hand the agent a credential (a token, a password, an
API key) and to hope that the transcript stays private. This paper describes MICP, a declarative protocol in
which an agent never holds a credential. Integrations declare what exists; accounts hold the secrets,
encrypted under a single vault key; grants decide, per account and per method, who may call what; and every
call is signed, scoped and recorded. We separate what a protocol must specify from what an implementation
may choose, and we argue that existing tool-calling protocols sit above this layer rather than replacing it.

**Keywords:** agent tooling, credential management, least privilege, declarative integration, audit

---

## Contents

1. [Introduction](#1-introduction)
2. [The model](#2-the-model)
3. [Keys](#3-keys)
4. [Calls](#4-calls)
5. [Discovery](#5-discovery)
6. [Implementation](#6-implementation)
7. [Conformance](#7-conformance)
8. [Status of this document](#8-status-of-this-document)
9. [Beyond agents](#9-beyond-agents)

---

![The system at a glance](images/fig1.png)

*Figure 1. The whole protocol in one drawing: the agent asks, the client assembles the tools, the
integration server holds the accounts and performs the call, and the vault key stays behind, outside the
database and outside every conversation.*

The centre of the picture is the **integration server**: one server that holds many integrations. An
integration is a type of service, such as email, messaging or spreadsheets. Inside each integration there are
accounts, one per connection, each with its own credential. And each account enables the methods it is allowed
to use. One server, many integrations, several accounts per integration, a few methods per account.

Two roles sit around it. The **client** is the software that talks to the server: it asks what exists, and
presents each method to the model as a tool. It may be a chat application, a workflow engine, an editor or a
command line. The **agent** is the model inside that client, choosing among the methods and asking for an
action. The protocol constrains what the server declares and how a call is authorised; it says nothing about
who builds the client or in what language.

---

## 1. Introduction

This paper describes the server that sits behind the agent: one integration server, holding the integrations an
operator has installed, each with its accounts and the methods those accounts may use. It is not a library and
not a framework. It is a place where credentials live and calls are decided.

An agent needs to send an email. The easy way is to paste an application password into the chat. It works.
For about three weeks, nobody notices anything.

Then the account is used by someone else. Or the password is changed at the provider. Or a support ticket
arrives with the whole conversation attached. The password now lives in four places: the chat, the
provider's logs, the client's history, and whatever the user pasted it into. None of them is where it should
be.

There is a second problem behind the first. A password is not a permission. A mail password that lets you
send is also a key to read every message, delete folders and change settings. Saying *send this email* hands
over much more than the sentence says. And because the password travels with the agent, nothing can check
whether this one action was allowed.

Tool-calling protocols made this better. They move the password out of the chat and behind a tool. That fixes
the first problem. The second stays: a tool is either visible to the model or it is not, and *who may call
what* lives in the client, if it lives anywhere. The record of what happened is whatever the client chose to
keep.

MICP describes the layer below. The unit that matters is not the integration but the *method on a specific
account*. Passwords belong to a vault, not to a caller. Permission is data, not a property of a key.

Section 2 gives the model. Section 3 covers keys. Section 4 shows a call. Section 5 explains discovery.
Section 6 describes how to build a client. Section 7 lists the conformance checks. Section 8 separates what
is implemented from what is planned. Section 9 relates this layer to other tool-calling standards.

---

## 2. The model

Three nouns, and one rule for each:

- An **integration** is a type, such as `gmail`, `stripe` or `telegram`. It declares the methods this service
  can perform and how a credential for it is obtained.
- An **account** is one credential attached to an integration: a mailbox, a store, a bot. Accounts are data,
  so adding the tenth Gmail account is one row, not a new integration.
- A **method** is one operation with a contract: an input schema, an output schema, a transport, and a
  declaration of the secret it needs.

![Integration, accounts and methods](images/fig2.png)

*Figure 2. The three nouns. Twenty accounts sharing six methods produce six contracts, not a hundred and
twenty. And a method is enabled per account, so one mailbox may send while another may also read.*

A method is *declarative*: an endpoint, a request template, an input schema and an output map. It contains no
code. The executor is written once and reads the declaration, which is why adding an integration is a
configuration change and not a deployment, and why a pack can be reviewed by reading it.

```json
{
  "micp": "0.2",
  "integration": "email",
  "description": "Send email from a mailbox.",

  "defaults": {
    "smtp_host": "smtp.gmail.com",
    "smtp_port": 465,
    "tls": "implicit"
  },

  "accounts": [
    { "account": "elizeu",
      "config":  { "email": "elizeu@exemplo.pt" },
      "custom":  { "assinatura": "Elizeu Carlos" },
      "secrets": { "app_password": "{{vault:email.elizeu.app_password}}" } },

    { "account": "suporte",
      "config":  { "email": "suporte@exemplo.pt", "smtp_port": 587, "tls": "starttls" },
      "custom":  { "assinatura": "Equipa de suporte" },
      "secrets": { "app_password": "{{vault:email.suporte.app_password}}" } }
  ],

  "methods": {
    "send": {
      "description": "Send an email from the selected account.",
      "transport": "smtp",

      "endpoint": {
        "host": "{{defaults.smtp_host}}",
        "port": "{{account.smtp_port | default:defaults.smtp_port}}",
        "tls":  "{{account.tls       | default:defaults.tls}}"
      },

      "request": {
        "auth":    { "user": "{{account.email}}", "password": "{{secret.app_password}}" },
        "from":    "{{account.email}}",
        "to":      "{{input.to}}",
        "subject": "{{input.subject}}",
        "body":    "{{input.body}}\n\n{{account.custom.assinatura}}"
      },

      "fields": {
        "to":      { "type": "email", "required": true },
        "subject": { "type": "text",  "required": true, "max": 200 },
        "body":    { "type": "text",  "required": true }
      },

      "output": { "sent": "constant:true", "message_id": "smtp.message_id" }
    }
  },

  "destinations": ["{{defaults.smtp_host}}"]
}
```

Configuration lives at three levels, and the method is written once for all of them. That is what makes a
second account a row of data instead of a second integration.

**Table 2. Where a value comes from.**

| In the template | Source | Visible to |
|---|---|---|
| `{{defaults.x}}` | the integration defaults, shared by every account | everyone |
| `{{account.x}}` | a field of the account, in `config` | discovery, UI, logs |
| `{{account.custom.x}}` | a user-defined extra, in `custom` | discovery, UI, logs |
| `{{secret.x}}` | the vault, decrypted for the duration of the call | nobody |
| `{{input.x}}` | a field declared in `fields` and sent by the caller | the caller |

The rule for placing a value is one question: *if this appeared in a screenshot or in the discovery response,
would it grant access?* If yes, it belongs in the vault. If no, it belongs to the account. Nothing else
decides.

Two consequences follow. First, the account is a *parameter* of the method, never a free string: the caller
picks from a list, so a model cannot invent a mailbox. Second, a secret is a *reference*, never a value: the
pack can be shared, reviewed and kept in a repository.

---

## 3. Keys

MICP has two kinds of key, and they answer different questions. Provider keys belong to the services, and
there are many. The vault key belongs to the installation, and there is one.

**Table 1. Keys in a MICP deployment.**

| Key | Count | Function | Leaves the server |
|---|---|---|---|
| Vault key | one per installation | encrypts and decrypts stored credentials | never |
| Account credential | one per account | what the provider issued | only on the outbound call |
| Access token | one per agent or run | who may call which method | to that caller |

The vault key is the only secret an operator has to keep, rotate and back up. It does not authenticate
anyone and it does not grant anything: it opens a store. Permission is decided in the grant table, because a
key that decides policy has to be edited to change policy, and that is how permissions end up spread across
files. It is a plain pair of keys: one private key that decrypts, one public key that encrypts. Two files,
two lines of configuration, no special format: X25519 to agree a secret, AES-256-GCM to seal each value, and
the record's address mixed into the authenticated data. Three commands cover its whole life: `keygen`,
`verify` and `rotate`.

![The vault key sits outside the database](images/fig3.png)

*Figure 3. The vault key lives outside the database. A stolen dump contains ciphertext, account names and
grant rows, and no usable secret.*

### 3.1 Storage and retrieval

A credential is encrypted before it touches disk, with a random nonce per record and the record's address
bound into the authenticated data: installation, integration, account, field. Copying a row into another
account therefore fails to decrypt, which turns a database write into a non-event.

![Four moments of a credential](images/fig4.png)

*Figure 4. The plaintext exists between iii and iv, in memory, for the duration of one call.*

Rotation replaces the vault key without asking anyone for anything: a new key is generated alongside the
old, new writes use it, a background pass re-encrypts the remaining rows, and the old key is retired when no
row refers to it. Rotation bounds the damage of a key copied without the operator's knowledge; it does not
stop an attack in progress, which is a matter for revocation.

---

## 4. Calls

A call carries a method, an account, the inputs, and a signature over the exact bytes. Two questions get two
different answers, and mixing them produces systems that authenticate and never authorise.

The *signature* says whether the request is genuine and unchanged. It covers the body, a timestamp and a
nonce, so a replay is visible and useless. The *grant* says whether this caller may run this method on this
account. A good signature from a caller without a grant is a well-formed refusal.

```http
POST /v1/call/email.elizeu.send
Authorization: Bearer <access token>       # who is calling, and only that
Content-Type: application/json

{ "input": { "to": "joao@exemplo.pt",
             "subject": "Relatório",
             "body": "Segue em anexo." } }
```

![Sequence of a call](images/fig5.png)

*Figure 5. The caller sees steps 1 and 8. Steps 2 to 7 happen behind the counter, and only step 6 leaves the
server, carrying the provider credential, never the vault key.*

Errors are typed rather than free text: invalid input, vault unavailable, not authorised, provider failure,
idempotency conflict. A caller can therefore react deterministically, and an operator can count failures by
kind. Idempotency is keyed by the caller's request identifier and attempt, so a retry after a crash does not
send the same email twice.

### 4.1 Three calls, end to end

The call carries the method in its name. Three groups, always in this order:
`integration.account.method`. The body carries only the inputs.

![Two hops](images/fig6.png)

*Figure 6. Two hops, never more. The caller speaks HTTP to the integration server; the server speaks the
provider's own transport. Between them sit the grant, the vault and the rendered template, and the key never
comes back up the wire.*

```http
POST /v1/call/email.elizeu.send
Content-Type: application/json

{ "input": { "to": "joao@exemplo.pt",
             "subject": "Relatório",
             "body": "Segue em anexo." } }
```

What the integration server does with it, with the secret masked:

```
defaults.smtp_host            smtp.gmail.com
account.smtp_port             465        (from defaults, no override)
account.email                 elizeu@exemplo.pt
account.custom.assinatura     Elizeu Carlos
secret.app_password           ***

CONNECT smtp.gmail.com:465  implicit TLS
AUTH  elizeu@exemplo.pt  ***
MAIL FROM: elizeu@exemplo.pt
RCPT TO:   joao@exemplo.pt
DATA
  Subject: Relatório

  Segue em anexo.

  Elizeu Carlos
```

The same method, called on the other account, produces a different request with no change to the pack: the
second account overrides `smtp_port` and `tls`, and brings its own signature and its own secret.

```http
POST /v1/call/email.suporte.send
{ "input": { "to": "cliente@exemplo.pt", "subject": "Bilhete #4821", "body": "Em análise." } }

CONNECT smtp.gmail.com:587  STARTTLS      <-- override da conta
AUTH  suporte@exemplo.pt  ***
```

A third call, on a service that uses an HTTP API and a custom field, shows the same shape applied elsewhere.
Here the pack reads a destination from `defaults`, a channel from the account, a tone from the user-defined
extras, a token from the vault, and three fields from the caller.

```http
POST /v1/call/telegram.canal_curso.send
{ "input": { "text": "Episódio 12 no ar", "silent": false } }

-> POST https://api.telegram.org/bot***/sendMessage
  { "chat_id": "-100123456",                 <- {{account.chat_id}}
    "text": "Episódio 12 no ar",             <- {{input.text}}
    "parse_mode": "Markdown",                <- {{account.custom.formato}}
    "disable_notification": false }          <- {{input.silent}}
```

Nothing in these three calls required a line of code per integration. What changed between them was data: an
override here, a custom field there, a different transport in the third.

### 4.2 What comes back

Describing the response is optional. If a method has no `output`, the provider's answer comes back as it
arrived. That is the default, and it is what you want while you are building the integration.

If the method declares `output`, only the listed fields come back, each with a type:

```json
"output": {
  "message_id": { "path": "body.id", "type": "text", "required": true },
  "accepted":   { "path": "smtp.accepted", "type": "list" }
}
```

`output` chooses and checks. It does not transform: no renaming, no date arithmetic, no joins. If you need
that, do it where you call, not inside the pack.

Two optional marks per field handle the two ways a response can disappoint you. `default` covers a field that
did not come: the server returns the default instead of failing. `expected` covers a field that came with the
wrong value: the call fails, because a response that says `ok: false` is not a success even when the HTTP
status is 200.

```json
"output": {
  "ok":   { "path": "body.ok",   "type": "boolean", "expected": true },
  "chat": { "path": "body.chat", "type": "text",    "default": "" }
}
```

When `expected` fails, the answer is a typed error:

```json
{ "ok": false, "error": "resposta_inesperada",
  "campo": "ok", "esperado": true, "recebido": false,
  "efeito_possivel": true, "tentavel": false }
```

`efeito_possivel` matters: the provider answered 200, so the action may have happened. It is not an error to
retry blindly; it is one to check first.

One exception is worth spelling out. A few services return a *new* credential in the response, a refresh
token being the usual case. If the answer carries a field whose name matches a secret already stored for that
account, the server stores it in the vault, encrypted, and returns `{"stored": true}` in its place. The value
never leaves the server. Most integrations never do this, and for them it costs nothing. Large responses are
cut at 64 KB.

---

## 5. Discovery

A server that exports everything exports too much. Discovery uses the same grants as calls, and the filtering
happens on the server, before the caller sees anything.

A method the caller cannot use is not refused later. It never appears. The difference is not cosmetic. A
refusal tells the agent that something exists and invites a second attempt; invisibility leaves nothing to
try. It also keeps prompts small, which is why most deployments end up doing this anyway.

![Three filters between catalogue and agent](images/fig7.png)

*Figure 7. Two filters in series. Neither happens in the model, and neither requires the caller to be
trusted.*

Because the announcement contains no secrets (names, descriptions, schemas), it can be served freely, cached
and shown in a user interface. It is a menu, not a key.

---

### 5.1 The discovery response

A caller asks the server what exists:

```http
GET /v1/discovery
```

```json
{ "micp": "0.2",
  "integrations": [
    { "integration": "email",
      "accounts": [
        { "account": "elizeu",
          "methods": [
            { "name": "email.elizeu.send",
              "description": "Send an email from the selected account.",
              "input":  { "to": {"type":"email","required":true},
                          "subject": {"type":"text","required":true,"max":200},
                          "body": {"type":"text","required":true} },
              "output": { "sent": {"type":"boolean","expected":true},
                          "message_id": {"type":"text"} } },
            { "name": "email.elizeu.list", "…": "…" }
          ] },
        { "account": "suporte", "methods": [ "…" ] }
      ] },
    { "integration": "telegram",
      "accounts": [ { "account": "canal_curso", "methods": [ "…" ] } ] }
  ] }
```

Three levels, in the order a person thinks about them: the integration, the accounts inside it, and the
methods each account may use. A method carries four keys: `name`, `description`, `input` and `output`.

The `name` is repeated at every level on purpose. It could be assembled from the three parts, but a caller
that has to concatenate strings eventually concatenates them wrong. Given the name, it copies it.

There are no paths, no headers, no transport and no endpoint in this response: those belong to the pack and
never leave the server. The list contains only what is installed and enabled. A credential grants access to
the server; how a deployment organises tenants, roles and interface is left to the implementation.

The same data can be rendered as a compact block for a prompt, with `?target=text`. Any other value is
refused with `alvo_invalido`, so a typo surfaces on the first attempt instead of in a client three weeks
later.

## 6. Implementation

A model does not need a protocol to get tools. It needs a name, a description and a JSON schema. Whoever
builds that list is the client, and building it is the work this section describes.

1. **Place the vault.** In the same process, or on a separate integration server. If separate, the client
   records the server's address and the client pairs once with a short-lived code.
2. **Expose discovery.** The server answers what is installed and enabled: accounts, methods, input and
   output schemas, filtered as in Section 5.
3. **Cross with grants.** Keep only what this agent may call. This happens before the model is prompted, not
   after it tries.
4. **Translate into tools.** One tool per account and method, with a stable name, a natural-language
   description and the fields as a schema. The account is folded into the name so that it cannot be invented.
5. **Forward calls.** When the model selects a tool, send
   `POST /v1/call/{integration}.{account}.{method}` with the inputs, verify the result and return the mapped
   output.
6. **Install the pack.** An integration is one file on the server. Drop `packs/<integration>.micp.json` and
   reload: nothing else to build, nothing to deploy.

Announcing everything is simpler to build and easier for a model to use. It also costs context on every call.
That is a product decision, not a protocol one, and it can be changed per account, which is why enablement
belongs to whoever operates the server.

### 6.1 Building an integration

An integration is a file, not a program. The pack says what the methods are; the server reads it and does the
work. There is no build step, no package to install and no code to review line by line: you read the file,
and you know what the integration can do.

To make that concrete, there is a builder at
[micpprotocol.com/builder.html](https://micpprotocol.com/builder.html). Four steps: the integration and its
defaults, the accounts, the methods, and the JSON. It shows the request that will be sent, with the secret
masked, and gives you the file to drop on your server.

What it will not do is the interesting part. There are no conditions, no loops, no expressions and no code
inside a pack. If a service needs logic to be called, it needs a new transport or it waits. A refusal is
easier to audit than a language.

---

## 7. Conformance

An implementation conforms to MICP v0.2 if it satisfies the following. The list is deliberately short: every
item can be checked by reading a pack or by issuing a request.

1. A method is one operation, with an input and an output schema.
2. Every method names a transport from `{http, smtp, imap}`.
3. Secrets are referenced, never embedded: `$vault:name.field`.
4. No pack contains a credential, in any form, including examples.
5. Every integration declares an allowlist of destinations.
6. Grants are per account and per method, never per integration.
7. Discovery returns only what the caller is granted.
8. Every call is idempotent per run, node and attempt.
9. Calls carry a timestamp and a nonce; replays are rejected.
10. Errors are typed, and the type is stable across versions.
11. The vault key is stored outside the database, with a separate backup and a tested restore.
12. Every call is recorded: account, method, caller, run, outcome, duration.

---

## 8. Status of this document

This document describes a protocol that is partly implemented. The difference matters, so it is stated rather
than implied.

**Table 3. Implemented and intended, as of v0.2.**

| Element | State | Note |
|---|---|---|
| Declarative methods | implemented | transports http, smtp, imap |
| Accounts as data | implemented | methods enabled per account |
| Grants per method | not in 0.2 | one access level per server; scoping is left to the implementation |
| Filtered discovery | implemented | only installed and enabled methods appear |
| Audit of calls | implemented | account, method, caller, outcome |
| Vault, X25519 and AES-256-GCM | implemented | keygen, verify and rotate; one key per installation |
| Discovery endpoint | implemented | `GET /v1/discovery`, JSON and text |
| Signed calls with nonce | in progress | scoped tokens shipped; body signature next |

The protocol is specified carefully because the failure is expensive: a password handed to an agent cannot be
taken back. A feature written down but not built is a promise; a feature built but not written down is a
rumour. The table keeps both visible.

---

## 9. Beyond agents

Nothing in this protocol assumes a model. A nightly job that publishes a report, a back end that sends an
invoice when an order ships, a command line a person uses by hand: each of them is a caller, and each is
authorized and recorded in the same way.

The agent case is what forced the design, because an agent can invent a call and a model cannot be trusted
with a key. But the rules it produced are not agent rules. A deterministic program that knows exactly which
method it wants benefits from the same three properties: the secret stays in the vault, the method is
declared rather than coded, and the call leaves a record.

A caller needs two things: the server's address and a credential. It does not need a model, a framework, or an
SDK. That is why the protocol is specified as HTTP and JSON: a cron entry with `curl` is a first-class caller,
and so is a compiler with a generated client.

The same holds for the other direction. A deployment that already has its own scheduler does not have to adapt
to an agent framework to use this: it calls
`POST /v1/call/{integration}.{account}.{method}` and reads `{ok, output}`. Whether a model, a queue worker or
a person asked for it is a detail the server records and does not need to know.

---

## How to cite

> Carlos, E. (2026). *MICP: Managed Integration & Credential Protocol* (Version 0.2). Zenodo.
> https://doi.org/10.5281/zenodo.XXXXXXX

Version 0.2 is the current draft. A `CITATION.cff` file is included, so GitHub shows a "Cite this repository"
button.

## Revision history

| Version | Date | Changes |
|---|---|---|
| 0.1 | 26 Nov 2026 | First public draft: nine sections, seven figures, twelve conformance checks. |
| 0.2 | 26 Nov 2026 | Methods declare `endpoint` and a `request` template. Three levels of configuration (`defaults`, `account`, vault) and five template namespaces. Calls use `integration.account.method`. Optional `output`. Added 4.1 (three end-to-end examples), 4.2 (responses), 6.1 (building), and the browser builder. |

## License and terms of use

**Copyright (c) 2026 Elizeu Carlos. All rights reserved.**

This specification, its text, figures and diagrams may not be reproduced, distributed, published, modified,
translated or adapted, in whole or in substantial part, without prior written permission. Brief quotation (up
to 150 words) for review, commentary or academic citation is permitted, provided the author and source are
identified and a link to https://micpprotocol.com is included. See [LICENSE](LICENSE) for the full notice.

Quoting this document in an article, a paper or a review is welcome. Republishing it, redistributing it, or
building a derived specification from it is not.

Independent implementations of the protocol require a separate licence: hello@micpprotocol.com
