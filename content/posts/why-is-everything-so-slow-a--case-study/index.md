---
title: '"Why is everything so slow?" - a case study'
date: 2026-06-04
cover: cover.jpg
tags:
  - Performance Engineering
  - Case Study
  - Juju
summary: |
  A Juju charm performance case study tracing slow NRPE target reconciliation to repeated downstream relation databag updates, and showing how batching cuts hook-tool invocations dramatically.
description: |
  COS Proxy became painfully slow in large fan-in deployments. The bottleneck was not Juju itself, but charm code that rewrote downstream scrape jobs and alert rules on every generated object. Batching those updates changed the cost from quadratic growth to a much smaller relation-event-bound pattern.
---

{{< disclaimer >}}
This post is very different from what I usually post about, but since it **might** be of interest to someone, I thought I'd re-post it here as well. The original post is available in the [Canonical Charmhub Discourse](https://discourse.charmhub.io/t/why-is-everything-so-slow-a-case-study/20491).
{{< /disclaimer >}}

Today I'd like to talk a little bit about how I go about doing performance improvement work in Juju charms, and especially about a certain use case that I've been analysing, and hopefully fixing, during the last few days.

If you've ever been using [COS Proxy](https://charmhub.io/cos-proxy) in any meaningful capacity, you most likely noticed that the time needed to reconcile NRPE targets grows very quickly as you start to add more and more targets to it. This turned out to become a real problem when you start to reach enterprise numbers, like 200 units from an assorted collection of applications.

We've not really known why this is happening, and we've not put much effort into resolving it either, given that COS Proxy only ever was meant to be a temporary stepping stone, bridging the world of LMA and the one of COS. At some point earlier this week I finally succumbed and decided to spend a few hours, trying to go to the bottom of whether this is something we, as charmers, can meaningfully look to improve or whether the bottleneck is within Juju itself.

## The framing

![waiting, and waiting, and waiting](waiting-line-ticket-movie-queue.gif)

COS Proxy can take a very long time to settle in large machine fan-in deployments that use the `monitors` relation. Our theory has been that this is caused by the large amount of events that need to be processed for it to reconcile.

In a medium-sized deployment, let's say 50 units, establishing a relation would result in each of those units first emitting a `relation-joined`, followed by a `relation-changed`, including only the delta up to that point. In the next event, we'd get that same delta, plus the new node that emitted the event. For each of those, we update our downstream databag with scrape information and alert rules, and then we repeat until the queue has been drained.

## The diagnosis

After some initial conversation with my agent, the first root cause suggested was a DNS resolution we do on every target address we receive. The agent stated that this very well could end up taking _a loooong time_.

When I started digging, however, it turned out that this change was introduced much later than when we first heard about there being performance issues with the proxy. After a bit of back and forth we could safely rule out the DNS resolution, although that's probably another area we'd need to look at improving going forward.

Instead, the expensive path turned out to be the `monitors-relation-changed` event handler, where NRPE monitors data is converted into downstream scrape jobs and alert rules matching the condition Nagios would have reacted on.

## The main offender

Let's see if you can spot it, just by looking at the code.

