# ESRally in Docker — Random Vector Track

Run ESRally benchmarks against an existing Elastic Cloud cluster using Docker and the `random_vector` track from the official [rally-tracks](https://github.com/elastic/rally-tracks) repository.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Docker Desktop | Elastic org sign-in required (see below) |
| Elastic cluster | URL + API key in `variables.env` |
| `rally-tracks` repo | Cloned locally (see step 2) |

### 1. Docker Desktop — Elastic Org Sign-In

Your organisation enforces Docker Desktop org membership. Before pulling `elastic/rally`, sign in:

1. Open **Docker Desktop → Sign In**
2. Use your Elastic SSO account (e.g. `yourname@elastic.co`)
3. Verify: `docker pull elastic/rally` succeeds

### 2. Clone rally-tracks

Sparse-clone only the `random_vector` track to save bandwidth:

```bash
git clone --depth=1 --filter=blob:none --sparse \
  https://github.com/elastic/rally-tracks.git \
  ~/Documents/GitHub/rally-tracks

cd ~/Documents/GitHub/rally-tracks
git sparse-checkout set random_vector
git checkout
```

Or clone the full repo if you plan to use other tracks:

```bash
git clone https://github.com/elastic/rally-tracks.git \
  ~/Documents/GitHub/rally-tracks
```

### 3. Cluster Credentials

`variables.env` is gitignored (contains secrets). Copy the template and fill in your values:

```bash
cp variables.env.example variables.env
```

Then edit `variables.env`:

```bash
ES_URL=https://your-cluster-id.es.your-region.aws.elastic.cloud
API_KEY=your-base64-encoded-api-key
```

To create an API key in Kibana: **Stack Management → API Keys → Create API key** — grant `write` and `read` privileges on the `vectors-benchmark-*` index pattern.

Load into your shell before running any Docker commands:

```bash
source variables.env
```

---

## Directory Structure

```
rally-docker/
├── variables.env          # Cluster URL and API key
├── README.md
├── params/                # Example track param files (mount into container)
│   ├── quick-test.json    # Minimal sanity-check run
│   ├── default-128d.json  # Track defaults, 128-dim vectors
│   ├── bbq-flat-1024d.json  # 1024-dim with bbq_flat (nightly)
│   ├── bbq-disk-1024d.json  # 1024-dim with bbq_disk (nightly)
│   └── high-throughput.json # High-concurrency stress test
└── myrally/               # Created at runtime — Rally state, logs, results
```

---

## Running ESRally in Docker

### Volume Mounts

| Host path | Container path | Purpose |
|---|---|---|
| `~/Documents/GitHub/rally-tracks` | `/rally/tracks` | Track source files |
| `./params` | `/rally/params` | Param JSON files |
| `./myrally` | `/rally/.rally` | State, logs, results (persisted) |

### Target Host Format

Strip `https://` from `ES_URL` and append `:443`:

```
demo-c4ecc8.es.us-east-1.aws.elastic.cloud:443
```

Or derive it in-shell:

```bash
ES_HOST=$(echo "$ES_URL" | sed 's|https://||'):443
```

### Base Docker Command

```bash
source variables.env

TRACKS_DIR=~/Documents/GitHub/rally-tracks
ES_HOST=$(echo "$ES_URL" | sed 's|https://||'):443

docker run --rm \
  -v "${TRACKS_DIR}:/rally/tracks:ro" \
  -v "$(pwd)/params:/rally/params:ro" \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="dims:128,vector_index_type:bbq_flat"
```

> **Note:** `myrally/` must exist before running. Create it with `mkdir -p myrally`. Rally stores logs in `myrally/logs/` and results in `myrally/benchmarks/`.

---

## Random Vector Track

The `random_vector` track benchmarks **filtered KNN vector search** using randomly generated vectors organised into a **multi-partition** model.

### How It Works

Each document contains:
- A random vector with `dims` dimensions (field: `emb`)
- A `partition_id` assigned via weighted random sampling across three tiers

**Partition tiers** (configurable counts):

| Tier | Documents per partition | Default count |
|---|---|---|
| small | 1,000 – 10,000 | 100 |
| medium | 10,000 – 100,000 | 20 |
| large | 100,000 – 1,000,000 | 5 |

**Benchmark phases:**
1. **Indexing** — bulk-insert random vectors across all partitions
2. **Search** — run filtered KNN queries, separately measured per tier:
   - `small-partition-search`
   - `medium-partition-search`
   - `large-partition-search`

### All Track Parameters

| Parameter | Default | Description |
|---|---|---|
| `dims` | `128` | Vector dimensions. Use `1024` for nightly runs. |
| `vector_index_type` | `"bbq_flat"` | Index type: `bbq_flat`, `bbq_disk`, `hnsw`, `int8_hnsw`, `flat` |
| `number_of_shards` | `1` | Index shard count |
| `number_of_replicas` | `0` | Replica count |
| `index_clients` | `1` | Parallel indexing clients |
| `index_iterations` | `1000` | Bulk requests per client |
| `index_bulk_size` | `1000` | Documents per bulk request |
| `index_target_throughput` | _unset_ | Rate-limit indexing (bulk req/sec). Omit = max speed. |
| `search_iterations` | `10000` | Search queries per phase |
| `search_clients` | `8` | Parallel search clients |
| `small_partitions` | `100` | Number of small-tier partitions |
| `medium_partitions` | `20` | Number of medium-tier partitions |
| `large_partitions` | `5` | Number of large-tier partitions |
| `paragraph_size` | `1` | Vectors per document. `>1` uses nested field `nested.emb`. |
| `custom_routing` | `false` | Route by `partition_id` for co-location |
| `rescore_oversample` | `-1` | `-1`=index default, `0`=disable, `>0`=explicit oversample |
| `enable_experimental_features` | `false` | Enable experimental dense vector features |
| `index_mode` | _unset_ | e.g. `"vectordb_document"` |
| `post_ingest_sleep` | `false` | Sleep after indexing (for serverless) |
| `post_ingest_sleep_duration` | `30` | Seconds to sleep if `post_ingest_sleep=true` |

**Total documents indexed:**
```
index_clients × index_iterations × index_bulk_size
```

Default: `1 × 1000 × 1000 = 1,000,000 documents`

---

## Example Runs

### Inline Params

Pass params directly on the command line with comma-separated `key:value` pairs.

**Quick sanity check:**
```bash
source variables.env
ES_HOST=$(echo "$ES_URL" | sed 's|https://||'):443

docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="dims:128,vector_index_type:bbq_flat,index_iterations:5,index_bulk_size:100,search_iterations:100,search_clients:1,small_partitions:5,medium_partitions:2,large_partitions:1"
```

**Nightly benchmark (bbq_flat, 1024-dim):**
```bash
docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="dims:1024,vector_index_type:bbq_flat"
```

**Nightly benchmark (bbq_disk, 1024-dim):**
```bash
docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="dims:1024,vector_index_type:bbq_disk"
```

---

### JSON Params Files

Store params in a JSON file and mount it into the container. This is preferred for repeatability and sharing.

**Syntax:** `--track-params="/rally/params/filename.json"`

#### params/quick-test.json — Sanity check

Minimal run: 500 total docs, 100 searches. Use this to verify Docker + cluster connectivity before a full run.

```bash
source variables.env
ES_HOST=$(echo "$ES_URL" | sed 's|https://||'):443

docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/params:/rally/params:ro" \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="/rally/params/quick-test.json"
```

Params: `dims=128, index_iterations=5, index_bulk_size=100, search_iterations=100, search_clients=1, small_partitions=5, medium_partitions=2, large_partitions=1`

---

#### params/default-128d.json — Track defaults

Matches the track's out-of-the-box defaults. Indexes 1M documents, runs 10K searches per tier.

```bash
docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/params:/rally/params:ro" \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="/rally/params/default-128d.json"
```

Params: `dims=128, vector_index_type=bbq_flat, index_iterations=1000, index_bulk_size=1000, search_iterations=10000, search_clients=8`

---

#### params/bbq-flat-1024d.json — Nightly (bbq_flat)

1024-dimension brute-force flat index. Recommended nightly config per track docs.

```bash
docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/params:/rally/params:ro" \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="/rally/params/bbq-flat-1024d.json"
```

Params: `dims=1024, vector_index_type=bbq_flat, index_iterations=1000, search_iterations=10000, search_clients=8`

---

#### params/bbq-disk-1024d.json — Nightly (bbq_disk)

1024-dimension disk-based index. Compare with `bbq_flat` to measure disk vs memory performance.

```bash
docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/params:/rally/params:ro" \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="/rally/params/bbq-disk-1024d.json"
```

Params: `dims=1024, vector_index_type=bbq_disk, index_iterations=1000, search_iterations=10000, search_clients=8`

---

#### params/high-throughput.json — Stress test

High-concurrency run with 3 shards, 4 index clients, 16 search clients. Measures system limits under load.

```bash
docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/params:/rally/params:ro" \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="/rally/params/high-throughput.json"
```

Params: `dims=128, number_of_shards=3, number_of_replicas=1, index_clients=4, index_iterations=2000, search_clients=16, search_iterations=20000`

---

## Overriding Individual Params

Combine a JSON file with inline overrides — inline values take precedence:

```bash
# Use default-128d.json but override search iterations only
docker run --rm \
  -v ~/Documents/GitHub/rally-tracks:/rally/tracks:ro \
  -v "$(pwd)/params:/rally/params:ro" \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally race \
  --track-path=/rally/tracks/random_vector \
  --pipeline=benchmark-only \
  --target-hosts="${ES_HOST}" \
  --client-options="use_ssl:true,verify_certs:true,api_key:'${API_KEY}'" \
  --track-params="/rally/params/default-128d.json" \
  --track-params="search_iterations:500"
```

---

## Viewing Results

Rally writes results to `myrally/benchmarks/` after each run.

**List race results:**
```bash
docker run --rm \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally list races
```

**Compare two races:**
```bash
docker run --rm \
  -v "$(pwd)/myrally:/rally/.rally" \
  elastic/rally compare \
  --baseline=<race-id-1> \
  --contender=<race-id-2>
```

**View logs:**
```bash
ls myrally/logs/
```

---

## Docker Image Versions

Use `elastic/rally:latest` or pin to a specific version for reproducibility:

```bash
# Latest
docker pull elastic/rally

# Pinned version
docker pull elastic/rally:2.13.0
```

Check available tags: https://hub.docker.com/r/elastic/rally/tags

### Custom Image (optional)

If you need to pre-bake configuration or install extra dependencies:

```dockerfile
FROM elastic/rally:2.13.0
COPY --chown=1000:0 rally.ini /rally/.rally/
```

```bash
docker build -t my-rally .
docker run --rm my-rally list tracks
```

---

## Limitations

- **Pipeline:** Only `benchmark-only` is supported in Docker (cannot provision Elasticsearch).
- **Load drivers:** Cannot distribute load across multiple machines from Docker.
- **Numpy:** Required by the random_vector track — pre-installed in the `elastic/rally` image.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Sign in to continue using Docker Desktop` | Sign in to Docker Desktop with your Elastic org account |
| `cluster_health_timeout` | Cluster may be asleep/unreachable — verify ES_URL is accessible |
| `api_key` auth rejected | Re-check `API_KEY` value in `variables.env`; key must have write+read perms |
| `No module named numpy` | Use the official `elastic/rally` image — it includes numpy |
| `track-path not found` | Verify the rally-tracks mount path and that `random_vector/track.json` exists |
| Results not persisting | Ensure `myrally/` exists on the host before running |
