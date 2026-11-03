# Taming Node.js Logs in Production: A Principal Architect’s Guide to PM2 Log Rotation

It happens to the best of us. You deploy a flawless Node.js application, monitor it for a few days, and everything looks golden. The CPU usage is flat, memory is stable, and response times are lightning-fast. Fast forward six months, and you get a pager alert at 3 AM: *No space left on device*.

The application is down. The database crashed. And the culprit? A massive, unwieldy `~/.pm2/logs` folder.

In my role overseeing cloud architectures at Zealous System, I’ve seen this exact scenario play out across countless infrastructures. PM2 is phenomenal for process management, but its default logging behavior will eventually eat your server's disk space alive if left unchecked. Instead of manually clearing out log files in a panic during an outage, you can automate the entire lifecycle using the `pm2-logrotate` module.

Below, we’ll explore the precise configuration I deploy to safeguard production workloads from sudden storage outages—keeping system architecture robust and overhead costs predictably low.

### How to Automate and Configure PM2 Log Rotation

Here is the exact, battle-tested configuration I use to keep production environments stable.

#### 1. Install the Log Rotation Module

First, you need to pull in the official PM2 module. This runs as its own background process to monitor your application logs.

```bash
pm2 install pm2-logrotate

```

#### 2. Standardize Your Timestamps

Finding the right log file during an incident investigation is stressful enough. Make your file names predictable by setting a clear, human-readable date format.

```bash
pm2 set pm2-logrotate:dateFormat YYYY-MM-DD_HH-mm-ss

```

#### 3. Force Strict Daily Rotations

By default, logs might rotate at awkward times based on uptime. I prefer forcing a strict rotation every day at midnight using standard cron syntax. This way, you always know exactly which file covers which day.

```bash
pm2 set pm2-logrotate:rotateInterval "0 0 * * *"

```

#### 4. Cap Your Retention History

You rarely need application logs from six months ago sitting on expensive SSD storage. Keep at most 30 rotated log files (a month's worth) plus the active one. PM2 will automatically delete the oldest files to free up space.

```bash
pm2 set pm2-logrotate:retain 30

```

#### 5. Prevent Mid-Day Log Splitting

By default, logrotate might split a file if it hits a relatively small size limit. This scatters your daily logs across multiple files and makes debugging a nightmare. To prevent size-based triggers from splitting logs mid-day, set the threshold artificially high.

```bash
pm2 set pm2-logrotate:max_size 10G

```

#### 6. Compress the Archives

Text files compress incredibly well. Enable gzip compression for all rotated files to drastically reduce your storage footprint.

```bash
pm2 set pm2-logrotate:compress true

```

---

### Why This is Mandatory: The Architectural and Financial Realities

When you are architecting systems at scale, leaving PM2 to its default logging behavior isn't just an oversight—it is an active threat to your infrastructure's stability and your cloud budget.

#### The System Architecture Perspective

From an architectural standpoint, unmanaged logs are a ticking time bomb. Node.js writes application logs sequentially. When PM2 appends to a single, monolithic `.log` file that has grown to 50GB or 100GB, every `console.log` operation begins to incur severe I/O latency. This disk bottleneck will silently drag down your application's request throughput.

More critically, when that log file eventually exhausts your server's disk space, the failure cascade is catastrophic. The OS can no longer write swap files, databases running on the same volume (like Redis or PostgreSQL) will immediately crash, and PM2 itself will fail to spawn new child processes. By rotating and restricting the file size, you guarantee bounded disk usage, ensuring your application never suffers a self-inflicted Denial of Service (DoS).

#### The Costing Perspective

Cloud storage is optimized for performance, not for hoarding raw text. If you are deploying on AWS, a standard gp3 Elastic Block Store (EBS) volume costs $0.08 per GB, per month.

Consider a moderately trafficked API that generates 10GB of logs per day (300GB per month). Without log rotation, your PM2 logs accumulate linearly. By month 12, you are paying to store 3.6 Terabytes of completely useless, stale text data on premium solid-state drives.

Implementing the configuration above changes the financial math entirely. By retaining only 30 days of logs and enabling gzip compression (which typically reduces raw JSON or text log sizes by 85% to 90%), you cap your storage ceiling. Instead of paying for exponential growth, your log storage costs become a flat, predictable line item.