```python
    def _on_nrpe_targets_changed(self, event: Optional[NrpeTargetsChangedEvent]):
        """When NRPE targets change, recalculate alert rules and scrape configs.

        This method updates the stored state and relation data downstream relations join
        (e.g. cos-agent, prometheus, etc.). It recalculates all the data sources and passes it to
        stored state.
        """
        if event and isinstance(event, NrpeTargetsChangedEvent):
            for target in event.removed_targets:
                self.metrics_aggregator.remove_prometheus_jobs(target)  # type: ignore

            for alert in event.removed_alerts:
                self.metrics_aggregator.remove_alert_rules(
                    self.metrics_aggregator.group_name(alert["labels"]["juju_unit"]),  # type: ignore
                    alert["labels"]["juju_unit"],  # type: ignore
                )

            nrpes = cast(List[Dict[str, Any]], event.current_targets)
            current_alerts = event.current_alerts
        else:
            # If the event arg is None, then the stored state value is already up-to-date.
            nrpes = self.nrpe_exporter.endpoints()
            current_alerts = self.nrpe_exporter.alerts()

        self._modify_enrichment_file(endpoints=nrpes)

        # Add scrape jobs for prometheus and cos-agent relations
        for nrpe in nrpes:
            self.metrics_aggregator.set_target_job_data(
                nrpe["target"], nrpe["app_name"], **nrpe["additional_fields"]
            )
        # NOTE: We never stop vector once started, so we assume that within an NRPE event, the
        #       vector target can be scraped in the future
        vector_target = {self.unit.name: {"hostname": self.host, "port": VECTOR_PORT}}
        self.metrics_aggregator.set_target_job_data(vector_target, self.app.name)

        for alert in current_alerts:
            self.metrics_aggregator.set_alert_rule_data(
                re.sub(r"/", "_", alert["labels"]["juju_unit"]),  # type: ignore
                alert,  # type: ignore
                label_rules=False,
            )
```

Notice how the for loops are all setting data in the relation databag at every iteration? No? Well, that makes sense as that exact call is hiding behind a library abstraction! But it [is there](https://github.com/canonical/cos-proxy-operator/blob/c0b7f453bee91412976961f55222ffa589e6f629/src/metrics_endpoint_aggregator.py#L303-L306) - scout's honor!

While Juju promises us it won't start to emit any events until the event processing is done, that doesn't mean that calling a hook tool is without cost. Every call will still go through ops, shell out, invoke a hook tool, have Juju process it, and then return back through the abstractions to the caller. Juju buffers the resulting relation data delta in memory until the current hook exits, and defers any resulting event emission until then, but that does not make the hook-tool invocation free.

This code is obviously not excellent. But it's also not sloppy or bad enough to trigger immediate suspicion that it would need change. In a relatively trivial test setup, let's say 10 units with 3 checks each, the whole routine still finishes quick enough that it wouldn't be caught. The problem surfaces at scale, as is often the case with these types of problems.

The current implementation regenerates the full current monitor state on every relation-changed event, only to then replay every generated scrape job and alert rule.

![just a small leak](flood-pipe.gif)

For setups with `U` units and `C` checks per unit, where the relation has just been established, downstream update operations grew quadratically, to

```
C * U * (U + 1) + U
```

which in Big O notation would be:

```
O(C * U^2)
```

For 200 units with 18 checks each, that is about 723,800 downstream update operations. And if that is true, it also means roughly 723,800 Juju hook-tools invocations. 😱 If this was done on a static collection, `ops` would have swallowed this, preventing it from ever reaching Juju, but since we expand the collection for each assignment, the equality check will never be able to catch it, as indeed: the collection **DOES** change for every iteration.

## Solution

The solution is trivial, once you've spotted it: Batch the updates and write each downstream databag key once per function call. In the proposed PR we build a full set of generated scrape jobs, alert rules, removed targets, and removed alerts in memory, and then apply them through a single batch aggregator method.

This changes the downstream relation update pattern from per-generated-object invocations to a single call carrying all the changed downstream databag keys.

The shape for downstream update operations changes from `O(C * U^2)` to `O(U)`.

## Caveats

Juju still emits `relation-changed` events as units join, and each hook sees the current relation state. This means COS Proxy still regenerates monitor-derived data for the current relation state on each event. In other words, this is still not efficient - far from, but it's efficient in the places that a charm author can meaningfully hope to influence.

## Conclusion

Just to verify the results, a manual benchmark was added under `tests/manual`, comparing sequential operations using the current `2/stable` release, and the batched approach suggested in the PR.

```
  20 units * 18 checks:
    sequential: 206.08s
    batched:     34.45s
    improvement: 5.98x faster, 83.3% reduction

  50 units * 18 checks:
    sequential: timed out after 1800s, incomplete
    batched:      97.81s, complete
    improvement: at least 18.4x faster
```

I don't want to extrapolate numbers for the actual impact on 200 units or more, but I think it's safe to say that the improvement will be significant.
