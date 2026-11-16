Here's your full hands-on lab. Let's start with a visual overview of what you'll build, then go step by step.---

# 🧪 Lab: Detecting Protobuf/gRPC Breaking Changes with Buf CLI

> **Goal**: You will author a gRPC service contract in Protobuf, snapshot it as a baseline, introduce breaking changes, and use `buf breaking` to catch them — both locally and inside a CI pipeline.

---

## 🗺️ Lab Map

| Phase | What you do |
|---|---|
| **0** | Install prerequisites |
| **1** | Scaffold the project |
| **2** | Write your v1 proto |
| **3** | Configure Buf |
| **4** | Build & lint |
| **5** | Snapshot the baseline |
| **6** | Introduce breaking changes |
| **7** | Run breaking detection |
| **8** | Fix one change, test again |
| **9** | Wire into GitHub Actions CI |

---

## Phase 0 — Install Prerequisites

> 📝 **Note**: Buf CLI is a single static binary. No Node, no Python, no Java needed.

```bash
# ── macOS (Homebrew) ─────────────────────────────────────
brew install bufbuild/buf/buf

# ── Linux (curl install) ─────────────────────────────────
BUF_VERSION="1.34.0"
curl -sSL \
  "https://github.com/bufbuild/buf/releases/download/v${BUF_VERSION}/buf-Linux-x86_64" \
  -o /usr/local/bin/buf
chmod +x /usr/local/bin/buf

# ── Windows (Scoop) ──────────────────────────────────────
scoop install buf

# ── Verify installation ───────────────────────────────────
buf --version
# Expected: 1.34.0 (or newer)
```

> ⚠️ **Reminder**: Always pin a specific `BUF_VERSION` in CI. Floating `latest` installs can break your pipeline when Buf ships a minor with different lint defaults.

---

## Phase 1 — Scaffold the Project

```bash
# Create project root
mkdir buf-breaking-lab && cd buf-breaking-lab

# Create directory structure
mkdir -p proto/v1
mkdir -p .github/workflows   # for Phase 9

# Confirm layout
tree .
# .
# ├── .github/
# │   └── workflows/
# └── proto/
#     └── v1/
```

> 📝 **Why `proto/v1/`?** Versioning your proto path is a Buf best practice. When you release v2, you add `proto/v2/` — old clients stay on the old import path without a big migration.

---

## Phase 2 — Write Your v1 Protobuf Contract

This is your **stable API contract** — the baseline clients are compiled against.

```bash
cat > proto/v1/user.proto << 'EOF'
syntax = "proto3";

package lab.user.v1;

option go_package = "github.com/yourorg/buf-breaking-lab/gen/go/user/v1;userv1";

// UserService handles CRUD operations for users.
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
}

message User {
  string id          = 1;
  string email       = 2;
  string full_name   = 3;
  int32  age         = 4;
  bool   is_active   = 5;
  Role   role        = 6;
}

enum Role {
  ROLE_UNSPECIFIED = 0;
  ROLE_VIEWER      = 1;
  ROLE_EDITOR      = 2;
  ROLE_ADMIN       = 3;
}

message GetUserRequest  { string id = 1; }
message GetUserResponse { User user = 1; }

message CreateUserRequest {
  string email     = 1;
  string full_name = 2;
  int32  age       = 3;
  Role   role      = 4;
}
message CreateUserResponse { User user = 1; }

message ListUsersRequest  { int32 page_size = 1; string page_token = 2; }
message ListUsersResponse {
  repeated User users         = 1;
  string        next_page_token = 2;
}
EOF
```

> 📝 **Proto field numbers are forever.** Field number `1` in `User` will ALWAYS mean `id` on the wire. If you remove field 1 and reuse it for something else, old serialized messages become corrupted. Buf catches this.

---

## Phase 3 — Configure Buf

```bash
cat > buf.yaml << 'EOF'
version: v2

# The module tells Buf which directories contain proto files.
modules:
  - path: proto

# Lint rules — enforce style and naming conventions.
lint:
  use:
    - DEFAULT        # opinionated but widely-adopted Buf defaults

# Breaking change detection rules.
# FILE  = checks field-level wire compatibility (safe default for most teams)
# WIRE  = stricter: also checks json_name, default values, etc.
# WIRE_JSON = strictest: catches anything that breaks JSON serialization too
breaking:
  use:
    - FILE
EOF
```

