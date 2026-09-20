# GSP355 - Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab

Google Cloud manual solution for the GSP355 Challenge Lab - migrating a standalone PostgreSQL database to Cloud SQL using Database Migration Service (DMS), promoting the instance, configuring IAM database authentication, and enabling point-in-time recovery (PITR).

> Replace the lab-specific values (e.g. ``, ``, ``, ``) with the values shown in your own lab panel before following along.

## Author

**Aayush Shrestha (Shinux)** - Computer Engineer

**Portfolio:** [aayushshrestha7.com.np](https://aayushshrestha7.com.np/) | **GitHub:** [aayush105](https://github.com/aayush105) | **LinkedIn:** [aayushrestha](https://www.linkedin.com/in/aayushrestha/)

[![Watch on YouTube](https://img.shields.io/badge/Watch_on_YouTube-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://www.youtube.com/@Er.Shinux)

---

## Lab Overview

This challenge lab has four tasks:

1. **Task 1:** Migrate the standalone PostgreSQL `orders` database on `postgresql-vm` to Cloud SQL for PostgreSQL using Database Migration Service.
2. **Task 2:** Promote the migrated Cloud SQL instance to a standalone read/write instance.
3. **Task 3:** Configure Cloud SQL IAM database authentication and grant the lab IAM user `SELECT` access.
4. **Task 4:** Enable and test point-in-time recovery by creating the required PITR clone.

> **Important:** This is a **manual solution** - you do not need to download or execute any GitHub script. Use the lab-provided values for `REGION`, Cloud SQL instance ID, Qwiklabs user account name, and PITR retention days.

---

## ⚠️ About the `Postgres Migration User` Username

Throughout Steps 9–12, you will see:

```text
Username: Postgres Migration User
Password: DMS_1s_cool!
```

> **`Postgres Migration User` is the exact username required by this lab's grader - do NOT change it.** The Database Migration Service connection profile and all SQL permission grants must use this exact username. It does not vary between lab sessions.

**However**, if you are adapting this guide for a different lab or your own environment and _do_ need to change the username, use the free online word replacer tool:

🔗 **[textcompare.io/word-replacer](https://textcompare.io/word-replacer)**

Copy the **📋 Quick Copy SQL blocks** found after Step 12 below, paste them into the tool, replace `Postgres Migration User` with your desired username, then copy the result and run it. This avoids manually editing 45+ occurrences by hand.

---

# TASK 1 - STEP 1: Prepare the PostgreSQL Database

## (This completes Check My Progress #1)

> Do ALL steps below before clicking "Check my progress" the first time.

---

## Step 1 - Enable the required APIs

Open **Cloud Shell** (the regular one, NOT SSH).

```bash
gcloud services enable datamigration.googleapis.com --quiet
gcloud services enable servicenetworking.googleapis.com --quiet
```

Verify (optional):

```bash
gcloud services list --enabled \
  --filter="name:(datamigration.googleapis.com OR servicenetworking.googleapis.com)"
```

You should see:

```text
datamigration.googleapis.com
servicenetworking.googleapis.com
```

---

## Step 2 - SSH into postgresql-vm

Go to: **Navigation menu -> Compute Engine -> VM instances**

Find `postgresql-vm` -> click **SSH**

> Steps 3 through 13 all run INSIDE the SSH terminal on postgresql-vm

---

## Step 3 - Check PostgreSQL version

Run:

```bash
psql --version
```

Then:

```bash
pg_lsclusters
```

You should see something similar to:

```text
Ver Cluster Port Status Owner    Data directory
14  main    5432 online postgres /var/lib/postgresql/14/main
```

In this example, the PostgreSQL version is:

```text
14
```

> Use the PostgreSQL version actually shown on your VM. Do not blindly use `14` if your VM has a different version.

---

## Step 4 - Install pglogical

For PostgreSQL 14:

```bash
sudo apt update
sudo apt install -y postgresql-14-pglogical
```

For another version, replace `14` with the version shown by `pg_lsclusters`.

For example:

```bash
sudo apt install -y postgresql--pglogical
```

---

## Step 5 - Confirm config file paths

```bash
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"
```

Should return:

- `/etc/postgresql/14/main/postgresql.conf`
- `/etc/postgresql/14/main/pg_hba.conf`

---

## Step 6 - Download and apply Google official config files

Still in the SSH terminal, run these 4 commands:

```bash
sudo su - postgres -c "gsutil cp gs://cloud-training/gsp918/pg_hba_append.conf ."
sudo su - postgres -c "gsutil cp gs://cloud-training/gsp918/postgresql_append.conf ."
sudo su - postgres -c "cat pg_hba_append.conf >> /etc/postgresql/14/main/pg_hba.conf"
sudo su - postgres -c "cat postgresql_append.conf >> /etc/postgresql/14/main/postgresql.conf"
```

> These files are from Google's own training bucket (`gs://cloud-training/`) - completely safe.
> They configure `wal_level`, `shared_preload_libraries`, `listen_addresses`, and `pg_hba` access rules exactly as the lab grader expects.

---

## Step 7 - Restart PostgreSQL

Run:

```bash
sudo systemctl restart postgresql
```

Check the service:

```bash
sudo systemctl status postgresql --no-pager
```

Then:

```bash
pg_lsclusters
```

The cluster should show `online`. Verify the listening address:

```bash
sudo -u postgres psql -c "SHOW listen_addresses;"
```

You should see `*`. Verify port 5432:

```bash
sudo ss -lntp | grep 5432
```

---

## Step 8 - Create the pglogical extension

```bash
sudo -u postgres psql
```

Switch to the `postgres` database:

```sql
\c postgres
```

Create the extension:

```sql
CREATE EXTENSION pglogical;
```

Switch to the `orders` database:

```sql
\c orders
```

Create the extension:

```sql
CREATE EXTENSION pglogical;
```

Exit:

```sql
\q
```

---

## Step 9 - Create the migration user

> **⚠️ The username `Postgres Migration User` is fixed for this lab - do NOT change it.**
> If you need to use a different username for a different environment, use the [word-replacer tool](https://textcompare.io/word-replacer) as described at the top of this guide.

```text
Username: Postgres Migration User
Password: DMS_1s_cool!
```

Open PostgreSQL:

```bash
sudo -u postgres psql
```

```sql
CREATE USER Postgres Migration User PASSWORD 'DMS_1s_cool!';
ALTER DATABASE orders OWNER TO Postgres Migration User;
ALTER ROLE Postgres Migration User WITH REPLICATION;
\q
```

---

## Step 10 - Check and fix primary keys

Connect to the `orders` database:

```bash
sudo -u postgres psql orders
```

Check which tables have primary keys:

```sql
SELECT table_name, constraint_name
FROM information_schema.table_constraints
WHERE constraint_type = 'PRIMARY KEY'
AND table_schema = 'public'
ORDER BY table_name;
```

All 5 tables must have a primary key:

- `distribution_centers`
- `inventory_items`
- `order_items`
- `products`
- `users`

If inventory_items is missing, add it:

```sql
ALTER TABLE inventory_items ADD PRIMARY KEY (id);
```

Exit:

```sql
\q
```

---

## Step 11 - Grant migration-user permissions in orders database

Open the `orders` database:

```bash
sudo -u postgres psql orders
```

### pglogical schema permissions:

```sql
GRANT USAGE ON SCHEMA pglogical TO Postgres Migration User;
GRANT ALL ON SCHEMA pglogical TO Postgres Migration User;
GRANT SELECT ON pglogical.tables TO Postgres Migration User;
GRANT SELECT ON pglogical.depend TO Postgres Migration User;
GRANT SELECT ON pglogical.local_node TO Postgres Migration User;
GRANT SELECT ON pglogical.local_sync_status TO Postgres Migration User;
GRANT SELECT ON pglogical.node TO Postgres Migration User;
GRANT SELECT ON pglogical.node_interface TO Postgres Migration User;
GRANT SELECT ON pglogical.queue TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set_seq TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set_table TO Postgres Migration User;
GRANT SELECT ON pglogical.sequence_state TO Postgres Migration User;
GRANT SELECT ON pglogical.subscription TO Postgres Migration User;
```

### public schema permissions:

```sql
GRANT USAGE ON SCHEMA public TO Postgres Migration User;
GRANT ALL ON SCHEMA public TO Postgres Migration User;
GRANT SELECT ON public.distribution_centers TO Postgres Migration User;
GRANT SELECT ON public.inventory_items TO Postgres Migration User;
GRANT SELECT ON public.order_items TO Postgres Migration User;
GRANT SELECT ON public.products TO Postgres Migration User;
GRANT SELECT ON public.users TO Postgres Migration User;
```

### Transfer table ownership to migration user:

```sql
ALTER TABLE public.distribution_centers OWNER TO Postgres Migration User;
ALTER TABLE public.inventory_items OWNER TO Postgres Migration User;
ALTER TABLE public.order_items OWNER TO Postgres Migration User;
ALTER TABLE public.products OWNER TO Postgres Migration User;
ALTER TABLE public.users OWNER TO Postgres Migration User;
```

Exit:

```sql
\q
```

---

## Step 12 - Grant migration-user permissions in postgres database

Open PostgreSQL:

```bash
sudo -u postgres psql
```

Switch to the `postgres` database:

```sql
\c postgres
```

```sql
GRANT USAGE ON SCHEMA pglogical TO Postgres Migration User;
GRANT ALL ON SCHEMA pglogical TO Postgres Migration User;
GRANT SELECT ON pglogical.tables TO Postgres Migration User;
GRANT SELECT ON pglogical.depend TO Postgres Migration User;
GRANT SELECT ON pglogical.local_node TO Postgres Migration User;
GRANT SELECT ON pglogical.local_sync_status TO Postgres Migration User;
GRANT SELECT ON pglogical.node TO Postgres Migration User;
GRANT SELECT ON pglogical.node_interface TO Postgres Migration User;
GRANT SELECT ON pglogical.queue TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set_seq TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set_table TO Postgres Migration User;
GRANT SELECT ON pglogical.sequence_state TO Postgres Migration User;
GRANT SELECT ON pglogical.subscription TO Postgres Migration User;
```

Exit:

```sql
\q
```

---

## 📋 Quick Copy SQL - Steps 9–12 All-in-One

> These blocks contain the **exact same SQL** as Steps 9–12 with no instructions mixed in - one click to copy.
> Use them to paste directly into `psql`, or into the [word-replacer tool](https://textcompare.io/word-replacer) if you need to swap the username.

### Block 1 - Run inside `orders` database

Open with: `sudo -u postgres psql orders`

```sql
-- Step 9: Create migration user
CREATE USER Postgres Migration User PASSWORD 'DMS_1s_cool!';
ALTER DATABASE orders OWNER TO Postgres Migration User;
ALTER ROLE Postgres Migration User WITH REPLICATION;

-- Step 10: Fix primary key if missing
ALTER TABLE inventory_items ADD PRIMARY KEY (id);

-- Step 11: pglogical schema permissions
GRANT USAGE ON SCHEMA pglogical TO Postgres Migration User;
GRANT ALL ON SCHEMA pglogical TO Postgres Migration User;
GRANT SELECT ON pglogical.tables TO Postgres Migration User;
GRANT SELECT ON pglogical.depend TO Postgres Migration User;
GRANT SELECT ON pglogical.local_node TO Postgres Migration User;
GRANT SELECT ON pglogical.local_sync_status TO Postgres Migration User;
GRANT SELECT ON pglogical.node TO Postgres Migration User;
GRANT SELECT ON pglogical.node_interface TO Postgres Migration User;
GRANT SELECT ON pglogical.queue TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set_seq TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set_table TO Postgres Migration User;
GRANT SELECT ON pglogical.sequence_state TO Postgres Migration User;
GRANT SELECT ON pglogical.subscription TO Postgres Migration User;

-- Step 11: public schema permissions
GRANT USAGE ON SCHEMA public TO Postgres Migration User;
GRANT ALL ON SCHEMA public TO Postgres Migration User;
GRANT SELECT ON public.distribution_centers TO Postgres Migration User;
GRANT SELECT ON public.inventory_items TO Postgres Migration User;
GRANT SELECT ON public.order_items TO Postgres Migration User;
GRANT SELECT ON public.products TO Postgres Migration User;
GRANT SELECT ON public.users TO Postgres Migration User;

-- Step 11: Transfer table ownership
ALTER TABLE public.distribution_centers OWNER TO Postgres Migration User;
ALTER TABLE public.inventory_items OWNER TO Postgres Migration User;
ALTER TABLE public.order_items OWNER TO Postgres Migration User;
ALTER TABLE public.products OWNER TO Postgres Migration User;
ALTER TABLE public.users OWNER TO Postgres Migration User;
```

### Block 2 - Run inside `postgres` database

Open with: `sudo -u postgres psql` then `\c postgres`

```sql
-- Step 12: pglogical schema permissions in postgres db
GRANT USAGE ON SCHEMA pglogical TO Postgres Migration User;
GRANT ALL ON SCHEMA pglogical TO Postgres Migration User;
GRANT SELECT ON pglogical.tables TO Postgres Migration User;
GRANT SELECT ON pglogical.depend TO Postgres Migration User;
GRANT SELECT ON pglogical.local_node TO Postgres Migration User;
GRANT SELECT ON pglogical.local_sync_status TO Postgres Migration User;
GRANT SELECT ON pglogical.node TO Postgres Migration User;
GRANT SELECT ON pglogical.node_interface TO Postgres Migration User;
GRANT SELECT ON pglogical.queue TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set_seq TO Postgres Migration User;
GRANT SELECT ON pglogical.replication_set_table TO Postgres Migration User;
GRANT SELECT ON pglogical.sequence_state TO Postgres Migration User;
GRANT SELECT ON pglogical.subscription TO Postgres Migration User;
```

---

## Step 13 - Configure pglogical_output

Before starting the DMS migration, configure PostgreSQL to allow the `pglogical_output` output plugin.

```bash
sudo sed -i "/shared_preload_libraries/a output_plugin_libraries = 'pgoutput, test_decoding, pglogical_output'" /etc/postgresql/14/main/postgresql.conf
```

Verify it was added:

```bash
sudo grep -n "output_plugin_libraries" /etc/postgresql/14/main/postgresql.conf
```

You should see:

```text
output_plugin_libraries = 'pgoutput, test_decoding, pglogical_output'
```

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql
```

Confirm the active value:

```bash
sudo -u postgres psql -c "SHOW output_plugin_libraries;"
```

Must show: `pgoutput, test_decoding, pglogical_output`

> ⚠️ Do **NOT** continue until you see `pglogical_output` in the output above.

---

## Step 14 - Get the internal IP of postgresql-vm

Switch back to **Cloud Shell** (not SSH).

```bash
gcloud compute instances list
```

Note the zone of postgresql-vm. Then get the internal IP:

```bash
gcloud compute instances describe postgresql-vm \
  --format="get(networkInterfaces[0].networkIP)"
```

Example output: 10.138.0.5

> Save this - it is the INTERNAL IP used for the DMS connection profile.

---

> ## NOW CLICK "Check my progress" for Task 1 - Check #1
>
> If everything above was done correctly, it will pass.

---

# TASK 1 - STEP 2: Create Connection Profile and Migration Job

## (This completes Check My Progress #2)

> All done manually in the Google Cloud Console. No scripts.

---

## Step 15 - Create the DMS Connection Profile

Go to: **Google Cloud Console -> Database Migration -> Connection profiles**

Click **CREATE PROFILE**

| Field                   | Value                           |
| ----------------------- | ------------------------------- |
| Source engine           | PostgreSQL                      |
| Connection profile name | Any (e.g. `postgresql-source`)  |
| Hostname / IP           | INTERNAL IP from Step 14        |
| Port                    | 5432                            |
| Username                | `Postgres Migration User`               |
| Password                | `DMS_1s_cool!`                  |
| Region                  | Your lab region (e.g. us-west1) |

Click **Create**

---

## Step 16 - Create the Migration Job

Go to: **Database Migration -> Migration jobs**

Click **CREATE MIGRATION JOB**

| Field                  | Value                             |
| ---------------------- | --------------------------------- |
| Migration Job Name     | Any (e.g. `postgresql-migration`) |
| Source database engine | PostgreSQL                        |
| Destination engine     | Cloud SQL for PostgreSQL          |
| Migration type         | **Continuous** ← VERY IMPORTANT   |

Click **Save & Continue**

---

## Step 17 - Configure the Destination Instance

- Destination type → **Existing instance**
- Destination instance ID → e.g. `postgres51-g3a47` (from your lab panel)

> If you see "private IP is being allocated", wait until **Create & Continue** becomes available.

---

## Step 18 - Configure VPC Peering

- Connectivity method → **VPC peering**
- VPC network → **default**

---

## Step 19 - Select the orders database

- Select **Specific databases**
- Select **orders**

Click **SAVE & CONTINUE**

---

## Step 20 - Test the Migration Job

Click **TEST JOB**

Wait for the test to complete - it must **PASS**.

If it fails, check:

- PostgreSQL is running
- `listen_addresses = *`
- `wal_level = logical`
- `pglogical` is in `shared_preload_libraries`
- `Postgres Migration User` has all permissions (Steps 11 and 12)
- Port 5432 is accessible

---

## Step 21 - Start the Migration

After the test passes, click **CREATE & START** (or **START**).

Wait for status: **Running** or **CDC in progress**

> ## NOW CLICK "Check my progress" for Task 1 - Check #2

---

# TASK 2 - Promote the Cloud SQL Instance

Go to: **Database Migration -> Migration jobs**

Open your migration job → click **PROMOTE** → Confirm

Wait for the job status to become: **Completed**

> ## Click "Check my progress" for Task 2

---

# TASK 3 - Cloud SQL IAM Database Authentication

## Step 22 - Get the external IP of postgresql-vm

In Cloud Shell:

```bash
gcloud compute instances describe postgresql-vm \
  --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
```

> This is the **EXTERNAL IP** - different from the internal IP used in DMS.

---

## Step 23 - Add VM external IP to Cloud SQL authorized networks

Go to: **Cloud SQL → `` → Connections → Networking**

Under Public IP, click **ADD A NETWORK**:

| Field   | Value                                               |
| ------- | --------------------------------------------------- |
| Name    | Any (e.g. `postgres-vm`)                            |
| Network | External IP from above + `/32` (e.g. `34.x.x.x/32`) |

Click **Done** -> **Save**

---

## Step 24 - Create Cloud IAM database user

Go to: **Cloud SQL → `` → Users → Add user account**

Select: **Cloud IAM**

Principal: e.g. `student-03-de8ac48dc571@qwiklabs.net` (use the value from your lab panel)

Click **Add**

---

## Step 25 - Connect to Cloud SQL and grant SELECT

Go to: **Cloud SQL → `` → Overview**

Under "Connect to this instance" → click **Open Cloud Shell**

When prompted for the password, enter:

```text
supersecret!
```

> Copy and paste - the password will not be visible as you type.

```sql
\c orders
```

If asked for the password again, enter:

```text
supersecret!
```

Grant SELECT permission:

Run:

```sql
GRANT SELECT ON TABLE TABLE_NAME
TO "Qwiklabs_User_Account_Name";
```

Replace:

```text
TABLE_NAME
```

Replace `` with the table specified by the lab, and `` with the exact Qwiklabs user account name from your lab panel.

> The lab asks for `SELECT` - do not replace it with `GRANT ALL PRIVILEGES`.

---

## Step 26 - Verify IAM user access

Connect as the IAM user and run:

Run:

```sql
SELECT COUNT(*) FROM TABLE_NAME;
```

Replace `TABLE_NAME` with the exact table used in the lab.

Should return a count number without errors.

> ## Click "Check my progress" for Task 3

---

# TASK 4 - Configure and Test Point-in-Time Recovery

## Step 27 - Enable backups and PITR

Go to: **Cloud SQL → `` → Overview → Edit → Data Protection**

Enable:

- **Point-in-time recovery**

Set transaction log retention to the exact value from your lab panel (e.g. 6 days).

Click **Save**

---

## Step 28 - Record the PITR timestamp BEFORE making changes

In Cloud Shell:

```bash
date -u --rfc-3339=ns | sed -r 's/ /T/; s/\.([0-9]{3}).*/\.\1Z/'
```

Example output:

```
2026-09-07T07:02:15.123Z
```

> ⚠️ **COPY AND SAVE THIS TIMESTAMP** - you will use it in Step 30.

---

## Step 29 - Insert a test row AFTER saving the timestamp

Go to: **Cloud SQL → `` → Overview**

Under "Connect to this instance" → click **Open Cloud Shell**

Password:

```text
supersecret!
```

Connect to `orders`:

```sql
\c orders
```

Password again:

```text
supersecret!
```

```sql
INSERT INTO distribution_centers VALUES (-80.1918, 25.7617, 'Miami FL', 11);
```

Exit:

```sql
\q
```

---

## Step 30 - Create the PITR clone

Back in regular Cloud Shell, set variables:

```bash
export CLOUDSQL_INSTANCE="YOUR_MIGRATED_INSTANCE_ID"
```

Replace `` with the actual instance ID from your lab panel.

```bash
export NEW_INSTANCE_NAME="postgres-orders-pitr"
```

```bash
export TIME_STAMP="YOUR_SAVED_UTC_TIMESTAMP"
```

Replace `` with the timestamp from Step 28. Example:

Example:

```bash
export TIME_STAMP="2026-09-07T07:02:15.123Z"
```

Run the clone:

```bash
gcloud sql instances clone "$CLOUDSQL_INSTANCE" "$NEW_INSTANCE_NAME" \
  --point-in-time "$TIME_STAMP"
```

Wait for it to complete (may take several minutes).

> ⚠️ Do **NOT** delete `postgres-orders-pitr` - the lab scoring system needs it.

> ## Click "Check my progress" for Task 4

---

# Quick Reference Table

| Item                  | Value                  |
| --------------------- | ---------------------- |
| Migration username    | `Postgres Migration User`      |
| Migration password    | `DMS_1s_cool!`         |
| Migration type        | Continuous             |
| Destination type      | Existing instance      |
| Cloud SQL instance ID | From your lab panel    |
| Connectivity method   | VPC peering            |
| VPC network           | default                |
| Cloud SQL password    | `supersecret!`         |
| IAM user              | From your lab panel    |
| PITR clone name       | `postgres-orders-pitr` |

> Get your Region, Zone, VM IPs, and PITR retention days from your current lab session panel - they change every session.

## Author

**Aayush Shrestha (Shinux)** - Computer Engineer

**Portfolio:** [aayushshrestha7.com.np](https://aayushshrestha7.com.np/) | **GitHub:** [aayush105](https://github.com/aayush105) | **LinkedIn:** [aayushrestha](https://www.linkedin.com/in/aayushrestha/)

[![Watch on YouTube](https://img.shields.io/badge/Watch_on_YouTube-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://www.youtube.com/@Er.Shinux)

### ⚠️ Disclaimer

- **This guide and instructions are provided strictly for educational purposes to help you learn Google Cloud services and advance your engineering skills. Please review all commands thoroughly to understand the underlying infrastructure before execution. Always comply with Qwiklabs/Google Cloud Skills Boost Terms of Service and YouTube Community Guidelines. This material is designed to enhance hands-on learning, not to bypass lab challenges.**

### © Credit & Attribution

- **All educational content, lab scenarios, and original resources belong to [Google Cloud Skills Boost](https://www.cloudskillsboost.google/). No copyright infringement is intended. If you are a copyright owner and have concerns, please reach out via direct message for proper attribution or immediate content removal.** 🙏
