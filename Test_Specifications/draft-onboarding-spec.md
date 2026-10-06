# Unikraft onboarding specification (draft)

## Goal

Learn the repository safely before proposing or making a source-code change.

## Scope

- Read the top-level `README.md` and `CONTRIBUTING.md` first.
- Identify the top-level build entry points and relevant test commands.
- Explain the roles of the `arch/`, `plat/`, `lib/`, `drivers/`, `include/`, and `support/` directories.
- Select one small, existing build or test workflow to run.

## Constraints

- Do not edit Unikraft source code, configuration, or build files.
- Cite the repository files used for conclusions.
- Stop and ask before downloading dependencies, changing configuration, or running a command that changes files.
- Keep the result suitable for a developer who is new to both Unikraft and the repository.

## Expected result

1. A short orientation of the repository.
2. The smallest verified build or test workflow, including prerequisites.
3. A suggested beginner-sized next task, with the relevant files to inspect.

## Prompt for Kiro

> Use this document as the scope for an onboarding investigation. Do not edit files yet. Read `README.md` and `CONTRIBUTING.md`, inspect only the relevant repository files, and return a proposed requirements list. Cite the files you consulted and identify any assumptions or missing prerequisites.