> 📝 **Breaking rule sets explained:**
> - `FILE` — prevents removing fields, changing types, changing field numbers, removing messages/enums/RPCs. Best for most gRPC teams.
> - `WIRE` — everything in FILE + wire-format JSON name changes.
> - `WIRE_JSON` — everything in WIRE + JSON serialization compat. Use when clients use `grpc-gateway` or `Connect` with JSON.
> - `PACKAGE` — checks stability at the package level, not the file level. Rarely needed.

---

## Phase 4 — Build & Lint Your Protos

```bash
# ── Build: compile and validate all proto files ───────────
buf build

# If this passes silently — congratulations, your proto is valid.
# Any syntax errors or import problems are printed here.

# ── Lint: enforce style rules ─────────────────────────────
buf lint

# Expected: no output (exit 0 = all good)
```

> ⚠️ **Common lint errors you may hit:**
> - `ENUM_VALUE_PREFIX` — enum values must be prefixed with the enum name (e.g. `ROLE_ADMIN`, not just `ADMIN`) ✅ already done above.
> - `SERVICE_SUFFIX` — service name must end in `Service` ✅ already done.
> - `PACKAGE_VERSION_SUFFIX` — package must end in `v1`, `v2`, etc. ✅ already done (`lab.user.v1`).

---

## Phase 5 — Snapshot the Baseline (Two Methods)

You need a "before" image to compare against. Buf supports two approaches:

### Method A — Git commit baseline (recommended for repos)

```bash
# Initialize a git repo and commit your v1
git init
git add .
git commit -m "feat: v1 UserService proto contract"

# Now Buf will compare HEAD vs this commit automatically.
# We'll use .git#branch=main as the baseline in Phase 7.
```

### Method B — File snapshot baseline

```bash
# Export a compiled binary image of your protos
buf build --output v1.binpb

# This is a FileDescriptorSet binary — Buf can diff against it later.
# Useful when the "old" version lives in a different repo or is published to BSR.
ls -lh v1.binpb
# -rw-r--r-- 1 user staff 1.2K  v1.binpb
```

> 📝 **Which to use?** Use the git approach day-to-day. Use the `.binpb` snapshot when comparing across monorepo boundaries, when the baseline is published to the **Buf Schema Registry (BSR)**, or in polyglot CI where cloning the full git history is slow.

---

## Phase 6 — Introduce Breaking Changes

Now you'll deliberately break the contract to see what Buf catches.

```bash
cat > proto/v1/user.proto << 'EOF'
syntax = "proto3";

package lab.user.v1;

option go_package = "github.com/yourorg/buf-breaking-lab/gen/go/user/v1;userv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  // BREAKING CHANGE #1: ListUsers RPC removed entirely
}

message User {
  string id          = 1;
  string email       = 2;
  // BREAKING CHANGE #2: full_name (field 3) removed
  string display_name = 7;  // added new field — this is fine
  // BREAKING CHANGE #3: age type changed from int32 → string
  string age         = 4;
  bool   is_active   = 5;
  Role   role        = 6;
}

enum Role {
  ROLE_UNSPECIFIED = 0;
  ROLE_VIEWER      = 1;
  ROLE_EDITOR      = 2;
  ROLE_ADMIN       = 3;
  // BREAKING CHANGE #4: reserving nothing — but we'll add a new value
  ROLE_SUPERADMIN  = 4;   // Adding enum value — safe in proto3
}

message GetUserRequest  { string id = 1; }
message GetUserResponse { User user = 1; }

message CreateUserRequest {
  string email     = 1;
  string full_name = 2;
  int32  age       = 3;
  Role   role      = 4;
}
message CreateUserResponse { User user = 1; }

// BREAKING CHANGE #5: ListUsersRequest and ListUsersResponse removed
EOF
```

