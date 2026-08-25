# CODEBUDDY.md

This file provides guidance to CodeBuddy Code when working with code in this repository.

## Repository Overview

pgvector is a PostgreSQL extension implemented primarily in C. It provides vector data types and approximate nearest-neighbor access methods. The extension currently declares version `0.8.2` in `vector.control`.

The build uses PostgreSQL PGXS on Unix-like systems and a separate MSVC/NMAKE build on Windows.

## Development Commands

### Linux and macOS

Building requires PostgreSQL server development files, `pg_config`, a C compiler, and `make`. Running the regression and TAP suites also requires an installed extension and a PostgreSQL test instance.

```sh
make
make install
make installcheck
make prove_installcheck
make clean
```

`make installcheck` runs SQL regression tests. `make prove_installcheck` runs the Perl TAP tests. Run both for the complete test suite.

Run one SQL regression test by naming the test without `.sql`:

```sh
make installcheck REGRESS=vector_type
```

Run one TAP test by passing its path:

```sh
make prove_installcheck PROVE_TESTS=test/t/001_ivfflat_wal.pl
```

For a portability build without host-specific CPU optimization:

```sh
make clean && make OPTFLAGS=""
```

For an assertion-enabled build similar to CI:

```sh
make clean && PG_CFLAGS="-DUSE_ASSERT_CHECKING" make && make install
```

There is no dedicated lint or formatter target. The closest repository-defined compile/static checks are the CI warning flags:

```sh
PG_CFLAGS="-DUSE_ASSERT_CHECKING -Wall -Wextra -Werror -Wno-unused-parameter -Wno-sign-compare" make
```

On macOS, CI also runs Clang static analysis with `scan-build --status-bugs make`.

The Makefile also provides Docker image targets when Docker is available:

```sh
make docker PG_MAJOR=17
```

### Windows

Use an x64 Native Tools Command Prompt for Visual Studio with PostgreSQL and `nmake` available. Set `PGROOT` to the PostgreSQL installation directory.

```cmd
set "PGROOT=C:\Program Files\PostgreSQL\18"
nmake /F Makefile.win
nmake /F Makefile.win install
nmake /F Makefile.win installcheck
nmake /F Makefile.win clean
nmake /F Makefile.win uninstall
```

`Makefile.win` defines the Windows regression set explicitly. Its `installcheck` target runs `pg_regress` against that complete set; the Unix Makefile is the supported path for selecting an individual regression test with `REGRESS=...`. PostgreSQL versions before 17 may require `PG_REGRESS=$(PGROOT)\bin\pg_regress`, as used by CI for PostgreSQL 14.

## Test Layout

- `test/sql/` contains SQL regression inputs.
- `test/expected/` contains expected `pg_regress` output.
- `test/t/` contains Perl TAP/integration tests, including WAL, vacuum, recall, filtering, planner costs, iterative scans, distance functions, input validation, comparisons, duplicate handling, storage, and low-memory build coverage.
- `test/perl/` contains PostgreSQL Perl test utilities needed for older PostgreSQL versions.

When changing SQL behavior, update the relevant SQL test and expected output together. When changing WAL, vacuum, index, or lifecycle behavior, inspect the related TAP tests as well as regression tests.

## Architecture

### Extension and SQL Interface

- `vector.control` defines the extension metadata, default version, shared library path, and relocatability.
- `sql/vector.sql` is the canonical extension SQL script. It declares types, C-backed functions, casts, operators, aggregates, access methods, and B-tree/HNSW/IVFFlat operator classes and support functions.
- `sql/vector--old--new.sql` files are upgrade scripts between extension versions.
- `Makefile` generates the versioned `sql/vector--0.8.2.sql` file, lists all C objects in the shared `vector` module, defines CPU portability and auto-vectorization flags, configures TAP compatibility paths, and provides distribution and Docker targets.

The SQL declarations use PostgreSQL's `MODULE_PATHNAME`, so function signatures and SQL operator/index definitions are part of the contract between the SQL layer and C implementation.

### Vector Types and Operations

- `src/vector.c` and `src/vector.h` implement the dense `vector` type, varlena representation, parsing/serialization, casts, arithmetic, distance functions, and module initialization.
- `src/halfvec.c` and `src/halfutils.*` implement half-precision vectors and their numeric helpers.
- `src/bitvec.c`, `src/bitvec.h`, and `src/bitutils.*` implement binary vectors and bit operations.
- `src/sparsevec.c` and `src/sparsevec.h` implement sparse vectors.

Changes to a type's on-disk representation, dimensionality limits, input/output behavior, or distance semantics can affect SQL declarations, indexes, upgrade behavior, and multiple test groups.

### Approximate Nearest-Neighbor Access Methods

HNSW and IVFFlat are PostgreSQL index access methods split into lifecycle-specific modules.

HNSW:

- `src/hnsw.c`: access-method initialization and options.
- `src/hnswbuild.c`: index construction.
- `src/hnswinsert.c`: tuple insertion.
- `src/hnswscan.c`: index scans.
- `src/hnswutils.c`: shared structures and helpers.
- `src/hnswvacuum.c`: vacuum and maintenance.
- `src/hnsw.h`: on-disk structures, locks, limits, and parameters.

IVFFlat:

- `src/ivfflat.c`: access-method initialization and options.
- `src/ivfbuild.c`: build phases.
- `src/ivfinsert.c`: tuple insertion.
- `src/ivfscan.c`: index scans.
- `src/ivfutils.c`: shared logic.
- `src/ivfvacuum.c`: vacuum and maintenance.
- `src/ivfkmeans.c`: centroid training.
- `src/ivfflat.h`: access-method structures, contracts, and limits.

Index behavior is exposed through definitions in `sql/vector.sql` and exercised by the HNSW/IVFFlat regression and TAP tests. Changes to build, scan, insert, vacuum, or storage code should be reviewed with the corresponding access-method modules and tests together.

## Formatting and CI Scope

`.editorconfig` requires tabs for indentation in `*.c`, `*.h`, `*.pl`, `*.pm`, and `*.sql` files. CI tests PostgreSQL versions 13 through 19 on Linux, with ARM runners for PostgreSQL 14 and 16; PostgreSQL 14 and 18 on macOS; and PostgreSQL 14 and 17 on Windows. A Debian i386 job runs both test suites on PostgreSQL 15. A PostgreSQL 18 Valgrind job builds with `OPTFLAGS=""` and enables undefined-behavior checking.
