# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal learning log for Docker and Kubernetes, written in Korean, aimed at interview prep. There is no source code, build system, or test suite — the entire repo is markdown notes plus real command output captured from hands-on practice (mostly on Windows with Docker Desktop/WSL2 and minikube).

## Structure

- `README.md` — repo overview and index of learning entries
- `notes/day01-docker-basics.md`, `notes/day02-kubernetes-basics.md`, etc. — one file per learning session ("Day N")

## Conventions for notes files

Each day's note follows a consistent template — match it when adding or editing entries:

- Title: `# <주제> — 면접 대비 학습 정리`, followed by a blockquote stating the learning goal.
- Topics are `##` sections, each structured as: concept explanation → **왜 이렇게 설계됐는가** (why it's designed this way) → a `> **면접 답변**:` blockquote giving a spoken interview-style answer → a "실제 실행 결과"/"직접 확인한 실습" subsection with the actual command and its real output pasted in.
- A closing `## 오늘 배운 것 전체 흐름 요약` numbered list ties the day's sections together, ending with a `**다음 학습 목표**:` line pointing at the next topic.
- Command output blocks are real captured output (from PowerShell on Windows), not fabricated — when continuing a session, wait for the user to paste actual output before writing it into the notes rather than inventing plausible-looking results.
- After adding a new day's file, update the "학습 기록" list in `README.md` with a link and short description.

## Working style for this repo

- The user runs `kubectl`/`minikube`/`docker` commands themselves in their own terminal rather than having Claude execute them via the Bash/PowerShell tool. When helping with hands-on exercises, provide the commands as code blocks in the response for the user to run, then incorporate the output they paste back into the notes file.
- When multiple near-duplicate note files show up (e.g. exported from another tool at different stages of a session), treat the most complete/final version as authoritative and consolidate into a single `dayNN-*.md` file rather than keeping multiple partial versions.