> 📝 **What's safe vs breaking in proto3:**
>
> | Change | Breaking? | Why |
> |---|---|---|
> | Add a new field (new number) | ✅ Safe | Old parsers ignore unknown fields |
> | Add a new enum value | ✅ Safe | Treated as 0 (unspecified) by old parsers |
> | Remove a field | ❌ Breaking | Old clients sending it — receivers silently drop data |
> | Change a field's type | ❌ Breaking | Wire encoding changes, parsers corrupt or crash |
> | Remove an RPC | ❌ Breaking | Clients calling it get `UNIMPLEMENTED` at best |
> | Change a field number | ❌ Breaking | Wire field tag mismatch, data misread |
> | Rename a field (same number) | ✅ Safe (wire) ❌ (JSON) | JSON uses field name, proto wire uses number |

---

## Phase 7 — Run Breaking Detection

### Using the git baseline (Method A)

```bash
buf breaking --against '.git#branch=main'
```

**Expected output:**

```
proto/v1/user.proto:1:1:Previously present file "proto/v1/user.proto" had service "UserService" with method "ListUsers" deleted.
proto/v1/user.proto:11:3:Field "3" with name "full_name" on message "User" was deleted.
proto/v1/user.proto:14:3:Field "4" with name "age" on message "User" changed type from "int32" to "string".
proto/v1/user.proto:1:1:Previously present file "proto/v1/user.proto" had message "ListUsersRequest" deleted.
proto/v1/user.proto:1:1:Previously present file "proto/v1/user.proto" had message "ListUsersResponse" deleted.
```

> ✅ **Exit code 1** — Buf exits non-zero when it finds breaking changes. This is what makes CI gates work.

### Using the .binpb snapshot (Method B)

```bash
buf breaking --against v1.binpb
```

> Same output. The `--against` flag accepts: a `.git` ref, a `.binpb` file, a `.tar.gz` archive, a directory path, or a `buf.build/org/repo` BSR reference.

### Suppress false positives with an ignore file

Sometimes you intentionally want to allow a specific change (e.g., a field you reserved but never shipped to clients). Add this to `buf.yaml`:

```yaml
breaking:
  use:
    - FILE
  ignore_only:
    FIELD_SAME_TYPE:
      - proto/v1/user.proto
```

> ⚠️ **Use `ignore_only` sparingly.** It silences the rule globally for that file. If you need per-field exceptions, use `buf:breaking_ignore` inline comments instead:

```protobuf
// buf:breaking_ignore
string age = 4;
```

---

## Phase 8 — Fix One Change, Verify the Rest

Let's fix the `full_name` removal properly — by marking the field reserved — while keeping the other intentional breaks in place:

```bash
cat > proto/v1/user.proto << 'EOF'
syntax = "proto3";

package lab.user.v1;

option go_package = "github.com/yourorg/buf-breaking-lab/gen/go/user/v1;userv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}

message User {
  string id           = 1;
  string email        = 2;
  reserved 3;                  // full_name retired — field number locked forever
  reserved "full_name";        // name also reserved to prevent reuse
  string display_name = 7;
  string age          = 4;     // still a breaking change — type changed
  bool   is_active    = 5;
  Role   role         = 6;
}

enum Role {
  ROLE_UNSPECIFIED = 0;
  ROLE_VIEWER      = 1;
  ROLE_EDITOR      = 2;
  ROLE_ADMIN       = 3;
  ROLE_SUPERADMIN  = 4;
}

message GetUserRequest  { string id = 1; }
message GetUserResponse { User user = 1; }

message CreateUserRequest {
  string email     = 1;
  string full_name = 2;
  int32  age       = 3;
  Role   role      = 4;
}
message CreateUserResponse { User user = 1; }
EOF
```

```bash
buf breaking --against '.git#branch=main'
```

Now you should see fewer errors — the `full_name` removal no longer triggers because `reserved` properly tombstones the field.

> 📝 **`reserved` is your proto retirement policy.** Always `reserve` both the field number AND the name when removing a field. This prevents a future developer from accidentally recycling the same number and corrupting old serialized messages stored in databases or queues.

---

## Phase 9 — Wire Into GitHub Actions CI

