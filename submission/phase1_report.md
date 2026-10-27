# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
47.7
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->249.2<!--/answer--> | <!--answer:vm_warm_median_ms-->213.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1669.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->648.5<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1549.3<!--/answer--> | <!--answer:function_warm_median_ms-->643.6<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```text
Warm-request latency distribution
20 requests per platform; each # represents one request.

Bucket (ms)    VM                           Cloud Run                    Cloud Function
------------------------------------------------------------------------------------------
0-149           (0)                         ### (3)                      ### (3)
150-299        ################### (19)      (0)                          (0)
300-599        # (1)                         (0)                          (0)
600-749         (0)                         ################# (17)       ################# (17)
750+            (0)                          (0)                          (0)

Warm-latency statistics (ms):
VM: min=210.4, median=213.4, p95=293.4, max=455.6
Cloud Run: min=120.4, median=648.5, p95=679.0, max=701.1
Cloud Function: min=119.1, median=643.6, p95=676.8, max=697.3
```

The VM measurements are comparatively stable, while the managed
platforms exhibit greater variation. Network conditions, connection
reuse, request routing and platform scheduling may contribute;
these observations alone cannot identify the exact causes.
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The e2-micro VM creates no additional machines automatically. One, ten or one hundred simultaneous requests must share its existing CPU, memory and application process, potentially causing queuing or failures. Its capacity is constrained by the selected --machine-type=e2-micro. Cloud Run automatically creates additional container instances when concurrency or CPU demand requires them. Its configured maximum is --max-instances=3. The gen2 Cloud Function also scales by starting Cloud Run-backed container instances, with the same configured maximum of three. Both managed services can scale to zero when idle. The maximum is normally enforced, although temporary overshoots are possible; requests beyond available capacity may queue or fail.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM required the most infrastructure work. I created the custom echo-net VPC and echo-subnet, configured firewall rules for HTTP and IAP-based SSH, provisioned an e2-micro instance, granted Artifact Registry access through a service account, installed Docker, and pulled and started the image manually. I also remain responsible for the VM operating system and container lifecycle. Cloud Run reused the existing Docker image and required a service deployment, port and resource configuration, public access, and autoscaling limits. Google manages the underlying servers. The gen2 Function required Python 3.11 source code, an HTTP entry point, dependencies, and deployment settings. Its container build and runtime infrastructure were handled automatically by Google.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```text
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-268693603078.us-central1.run.app
LAST DEPLOYED BY: service-268693603078@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-26T23:16:41.433574Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-268693603078.us-central1.run.app
LAST DEPLOYED BY: alexandre.ouddah@eurecom.fr
LAST DEPLOYED AT: 2026-10-26T23:15:22.509170Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The service listing shows both echo-bot and echo as Cloud Run services. The second-generation HTTP Function is built into a container and executed on Cloud Run infrastructure. Both therefore benefit from managed container instances and automatic scaling. Their warm median latencies are 648.5 ms and 643.6 ms, respectively, a difference of 4.9 ms (0.8%). Their first measured requests took 1669.7 ms and 1549.3 ms. These first-request measurements are only approximations of cold starts: the measurement script cannot guarantee scale-to-zero or isolate startup time from network latency. A single campaign cannot establish a general cold-start ranking. The main deployment distinction remains the packaging process: Cloud Run uses the explicitly built Docker image, whereas the Function accepts Python source and an HTTP entry point, with container building handled automatically.
<!--/answer-->
