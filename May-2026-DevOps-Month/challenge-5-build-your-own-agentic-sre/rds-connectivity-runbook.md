# Database Connectivity Failure Runbook

## Purpose

This runbook is used when the Banking API starts returning HTTP 500 errors due to loss of connectivity between the Lambda function and the PostgreSQL database.

This scenario simulates a production outage caused by a network configuration change, such as an accidental removal of a Security Group rule.

---

# Architecture

```text
Client
  ↓
API Gateway
  ↓
Lambda (challenge5-bank-api)
  ↓
PostgreSQL RDS (challenge5-postgres)
```

---

# Symptoms

The following symptoms may indicate a database connectivity issue:

- API Gateway returns `500 Internal Server Error`
- CloudWatch Alarm `challenge5-bank-api-errors` is in `ALARM`
- Lambda `Errors` metric increases
- Application health endpoint fails
- Lambda logs contain:
  - `Database connectivity failed`
  - `timed out`
  - `Connection refused`

---

# Impact

- Customer requests fail with HTTP 500 responses.
- Application cannot access its database dependency.
- Service degradation or complete outage may occur.

---

# Investigation Procedure

## Step 1 – Verify Application Failure

```bash
curl https://9qugw4smnf.execute-api.ap-southeast-2.amazonaws.com/prod/health
```

Expected failure:

```json
{"message":"Internal Server Error"}
```

---

## Step 2 – Verify CloudWatch Alarm

```bash
aws cloudwatch describe-alarms \
  --alarm-names challenge5-bank-api-errors \
  --query "MetricAlarms[0].StateValue"
```

Expected:

```text
ALARM
```

---

## Step 3 – Review Lambda Logs

```bash
aws logs tail \
  /aws/lambda/challenge5-bank-api \
  --follow \
  --region ap-southeast-2
```

Look for:

```text
Database connectivity failed
timed out
```

or

```text
Connection refused
```

---

## Step 4 – Verify Lambda Errors

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=challenge5-bank-api \
  --statistics Sum \
  --start-time $(date -u -v-30M +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --period 60 \
  --region ap-southeast-2
```

Confirm that Lambda errors have increased.

---

## Step 5 – Verify RDS Status

```bash
aws rds describe-db-instances \
  --db-instance-identifier challenge5-postgres \
  --query "DBInstances[0].DBInstanceStatus" \
  --region ap-southeast-2
```

Expected:

```text
available
```

If the database is available, continue investigating network connectivity.

---

## Step 6 – Obtain Security Group IDs

```bash
aws cloudformation describe-stacks \
  --stack-name challenge-5 \
  --query "Stacks[0].Outputs"
```

Expected:

```text
LambdaSecurityGroup : sg-00e7c9eb09716132a
RdsSecurityGroup    : sg-009d8ac4a3f087388
```

---

## Step 7 – Inspect RDS Security Group

```bash
aws ec2 describe-security-groups \
  --group-ids sg-009d8ac4a3f087388 \
  --region ap-southeast-2
```

Verify that the following rule exists:

```text
Protocol : TCP
Port     : 5432
Source   : sg-00e7c9eb09716132a
```

If this rule is missing, the Lambda function cannot connect to PostgreSQL.

---

# Root Cause Indicators

The incident is likely caused by a Security Group misconfiguration if all the following conditions are true:

- RDS instance status is `available`
- Lambda function is running
- Lambda logs show database connection timeouts
- API returns HTTP 500 errors
- PostgreSQL Security Group rule allowing Lambda access is missing

---

# Root Cause

A Security Group change removed the PostgreSQL ingress rule that allowed traffic from the Lambda Security Group to the RDS instance on TCP port 5432.

This prevented the application from reaching its database dependency and caused customer-facing HTTP 500 errors.

---

# Recovery Procedure

Restore the Security Group rule:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-009d8ac4a3f087388 \
  --protocol tcp \
  --port 5432 \
  --source-group sg-00e7c9eb09716132a \
  --region ap-southeast-2
```

---

# Validation Steps

Verify the API:

```bash
curl https://9qugw4smnf.execute-api.ap-southeast-2.amazonaws.com/prod/health
```

Expected:

```json
{
  "status": "database reachable",
  "host": "challenge5-postgres.c7youakm2e05.ap-southeast-2.rds.amazonaws.com"
}
```

Verify the alarm:

```bash
aws cloudwatch describe-alarms \
  --alarm-names challenge5-bank-api-errors \
  --query "MetricAlarms[0].StateValue"
```

Expected:

```text
OK
```

---

# Post-Incident Actions

1. Document the incident timeline.
2. Identify how the Security Group change occurred.
3. Review change management procedures.
4. Update Infrastructure as Code definitions.
5. Implement preventive controls.

---

# Prevention Recommendations

- Manage Security Groups only through Infrastructure as Code.
- Enable AWS Config rules for Security Group compliance.
- Implement change approval for production network changes.
- Create synthetic monitoring for database connectivity.
- Add automated rollback procedures for unauthorized changes.
- Maintain runbooks for dependency failures and network outages.

---

# Incident Summary

**Service:** Banking API (`challenge5-bank-api`)

**Customer Impact:** HTTP 500 responses.

**Root Cause:** Missing PostgreSQL ingress rule in the RDS Security Group.

**Resolution:** Restored Security Group access from the Lambda Security Group to PostgreSQL on TCP port 5432.

**Recovery Time Objective:** Less than 5 minutes after root cause identification.