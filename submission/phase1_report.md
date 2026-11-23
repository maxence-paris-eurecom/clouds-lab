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

<!--answer:image_size_mb-->50<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->echo-net<!--/answer-->

<!--answer:subnet_name-->echo-subnet<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->240.3<!--/answer--> | <!--answer:vm_warm_median_ms-->232.8<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2814.9<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->186.7<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->3456.1<!--/answer--> | <!--answer:function_warm_median_ms-->170.7<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
Phase 1 — deployment measurements (cold / warm median, ms)
vm        cold=  240.3  warm_median=  232.8   ########################################
cloudrun  cold= 2814.9  warm_median=  186.7   ################################
function  cold= 3456.1  warm_median=  170.7   #############################
```
Warm min/median/p95/max: VM 230.9/232.8/233.3/234.4, Cloud Run 167.6/186.7/197.7/208.0, Function 166.1/170.7/230.0/272.1. The VM is the slowest warm because its request goes over plain HTTP to a single e2-micro in us-central1 with no edge proxy; the cold gap on Cloud Run and the Function is about 2.6 s and 3.3 s.
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

<!--answer:scaling_behaviour-->VM: nothing is created automatically. Ten or a hundred concurrent requests all hit the one e2-micro container, so the hard limit is the size of the machine I picked (2 shared vCPUs, 1 GB RAM) plus gunicorn's 1 worker x 8 threads. Cloud Run: new container instances are created automatically, up to the --max-instances=3 flag I passed myself, each handling concurrent requests; it scales back to zero when idle. Cloud Function (gen 2): it is the same mechanism, since a gen-2 function is a Cloud Run service built from my source; the platform adds instances on demand up to its own default maximum, which I did not set.<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->VM (most effort): I built a custom VPC, subnet and two firewall rules (8080 and 22, because a custom VPC has no default SSH rule), created the instance with a startup script that installs Docker, logs in to Artifact Registry and runs the container, and I would have to patch the OS and restart it myself. Cloud Run: one command once the image exists, but I had to write the Dockerfile, build the image with Cloud Build and push it to Artifact Registry; scaling, TLS and the URL are managed. Cloud Function: the least effort, as I only gave it the source folder and an entry point (main.py); Google builds the container, but I had to grant the build service account the right roles on a new project.<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
   SERVICE    REGION       URL                                                 LAST DEPLOYED BY                                              LAST DEPLOYED AT
✔  echo       us-central1  https://echo-790423926593.us-central1.run.app       service-790423926593@gcf-admin-robot.iam.gserviceaccount.com  2026-11-23T19:11:58.099476Z
✔  echo-bot   us-central1  https://echo-bot-790423926593.us-central1.run.app   maxence.eurecom@gmail.com                                     2026-11-23T19:11:07.840415Z
✔  mr-worker  us-central1  https://mr-worker-790423926593.us-central1.run.app  service-790423926593@gcf-admin-robot.iam.gserviceaccount.com  2026-11-23T19:16:45.364346Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->The list shows my function `echo` as an ordinary Cloud Run service, next to `echo-bot`, with the same URL pattern (*.run.app). Its last deployer is the gcf-admin-robot service account: Google deployed it as Cloud Run on my behalf. A gen-2 Cloud Function is therefore a Cloud Run service whose container Google builds from my source with buildpacks, so both run on the same infrastructure and pay the same cold start (pull the image, start the container, boot gunicorn/Flask): 2815 ms and 3456 ms, far above the warm 187 ms and 171 ms. What genuinely differs is the deployment path: for Cloud Run I build and push the image and own the Dockerfile; for the function Google builds it and I only give source plus an entry point, which can add a little startup time (the functions framework, extra dependencies) but not a different execution model.<!--/answer-->
