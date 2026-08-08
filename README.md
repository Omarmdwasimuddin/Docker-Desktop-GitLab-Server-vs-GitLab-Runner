# GitLab Server vs GitLab Runner — কনসেপ্ট ক্লিয়ার করা

এই ডকুমেন্টে GitLab Server এবং GitLab Runner এই দুইটা জিনিসের মধ্যে পার্থক্য, তাদের কাজ, এবং তারা কীভাবে একসাথে কাজ করে — সব বিস্তারিতভাবে ব্যাখ্যা করা হয়েছে।

---

## ১. মূল পার্থক্য (Overview)

| বিষয় | GitLab Server | GitLab Runner |
|---|---|---|
| **এটা কী** | পুরো GitLab অ্যাপ্লিকেশন — Web UI, Database, Git repository host সব কিছু | একটা lightweight এজেন্ট/প্রোগ্রাম যেটা CI/CD job রান করে |
| **কাজ** | কোড হোস্ট করা, Merge Request, Issue Tracking, Project ম্যানেজমেন্ট, CI/CD পাইপলাইন *ডিফাইন* করা | CI/CD পাইপলাইনে যে job গুলো আছে, সেগুলো *এক্সিকিউট* করা |
| **চলে কোথায়** | নিজস্ব সার্ভার (VM, Docker container, cloud) | GitLab Server এর সাথে বা আলাদা মেশিনে, এমনকি ডেভেলপারের নিজের মেশিনেও |
| **সংখ্যা** | সাধারণত একটাই GitLab instance | একাধিক Runner রাখা যায় (parallel job এর জন্য) |
| **নির্ভরতা** | স্বাধীনভাবে কাজ করে | GitLab Server এর সাথে রেজিস্টার্ড থাকতে হয়, নাহলে কাজ করবে না |

**সহজ কথায়:** GitLab Server হলো "ব্রেইন" — যেখানে সব কোড, প্রজেক্ট, এবং পাইপলাইনের নির্দেশনা (`.gitlab-ci.yml`) থাকে। আর GitLab Runner হলো "হাত-পা" — যে আসলে সেই নির্দেশনা অনুযায়ী কাজ (build, test, deploy) করে দেয়।

---

## ২. GitLab Server — বিস্তারিত

GitLab Server হলো মূল অ্যাপ্লিকেশন যেটা নিচের সব ফিচার প্রোভাইড করে:

- **Git Repository Hosting** — কোড push/pull, branch, merge
- **Web UI / Dashboard** — Project, Issue, Merge Request দেখা ও ম্যানেজ করা
- **CI/CD Pipeline Definition** — `.gitlab-ci.yml` ফাইলের মাধ্যমে পাইপলাইনের স্টেজ, জব ঠিক করা
- **User & Permission Management** — কে কোন প্রজেক্টে অ্যাক্সেস পাবে
- **Database (PostgreSQL)** — সব মেটাডেটা (users, projects, issues) স্টোর করে

আগের ডকুমেন্টে যেভাবে Docker Compose দিয়ে `gitlab/gitlab-ce` ইমেজ রান করানো হয়েছিল, সেটাই আসলে GitLab Server।

> **Important:** GitLab Server নিজে থেকে কোনো CI/CD job *রান* করে না। ও শুধু ঠিক করে দেয় কী কী job রান হবে এবং কোন অর্ডারে — actual execution এর জন্য Runner লাগে।

---

## ৩. GitLab Runner — বিস্তারিত

GitLab Runner হলো একটা ছোট এপ্লিকেশন (Go দিয়ে লেখা) যেটার একমাত্র কাজ:

> GitLab Server থেকে job "পিক আপ" করা, সেটা রান করা, এবং রেজাল্ট (success/fail, log) আবার সার্ভারে ফেরত পাঠানো।

### Runner এর কাজের ফ্লো

```
GitLab Server (.gitlab-ci.yml দেখে job তৈরি করে)
        │
        ▼
   Job Queue-তে যোগ হয়
        │
        ▼
GitLab Runner (poll করে নতুন job আছে কিনা)
        │
        ▼
   Job পিক করে এক্সিকিউট করে (build/test/deploy)
        │
        ▼
   Log ও Result GitLab Server-এ পাঠায়
```

### Runner এর ধরন (Executor)

