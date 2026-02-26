# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Book Agent는 Claude Code의 Skill과 Agent를 활용하여 **책 집필 워크플로우를 자동화**하는 AI Agent 시스템 프로젝트입니다. 이 저장소는 가이드 문서와 MSA 아키텍처 설계 문서로 구성됩니다.

## Repository Structure

- `README.md` — Claude Code Skills & Agents 가이드 (Skill 생성, Agent 생성, Agent SDK 사용법, 프로젝트 구조 설계)
- `msa-book/` — MSA 아키텍처 관련 책 원고 (Ingress + auth-url, Gateway API, Spring Cloud Gateway 등 4가지 아키텍처 비교)

## Architecture: Skill ↔ Agent 관계

- **Skill** = 재사용 가능한 지시사항 세트 (`.claude/skills/<name>/SKILL.md`). "어떻게 하는지"를 정의
- **Agent** = Skill을 조합하여 작업을 수행하는 전문 AI 어시스턴트 (`.claude/agents/<name>/AGENT.md`). "무엇을 하는지"를 담당
- Agent는 `skills` 필드로 여러 Skill을 미리 로드하여 사용

## Language

이 프로젝트의 문서는 **한국어**로 작성됩니다. 응답 및 문서 작성 시 한국어를 사용하세요.

## Key Conventions

- Skill/Agent 이름: 소문자, 하이픈만 사용, 최대 64자
- Skill 최소 구성: `SKILL.md` 파일 하나
- Agent 최소 구성: `AGENT.md` 파일 하나
- 문서 형식: Markdown, YAML 프론트매터 사용
