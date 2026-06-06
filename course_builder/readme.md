# Course Builder

This folder contains the prompt files and reference material used to build Moodle course backups from source content. It is designed for local use in your own workspace, where you can adapt the paths, output locations, and Moodle conventions to match your environment.

## What This Repo Contains

- `SKILL.md` - the workflow instructions for building Moodle `.mbz` backups, Moodle XML question banks, and companion H5P files.
- `PROJECT_PROMPT.md` - the project-level instructions that describe the expected course-building process.
- `gemma_4_12B/` - a sample course package and related prompt files for the Gemma 4 12B course.

## How To Use It In Your Own Environment

1. Open this repository in VS Code or your preferred editor.
2. Read `SKILL.md` and `PROJECT_PROMPT.md` to understand the required workflow.
3. Replace any example paths in the prompts with paths that exist on your machine.
4. Provide your own source document, URL, or `.mbz` template when starting a course build.
5. Approve the proposed course layout before the course package is generated.
6. Import the generated `.mbz` file into Moodle and upload any companion `.h5p` files through the Moodle Content Bank.

## Typical Workflow

1. Share source material.
2. Review the proposed course structure.
3. Generate the Moodle backup package.
4. Generate the Moodle XML question bank.
5. Upload any H5P activities separately if they are part of the course.

## Local Setup Notes

- Use paths that match your own computer or server.
- Do not rely on any internal development hostnames or hard-coded temporary directories.
- If you are working with a Moodle backup template, keep it as a reference only unless you intentionally want to reuse its structure.
- Make sure the Moodle version you target supports the activity and backup format described in the prompts.

## Example Request

You can start with a request like:

> Build a Moodle course from this source URL and this template `.mbz` file.

The workflow will then inspect the source, propose a structure, wait for approval, and generate the course backup and question bank.

## Repository Purpose

This repository is documentation and prompt scaffolding for a Moodle course-building workflow. It does not depend on any one machine, and it should be easy to adapt to your own environment by updating paths, output folders, and Moodle-specific settings.