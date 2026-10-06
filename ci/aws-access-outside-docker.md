# Plan: AWS access from the runner host only

This is a plan, not an audit: the table comes from a grep of `ci/` at `master` on 2026-10-06, and it does not read each job in depth. The instance metadata settings of the Linux runner pools are defined outside this repository, so the plan lists them as open questions instead of facts.

## The problem in general terms {#the-problem}

A CI job runs code that the author of a pull request controls. The job runs in a container, but some containers get the cloud credentials of the host machine.

A process that holds credentials keeps them in memory, in its environment and sometimes in files. A crash dump, a log line or an uploaded artifact can copy them out.

The credentials of the host have write access to buckets that other jobs and users read. A person with a copy of them can replace a built binary, a cache entry or a report. Downstream jobs and users then consume the replaced object as if CI produced it.

Together: one leak in a container that the pull request controls can become a substituted release artifact.

## Terms {#terms}

- **Host**: the EC2 runner machine. The `praktika` runner process runs on it.
- **Job container**: the container that `praktika` starts from `Job.Config.run_in_docker`, see `ci/praktika/runner.py`.
- **IMDS**: the EC2 instance metadata service at `169.254.169.254`. It gives the instance role credentials to any process that can reach it.
- **AWS access**: any call signed with AWS credentials: S3 read or write, SSM Parameter Store, Secrets Manager, STS, KMS, ECR. Anonymous HTTPS downloads from public buckets are not AWS access.
- **Lock**: one independent control that stops an attack by itself. The target state has at least two locks for each risk.

## How a job container gets AWS credentials today {#how-credentials-reach-containers}

1. **Host network.** `--network=host` gives the container the network stack of the host, so it reaches IMDS with the default hop limit of 1. `ci/defs/job_configs.py` says so for the fast test: `--network=host required for ec2 metadata http endpoint to work`.
2. **Bridge network with hop limit 2 or more.** A container on the default bridge reaches IMDS if the launch template sets `HttpPutResponseHopLimit` to 2 or more. The Linux pool templates are not in this repository (open question 1).
3. **Host Docker socket.** The integration and Keeper stress containers mount `/run` or `/var/run` from the host. The socket accepts writes on a read-only mount, so the container can start a new container with `--network=host` and reach IMDS.
4. **`--privileged`.** A privileged container can mount host devices and leave the container. A privileged container is the host for this threat model.
5. **Environment variables.** `sign_macos_binary.py` assumes `release_signing` in the container and exports `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN` to all child processes.

The `praktika` pre-run (artifact download) and post-run (artifact and report upload, `CIDB` insert) already run on the host. Jobs without `run_in_docker` (release, nightly, hourly and statistics jobs, stress, fuzzers, upgrade, `libFuzzer`, Jepsen, install check, Docker image jobs) already run their AWS calls on the host.

## Jobs that use AWS from a container {#jobs-table}

| Job (config in `ci/defs/job_configs.py`) | Runner | Path to credentials | AWS service | What for | Proposed move |
|---|---|---|---|---|---|
| Build (`common_build_job_config`, `build_clickhouse.py`) | `amd-large`, `arm-large` | host network | S3 `clickhouse-builds` | `sccache` and `clang-tidy` cache: read on PRs, read and write on `master` and release branches | `sccache` server or a signing cache proxy on the host; the container talks to it over a socket |
| Fast test (`fast_test`, `fast_test.py`) | `amd-large` | host network | S3 | `sccache` read | same as Build |
| Unit test bugfix validation (`bugfix_validation_ut_job`) | `amd-medium` | host network, `--privileged` | S3 | `sccache` read | same as Build |
| Toolchain build (`toolchain_build_jobs`) | `arm-large` | host network | S3 | cache; to check if it uploads the toolchain | same as Build; upload as a `praktika` artifact |
| Collect profiles (`collect_clickhouse_profiles_jobs`) | `arm-large` | host network | S3 (to check) | profile download and upload | declare artifacts; drop host network |
| Bugfix validation, functional (`bugfix_validation_ft_pr_jobs`) | `arm-medium` | host network, `--privileged` | S3, SSM | old binary download, `CIDB` and log cluster secrets | host resolves inputs; drop host network |
| ClickBench (`clickbench_jobs`) | `arm-medium` | host network | none found | host network exposes IMDS without a need | drop host network or block IMDS |
| Stateless tests (`common_ft_job_config`, `functional_tests.py`) | `amd-medium`, `arm-medium` | bridge, `--privileged`; IMDS only if hop limit is 2 or more | SSM | `CIDB` connection for test targeting (`find_tests.py`), log cluster URL and password (`log_cluster.py`, `log_export.py`) | host resolves the secrets in pre-run; better, give the log cluster a per-job token |
| Integration tests (`common_integration_test_job_config`) | `amd-medium`, `arm-medium` | host Docker socket, `--privileged` | SSM | `CIDB` connection for test targeting | host computes the targeted test list in pre-run; remove the socket mount (Docker in Docker only) |
| Keeper stress (`keeper_stress_job`) | `arm-large` | host Docker socket, `--privileged` | none found | the socket exposes IMDS without a need | remove the socket mount |
| SQLancer (`sqlancer_master_jobs`, `sqlancer_job.sh`) | `arm-medium` | bridge | S3 (to check) | result upload | declare artifacts |
| Sign macOS binary (`sign_macos_binary_jobs`) | `release-runner` | host network, environment variables | STS, KMS, SSM | assume `release_signing`, sign with a KMS key through PKCS#11, read the Apple notary key | run the KMS PKCS#11 module on the host and give the container only the `p11-kit` socket; run notarization on the host |