```bash
cat > .github/workflows/buf-check.yml << 'EOF'
name: Buf Proto Checks

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    paths:
      - 'proto/**'        # Only run when proto files change
      - 'buf.yaml'

jobs:
  buf:
    name: Lint & Breaking Change Check
    runs-on: ubuntu-latest

    steps:
      # 1. Checkout with full history so buf can diff against main
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0   # IMPORTANT: shallow clone breaks git-based diffing

      # 2. Install Buf CLI (official action)
      - name: Setup Buf
        uses: bufbuild/buf-action@v1
        with:
          version: '1.34.0'     # Pin the version — never use 'latest' in CI

      # 3. Validate syntax
      - name: Build
        run: buf build

      # 4. Enforce style
      - name: Lint
        run: buf lint

      # 5. Check for breaking changes vs main branch
      #    On PRs: compares PR head vs main
      #    On push to main: compares this commit vs previous commit
      - name: Breaking change detection
        run: |
          buf breaking \
            --against '.git#branch=origin/main' \
            --error-format github-actions

      # Note: --error-format github-actions annotates the PR diff inline
      # instead of just printing to stdout — engineers see the error
      # right on the changed line in the GitHub PR review UI.
EOF
```

> 📝 **`fetch-depth: 0` is mandatory.** GitHub Actions checks out a shallow clone by default (`fetch-depth: 1`). Buf needs the full git history to find the baseline commit on `origin/main`. Without it, you get `fatal: no common commits`.

> 📝 **`--error-format github-actions`** outputs errors in the GitHub Actions annotation format: `::error file=proto/v1/user.proto,line=11::Field "full_name" deleted`. This pins the error to the exact line in the PR diff — much better DX for reviewers.

---

## 🧩 Bonus: Compare Against the Buf Schema Registry (BSR)

If your team publishes protos to the [Buf Schema Registry](https://buf.build):

```bash
# Login to BSR
buf registry login

# Push your module (needs buf.yaml with 'name: buf.build/yourorg/userapi')
buf push

# Compare local changes against the published BSR version
buf breaking --against 'buf.build/yourorg/userapi'

# Compare against a specific tagged version on BSR
buf breaking --against 'buf.build/yourorg/userapi:v1.2.0'
```

> 📝 **BSR is the "npm for protos."** It hosts versioned, searchable proto modules with generated SDKs. The breaking check against BSR is the most powerful workflow: even if the local git history is squashed or force-pushed, the BSR always has the canonical published version to diff against.

---

## 🧠 Quick Reference — Key Commands

```bash
# Build (syntax check)
buf build

# Lint (style check)
buf lint

# Breaking: vs git branch
buf breaking --against '.git#branch=main'

# Breaking: vs git tag
buf breaking --against '.git#tag=v1.0.0'

# Breaking: vs binary snapshot
buf breaking --against ./snapshots/v1.binpb

# Breaking: vs BSR module
buf breaking --against 'buf.build/acme/petapis'

# Breaking: only check specific files
buf breaking --against '.git#branch=main' --path proto/v1/user.proto

# Breaking: output as JSON (for tooling)
buf breaking --against '.git#branch=main' --error-format json

# Format all proto files in place
buf format -w

# Generate code (needs buf.gen.yaml)
buf generate
```

---

## 🎯 Lab Summary

You just built a complete Buf CLI breaking-change detection system from scratch. The key mental model:

1. **Your `.proto` files are a contract** — clients compile their code against them. A breaking change is any modification that makes compiled old clients fail against new servers (or vice versa).
2. **Buf snapshots the contract** — via git commits, binary images, or the BSR — and diffs it structurally, not textually. Renaming a variable in your code is fine; renaming a proto field while keeping the number is still a wire-compat break in JSON.
3. **Exit codes drive CI gates** — `buf breaking` exits `1` on violations, which fails the job and blocks the PR merge. No manual review needed for these mechanical checks.
4. **`reserved` is your tombstone** — never delete a field number, always reserve it. Buf enforces this, and it saves you from corrupted messages in long-lived systems like Kafka topics, databases with serialized protos, or caches.