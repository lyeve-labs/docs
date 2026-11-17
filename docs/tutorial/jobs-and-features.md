---
title: "19. Jobs and features"
description: "Call an address on a schedule, get told when the job fails, and see which features run on your engine and what each one serves."
sidebar:
  order: 19
covers:
  console:
    - "/admin/jobs"
    - "/admin/plugins"
    - "/admin/plugins/[name]"
  api:
    - "POST /api/admin/cron/jobs"
    - "POST /api/admin/cron/jobs/{id}/trigger"
    - "GET /api/admin/jobs"
    - "GET /api/admin/plugins/status"
    - "GET /api/admin/plugins/running"
verified:
  engine: v0.62.0
  console: v0.28.0
---

Some work has to happen on a clock, such as rebuilding the trip pages of your site every
night. In this lesson you schedule that call and get told when it fails. Then you look at
which features run on your engine, which is where you find out why a screen from an earlier
lesson is locked. Sign in as a super admin, and keep a token in `TOKEN` as in
[lesson 18](privacy-and-audit.md).

## Call an address every night

> **Included free** on every install, with email alerts for a failing job. Slack, Discord,
> PagerDuty and webhook alerts need a license with the `alerts-pro` feature.

<div class="paths">
<div data-path="console">

1. Open **Operations > Scheduled jobs** and choose **New job**.
2. Enter `rebuild-trip-pages` in **Name** and what it does in **Description**.
3. Enter `0 2 * * *` in **Cron schedule**, which runs at 02:00 every day.
4. Enter your site's rebuild address in **Endpoint URL**, and `{"scope": "trip"}` in
   **JSON payload (optional)**.
5. Keep **Enabled** on.
6. Under **Failure alerts**, enter your address in **Email** and `3` in
   **Alert after this many failures in a row**.
7. Choose **Create**.

</div>
<div data-path="api">

```bash
curl -X POST http://localhost:3001/api/admin/cron/jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "rebuild-trip-pages",
        "schedule": "0 2 * * *",
        "endpoint": "https://build.example.com/hooks/rebuild",
        "payload": {"scope": "trip"}
      }'
```

The answer is `201` with the job. Copy its `id` into `JOB_ID`.

</div>
</div>

![Opening New job, entering the schedule and the endpoint, setting an email alert and creating the job.](./media/jobs-and-features-new-job.mp4)

The list shows each job's **Schedule**, **Endpoint**, **Status** and **Last run**. The
buttons on a row edit the job or delete it. Deleting stops its schedule and deletes its run
history.

On a free install, the Slack, Discord, webhook and PagerDuty fields carry a not enabled marker,
and you cannot add a channel there. A channel you already set keeps sending.

## Run a job now

This is an API task. The console has no button to run a job outside its schedule.

```bash
curl -X POST http://localhost:3001/api/admin/cron/jobs/$JOB_ID/trigger \
  -H "Authorization: Bearer $TOKEN"
```

The answer is `202`, and the run shows under **Last run** once it finishes.

## See which features run

> **Included free** on every install.

Every screen in the console belongs to a feature. A feature that is not running for your
tenant shows a locked card or no screen at all.

<div class="paths">
<div data-path="console">

1. Open **Operations > Plugins**. Each feature has a card with its state: **Running**,
   **Not enabled**, **Not started**, **Failed** or **Not in this build**.
2. Narrow the list by name, by **Category** or by **State**.
3. On a feature that is not enabled, **How to enable it** opens the page that says how.

</div>
<div data-path="api">

```bash
curl http://localhost:3001/api/admin/plugins/status \
  -H "Authorization: Bearer $TOKEN"
```

Each feature carries `compiled`, `entitled`, `requested` and `active`, and a `reason` when it
is off. `GET /api/admin/plugins/running` answers any signed-in role with the features serving
your tenant.

</div>
</div>

When a feature is in this build and enabled but did not start, the page says so at the top.
Its routes answer `404` until it starts.

![The Plugins page on a free install, with paid features marked Not enabled.](./media/jobs-and-features-plugins-free.png)

![The Plugins page on a licensed install, with every feature running.](./media/jobs-and-features-plugins-licensed.png)

**Manage license**, at the top of the page, opens the license screen, which
[lesson 26](customize-the-console.md) covers.

## See what one feature serves

This is a console task. Over the API, the `manifest` in each entry of the status answer holds
the same.

1. On **Operations > Plugins**, open a feature, such as the one for scheduled jobs.
2. **What it serves** lists each route with its **Method**, **Path** and
   **Who can call it**.
3. **Requests sampled** shows the **Average**, **Slowest** and **Allocated** figures from the
   profiler's recent sample, across every tenant.
4. **Settings** opens the feature's settings in the engine configuration.

A feature that is not running shows **Not running** in place of its routes.

![The page of one feature, with its version, its routes and the request figures.](./media/jobs-and-features-plugin-detail.png)

## Next

[20. Rate limits](rate-limits.md) caps how fast callers reach your engine. For depth, read
[Scheduled jobs](../features/scheduled-jobs.md), [Alerts](../features/alerts.md) and
[Licensing and tiers](../concepts/licensing-and-tiers.md#check-what-is-running).