Not in the table: jobs in containers that use only anonymous HTTPS to public buckets (performance comparison, vector search stress, parser and storage memory checks, SQL tests). They need no credentials. Their containers still need IMDS blocked (lock 2 below).

## Risks and their locks {#risks}

1. **A crash dump contains credentials.** A core of the server, `sccache`, Python or the AWS CLI holds the keys in memory. Locks: (a) no credentials in any container process; (b) IMDS blocked from containers, so a process cannot fetch keys later; (c) cores encrypted with `ci/defs/public.pem` before upload, see `ci/decrypt-cores.md`; (d) short session duration of the instance role. Lock (c) alone is weak: the encryption code comes from the PR checkout, the core is on the runner disk unencrypted before encryption, and the holders of `private-cores.pem` can read the keys.
2. **Leaked credentials replace a built binary.** A writer to `clickhouse-builds` can replace a binary that a later job, the install script or a user downloads. Locks: (a) the `untrusted_runner` role has no write access to build and cache prefixes; (b) PR and trusted jobs never share a runner pool or a role; (c) the host records the SHA-256 of each artifact in the job result and consumers check it; (d) release artifacts are signed, and S3 versioning or Object Lock is on for release prefixes. `SCCACHE_S3_RW_MODE=READ_ONLY` in `build_clickhouse.py` is not a lock: the PR can delete the line. IAM must give the same result.
3. **Cache poisoning.** A job that writes to `sccache`, `clang-tidy-cache` or Docker images changes the output of later trusted builds. Locks: as in risk 2, plus separate cache prefixes for each trust level and a cache key that includes the toolchain digest.
4. **The host process runs PR code.** `praktika` on the host imports `ci/defs`, `ci/workflows` and job hooks from the checkout. If the host runs PR code, a move out of the container stops leaks from crashes, but not from a hostile pull request. Locks: (a) the host loads `praktika` from the staged package and the job configuration from the base branch; (b) PR runners get a role that is safe to leak.
5. **Secrets in logs and files.** `praktika` prints the full `docker run` command. A secret given with `-e` appears in the log. `sign_macos_binary.py` writes the notary key to `ci/tmp`, and `release_job.py` writes `~/.r2_auth`. Locks: secrets go through a file mount outside the workspace; artifacts never include `ci/tmp` as a whole; the log masks known secret values.
6. **Runners are reused.** If a runner is not ephemeral, a job can leave a process or a modified image for the next job. Locks: ephemeral runners (open question 2); image pull by digest.
7. **The instance role is too wide.** One instance profile serves all jobs on a pool. Locks: one role for each trust level; session policies for each job, made on the host by `praktika` with `sts:AssumeRole`.

## Plan {#plan}

1. **Verify the facts.** On each pool, run `curl -X PUT http://169.254.169.254/latest/api/token` from a bridge container and from a `--network=host` container. Record the hop limit, the role and its policy. Confirm which roles can write to `clickhouse-builds`.
2. **Block IMDS from containers (lock 2 for all jobs).** Set hop limit 1 on all launch templates. Add an `iptables` rule in the `DOCKER-USER` chain that drops traffic to `169.254.169.254` from Docker bridges. Remove `--network=host` from jobs that need it only for IMDS.
3. **Move S3 transfers to the host.** Job scripts write outputs to `ci/tmp` and declare them as `praktika` artifacts. Inputs come from `requires`. The host does all signed S3 calls in pre-run and post-run.
4. **Move secret reads to the host.** `Job.Config` lists the secrets of the job. The host resolves them and mounts them read-only from outside the workspace. Where a secret is a credential for a service that CI owns (`CIDB`, log cluster), issue a short-lived token for each job.
5. **Run caches on the host.** Run the `sccache` server on the host with the role credentials. The container connects through a socket and has no credentials. Alternative: a host-local signing proxy, as `ci/praktika/infrastructure/native/s3_proxy_user_data.sh` does for reports.
6. **Move signing to the host.** The host assumes `release_signing` and exposes the KMS PKCS#11 token through a `p11-kit` server socket. The container signs with the socket and never sees the keys.
7. **Remove the host Docker socket** from the integration and Keeper stress containers. They already run their own daemon (`docker_in_docker.sh`).
8. **Add the artifact integrity lock.** Record digests in the job result on the host and check them in consumers.
9. **Enforce.** Make `praktika` refuse a `run_in_docker` with `--network=host` or a host socket mount unless the job is on an allow list.

## Open questions {#open-questions}

1. What is `HttpPutResponseHopLimit` for each Linux pool? The stateless tests read SSM from a bridge container. Either the hop limit is 2 or more, or these reads fail.
2. Are all Linux runners ephemeral?
3. Does the host `praktika` process import code from the PR checkout?
4. Which roles can write to `clickhouse-builds`, and are PR and `master` jobs on separate roles?
