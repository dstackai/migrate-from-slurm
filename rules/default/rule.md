---
alwaysApply: true
---

## Project Overview
- All rules below are super important.
- This project maps Slurm concepts/approaches into dstack concepts/approaches. Target audience are Slurm users or admins that want to migrate to dstack.

## File structure
- `TOC.md` has concept ordered concept names and links to the detailed md files under the `concepts` folder. Each file under the `concepts` folder has title (matching TOC) and two sections: `Slurm` and `dstack`.
- When updating any of the files, make sure to update ordering numbers across TOC (and under `concepts`).

## Writing Guidelines
- Only highly technical and non-abstract information.
- Never use vague or ambiguous terminology. Always be super-clear and accurate. If not sure about how something works, explicitly state that you don't know for sure.
- As an intermediate step, write how dstack works in certain aspects into temporary md files (specify location if needed).

## Documentation sources
- Rely on Slurm official documentation: https://slurm.schedmd.com/
- Where it makes sense, rely on other Slurm-related highly accurate technical resources, the newer - the better.
- In some cases, also rely on other cloud vendor Slurm-related documentation (incl. but not limited to Slurm with Kubernetes, etc).
- Rely heavily on dstack documentation: https://dstack.ai/docs/
- Rely heavily on dstack source code: https://github.com/dstackai/dstack/. Especially the scripts prefixed with `processing_` that hold the scheduling and provisioning logic.
- Rely heavily on documents under https://github.com/dstackai/dstack/tree/master/contributing.