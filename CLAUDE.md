# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal learning log for Docker and Kubernetes, written in Korean, aimed at interview prep. There is no source code, build system, or test suite — the entire repo is markdown notes plus real command output captured from hands-on practice (mostly on Windows with Docker Desktop/WSL2 and minikube).

## Structure

- `README.md` — repo overview, index of learning entries ("학습 기록"), and the forward roadmap ("학습 로드맵") planning upcoming Day topics
- `notes/day01-docker-basics.md`, `notes/day02-kubernetes-basics.md`, etc. — one file per learning session ("Day N")

When starting a new day's session, pick the topic from the README roadmap (and the previous day's `**다음 학습 목표**:` line). After finishing a day, update both the "학습 기록" list and the roadmap if the plan shifted.

## Conventions for notes files

Each day's note follows a consistent template — match it when adding or editing entries:

- Title: `# <주제> — 면접 대비 학습 정리`, followed by a blockquote stating the learning goal.
- Topics are `##` sections, each structured as: concept explanation → **왜 이렇게 설계됐는가** (why it's designed this way) → a `> **면접 답변**:` blockquote giving a spoken interview-style answer → a "실제 실행 결과"/"직접 확인한 실습" subsection with the actual command and its real output pasted in.
- A closing `## 오늘 배운 것 전체 흐름 요약` numbered list ties the day's sections together, ending with a `**다음 학습 목표**:` line pointing at the next topic.
- Command output blocks are real captured output (from PowerShell on Windows), not fabricated — when continuing a session, wait for the user to paste actual output before writing it into the notes rather than inventing plausible-looking results.
- After adding a new day's file, update the "학습 기록" list in `README.md` with a link and short description.

## Working style for this repo

- Always answer with current, verified information. Tool behavior (Docker Desktop, minikube, kubectl, Kubernetes APIs) changes across versions, and training data goes stale — before stating version-dependent facts in the notes, verify against current official docs/release notes (WebFetch/WebSearch) or the user's actual environment output, and cite the specific version and date in the note (e.g. "Docker Desktop 4.30 (2024-05)부터 단일 WSL 배포판으로 통합"). Never present remembered-but-unverified behavior as current fact; if something can't be verified, say so explicitly.
- The user runs `kubectl`/`minikube`/`docker` commands themselves in their own terminal rather than having Claude execute them via the Bash/PowerShell tool. When helping with hands-on exercises, provide the commands as code blocks in the response for the user to run, then incorporate the output they paste back into the notes file.
- When multiple near-duplicate note files show up (e.g. exported from another tool at different stages of a session), treat the most complete/final version as authoritative and consolidate into a single `dayNN-*.md` file rather than keeping multiple partial versions.
- Notes go through accuracy-review passes ("N일차 노트 내용 평가") after being written: check claims against official docs and the captured output, then fix errors in place. When asked to evaluate a day's notes, verify factual claims (with sources) rather than only reviewing style.
- Commits use Conventional Commits with `docs:` type (this repo is documentation-only), message subject in English, and are pushed to `origin/main` on GitHub (`reha-design/docker-kubernetes-learn`, private).
