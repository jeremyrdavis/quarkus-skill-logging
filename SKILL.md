---
name: quarkus-logging
description: >
  Use when adding or modifying log statements in a Quarkus application — invoking Log.info / debug / warn / error,
  importing org.jboss.logging.Log, choosing log levels, formatting log messages with placeholders, or deciding
  whether something is safe to log. Trigger on string concatenation inside log calls, missing parameterized
  format (debugf / infof), or PII appearing in log output. Does not cover application bootstrap, configuration of
  log handlers, or external sinks (Loki, OpenTelemetry) — only the call site.
---

# Quarkus Logging Skill

Conventions for emitting log statements in Quarkus code. The single most important rule: **never concatenate
strings inside a log call**. Use the parameterized `*f` variants instead.

---

## When to Use

- Writing or editing any code that calls `Log.info`, `Log.debug`, `Log.warn`, `Log.error`, `Log.trace` (or their
  `*f` variants).
- Adding a new resource, service, or repository that should emit observability events.
- Reviewing a diff that contains `Log.X("..." + something)` — that's a bug; this skill explains how to fix it.
- Deciding whether a value (entity, payload, request body) is safe to put in a log line.

**Out of scope** (don't load this skill for these): configuring log handlers in `application.properties`, wiring
up structured logging to external systems, OpenTelemetry trace export, or `quarkus-logging-json` setup.

---

## Core Rules

1. **Use the static `Log` API.** Import `io.quarkus.logging.Log`. Don't declare per-class loggers (`LoggerFactory`,
   `private static final Logger LOG = ...`) — Quarkus rewrites the static `Log` calls at build time and the result
   is shorter, equivalent, and zero-overhead.
2. **Always use the `*f` variants when there are arguments**: `Log.debugf`, `Log.infof`, `Log.warnf`, `Log.errorf`,
   `Log.tracef`. They take a printf-style format string and varargs. The non-`f` variants take a single message
   and are for fixed strings only.
3. **Never concatenate strings inside a log call.** `Log.debug("x=" + x)` always evaluates the concatenation, even
   when `debug` is disabled — wasted work. `Log.debugf("x=%s", x)` is lazy: the formatter only runs if the level
   is enabled.