Runner বিভিন্ন **Executor** দিয়ে job রান করতে পারে:

| Executor | কাজ করে কীভাবে |
|---|---|
| `shell` | Runner যে মেশিনে ইনস্টল করা, সেখানে সরাসরি shell command রান করে |
| `docker` | প্রতিটা job এর জন্য আলাদা Docker container spin up করে |
| `docker+machine` | Auto-scaling এর জন্য, প্রয়োজন অনুযায়ী নতুন VM তৈরি করে |
| `kubernetes` | Kubernetes cluster এর ভিতরে Pod হিসেবে job রান করে |
| `virtualbox` / `parallels` | VM এর ভিতরে job রান করে (isolation বেশি) |

### Runner এর স্কোপ (Scope)

Runner কে GitLab-এর সাথে register করার সময় ঠিক করে দেওয়া যায় ও কোন লেভেলে কাজ করবে:

- **Instance-wide Runner** — পুরো GitLab instance-এর সব প্রজেক্টের জন্য শেয়ারড
- **Group Runner** — একটা নির্দিষ্ট গ্রুপের সব প্রজেক্টের জন্য
- **Project-specific Runner** — শুধু একটা নির্দিষ্ট প্রজেক্টের জন্য (Specific Runner)

---

## ৪. তারা একসাথে কীভাবে কাজ করে (Full Flow)

1. ডেভেলপার কোড `push` করে GitLab Server-এ
2. GitLab Server, প্রজেক্টের `.gitlab-ci.yml` ফাইল পড়ে বুঝে নেয় কী কী **stage** ও **job** আছে (যেমন: `build`, `test`, `deploy`)
3. প্রতিটা job একটা queue-তে যোগ হয়
4. যেসব Runner সেই প্রজেক্টের সাথে **register** করা আছে, তারা GitLab Server-কে বারবার জিজ্ঞেস করে ("polling") — *"আমার জন্য কোনো job আছে?"*
5. Runner একটা job পেলে সেটা তার Executor (docker/shell/kubernetes ইত্যাদি) দিয়ে রান করে
6. Job শেষ হলে output/log আবার GitLab Server-এ ফেরত যায়, আর Pipeline-এর status আপডেট হয় (✅ Passed / ❌ Failed)

---

## ৫. Registration — Runner কীভাবে Server এর সাথে যুক্ত হয়

Runner ব্যবহার করার আগে GitLab Server-এর সাথে **register** করতে হয়, একটা **Registration Token** দিয়ে। উদাহরণ কমান্ড:

```bash
gitlab-runner register
```

এই কমান্ড চালালে যা যা জিজ্ঞেস করে:

| প্রশ্ন | মানে |
|---|---|
| GitLab instance URL | কোন GitLab Server-এর সাথে connect হবে |
| Registration Token | Server থেকে পাওয়া token, যেটা প্রমাণ করে এই Runner authorized |
| Description | Runner-এর নাম (identify করার জন্য) |
| Executor | কোন পদ্ধতিতে job রান হবে (docker, shell, ইত্যাদি) |

একবার register হয়ে গেলে, Runner GitLab Server-এর Dashboard-এ (**Settings → CI/CD → Runners**) দেখা যাবে।

---

## ৬. এক নজরে সারমর্ম

| প্রশ্ন | উত্তর |
|---|---|
| কোনটা কোড হোস্ট করে? | GitLab Server |
| কোনটা পাইপলাইন define করে (`.gitlab-ci.yml`)? | GitLab Server |
| কোনটা আসলে job রান করে (build/test/deploy)? | GitLab Runner |
| একাধিক Runner একসাথে কাজ করতে পারে? | হ্যাঁ, parallel job-এর জন্য |
| Runner ছাড়া শুধু Server দিয়ে CI/CD চলবে? | না — Runner ছাড়া job queue-তে আটকে থাকবে, রান হবে না |
| Runner আলাদা মেশিনে হতে পারে? | হ্যাঁ, এমনকি ডেভেলপারের নিজের ল্যাপটপেও হতে পারে |

**মূল কথা:** GitLab Server হলো কন্ট্রোল সেন্টার যেখানে কোড ও পাইপলাইন-এর নিয়ম থাকে, আর GitLab Runner হলো সেই নিয়ম অনুযায়ী আসল কাজ (execution) করে দেওয়া worker।