4. **Don't mix placeholders and concatenation.** `Log.debug("Returning %s orders: " + orders.size())` is broken
   twice over: the `%s` is never substituted (the non-`f` variant doesn't format), and the concatenation runs
   eagerly. Pick one style, and the right one is `Log.debugf`.
5. **Pass the `Throwable` first when logging an exception.** `Log.errorf(t, "failed to process order %s", id)`.
   Don't log `t.getMessage()` — you lose the stack trace.
6. **Pick the level by audience, not by emphasis.** See the level table below.
7. **Don't log PII, secrets, or whole entities.** Log identifiers, action verbs, and decision branches. Personal
   data (email, name, address), credentials, tokens, and session IDs never go in log output.

---

## Canonical Example

```java
package com.example.orders.application;

import io.quarkus.logging.Log;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

@ApplicationScoped
public class OrderApplicationService {

    @Transactional
    public OrderId placeOrder(PlaceOrderCommand cmd) {
        Log.debugf("placing order for customer=%s, lineCount=%d", cmd.customerId(), cmd.lines().size());

        try {
            Order order = Order.place(cmd);
            orderRepository.save(order);
            Log.infof("order placed id=%s customer=%s", order.id(), cmd.customerId());
            return order.id();
        } catch (DomainException e) {
            Log.warnf("rejected order for customer=%s reason=%s", cmd.customerId(), e.getCode());
            throw e;
        } catch (RuntimeException e) {
            Log.errorf(e, "unexpected failure placing order for customer=%s", cmd.customerId());
            throw e;
        }
    }
}
```

What this demonstrates:

- Static `Log` import — no per-class logger field.
- `*f` variants throughout, with positional placeholders.
- Levels chosen by audience: `debug` for development, `info` for one-line state changes that operations cares
  about, `warn` for recoverable domain rejections, `error` (with the throwable first) for unexpected failures.
- Logged values are IDs and action codes — not the whole `cmd` object or the customer's name.

---

## Anti-patterns

Avoid these common mistakes when writing log statements:

| Don't | Why it's wrong | Fix |
|---|---|---|
| `Log.debug("Order with id " + id + " not found.")` | Concatenation always evaluates — wasted work when debug is disabled. | `Log.debugf("Order with id %s not found.", id);` |
| `Log.debug("Returning %s orders: " + orders.size())` | The `%s` is never substituted (non-`f` variant doesn't format), AND concatenation runs eagerly. | `Log.debugf("Returning %d orders", orders.size());` |
| `Log.error("failed: " + e.getMessage())` | Loses the stack trace; concatenation eagerly evaluates. | `Log.errorf(e, "failed");` |
| `Log.info("user " + user)` (where `user` is a JPA entity) | Calls `toString()` which often dumps the entire object graph, leaking PII. | Log the ID only: `Log.infof("user id=%s", user.id());` |
| `private static final Logger LOG = LoggerFactory.getLogger(...)` | Pre-Quarkus idiom; redundant given the static `Log` API. | Use `import io.quarkus.logging.Log;` and the static methods. |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in Slack. The right column is what defeats it.

| Excuse | Reality |
|---|---|
| "Just put the email in for one weekend — we'll roll it back Monday." | Logged data replicates to aggregators, backups, and SIEM in seconds. Rolling back the *code* removes future lines, not the ones already shipped. PII in logs is irreversible. |
| "I can't trace this specific customer's order without their email in the log." | The customer is already keyed by `customerId` everywhere else (database, support ticket, trace span). Look up the ID once; log the ID, not the email. |
| "It's only `debug` level — it won't be on in production." | Debug is enabled in production during incidents. The line you write is the line on-call sees at 3 AM. Treat every level the same way for PII. |
| "I'll concatenate this once because it's a one-off." | The lazy-evaluation rule isn't about code beauty; it's about not doing string work when the level is disabled. Once-off concatenations end up in hot loops six months later. |
| "Logging the whole entity gives the next debugger more context." | `toString()` on a JPA entity drags in the entire object graph, including lazy-loaded fields that may throw mid-serialization. The next debugger gets a stack trace instead of context. |

---

## Quick Reference

### Choose level by audience

| Level | Audience | Use when | Examples |
|---|---|---|---|
| `error` | Operations / on-call | The system can't continue or an invariant is violated | DB unreachable, unhandled exception, data corruption |
| `warn` | Operations | Degraded but recovered, or a domain rule rejected an action | Retry exhausted then succeeded, validation failure on a request, fallback used |
| `info` | Operations / audit | Notable state change worth keeping in production logs | Service started, scheduled job finished, order placed/cancelled |
| `debug` | Developer | Troubleshooting in dev or with debug enabled in prod | Method entry with key params, branch decisions, intermediate values |
| `trace` | Developer | Fine-grained, off in prod | Loop iterations, every message in a stream |

### Method cheat sheet

| You want to... | Call |
|---|---|
| Log a fixed message | `Log.info("started")` |
| Log a message with values | `Log.infof("id=%s count=%d", id, count)` |
| Log an exception with context | `Log.errorf(e, "failed for id=%s", id)` |
| Cheaply guard expensive logging | `if (Log.isDebugEnabled()) { ... }` (rare; the `*f` variants are already lazy) |

### What to log vs not log

| Log | Don't log |
|---|---|
| Identifiers (`orderId`, `customerId`) | Names, emails, addresses, phone numbers |
| Action verbs (`placed`, `cancelled`, `rejected`) | Full entity `toString()` |
| Decision branches and reason codes | Passwords, tokens, API keys, session IDs |
| Counts and sizes | Whole request / response bodies |
| Correlation IDs (request ID, trace ID) | SQL query parameter values that contain PII |
