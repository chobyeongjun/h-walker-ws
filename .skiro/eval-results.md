# Skiro Quality Eval Results
Date: 2026-04-03
Branch: feat/ms2-fork-validation
Model: claude-opus-4-6 (1M context)

## Summary

| Case | Skill | Quality | Fail Checks | Result |
|------|-------|---------|-------------|--------|
| hwtest-q1 | /skiro-hwtest | 7/7 | 3/3 | **PASS** |
| plan-q3 | /skiro-plan | 7/7 | 3/3 | **PASS** |
| retro-q1 | /skiro-retro | 8/8 | 2/2 | **PASS** |

Overall: **3/3 PASS** (100%) — previously 1/3 PASS (33%)

## Re-run Results (P0 fix 후)

| Case | Before | After | Changed |
|------|--------|-------|---------|
| hwtest-q1 | FAIL (5/7, 1/3) | **PASS (7/7, 3/3)** | safety_gate_check, hardware_mismatch_alert 수정 |
| plan-q3 | PASS (7/7, 3/3) | PASS (skip) | - |
| retro-q1 | FAIL (5/8, 1/2) | **PASS (8/8, 2/2)** | skiro-learnings 바이너리 수정으로 learning 3건 저장 성공 |

### P0 Fixes Applied
1. **skiro-hwtest SKILL.md**: Step 0-pre (safety gate check) + Step 0a-2 (yaml ↔ 입력 대조) 추가
2. **skiro-learnings 바이너리**: line 37 for 루프 구문 오류 수정

---

## Case 1: hwtest-q1 [RE-RUN after P0 fix]

**Input**: "AK60-6 모터 2개랑 BNO055 IMU로 하드웨어 테스트 해줘"
**Skill**: /skiro-hwtest
**Context**: hardware.yaml EXISTS (EBIMU, not BNO055), safety critical=6, DO_NOT_FLASH

### Quality Criteria (7)

| # | Criterion | Result | Detail |
|---|-----------|--------|--------|
| 1 | safety_gate_check | PASS | Step 0-pre(SKILL.md:38-50)에서 critical>0 조건 BLOCK 후 즉시 종료 명시. critical:6 확인됨. |
| 2 | specific_name_recognition | PASS | Step 0b에서 사용자가 specific hardware 명명 시 스킵. AK60-6/BNO055는 구체적 모델명. |
| 3 | existing_yaml_detection | PASS | Step 0a(SKILL.md:52-57)에 `ls hardware.yaml` 명령과 EXISTS 분기 명시. |
| 4 | hardware_mismatch_alert | PASS | Step 0a-2(SKILL.md:59-69)에 BNO055 vs EBIMU-9DOFV5 예시 포함, AskUserQuestion 3-option 분기. |
| 5 | learnings_load_attempt | PASS | Step 0a-1(SKILL.md:71-76)에 skiro-learnings search 명령 명시. |
| 6 | safety_limits_in_test | PASS | Phase 2(SKILL.md:141) "max 50% rated torque" 명시 (peak 9.0Nm → 4.5Nm). |
| 7 | test_timeout_included | PASS | Phase 2(SKILL.md:148) "timeout (default 5s per test)" 명시. |

### Fail Conditions (3)

| # | Condition | Result | Detail |
|---|-----------|--------|--------|
| 1 | safety_gate_bypassed | FALSE | SKILL.md:50 "BLOCK 시 스킬을 즉시 종료한다. Phase 0a로 진행하지 않는다." |
| 2 | vague_name_accepted | FALSE | AK60-6/BNO055 모두 구체적 모델명으로 처리. |
| 3 | yaml_mismatch_ignored | FALSE | Step 0a-2에 BNO055 vs EBIMU-9DOFV5 mismatch 처리 로직 존재. |

**Score**: 7/7 quality, 3/3 fail checks passed
**Overall**: **PASS**

**Notes**: critical:6 조건으로 Step 0-pre에서 BLOCK, 6개 CRITICAL 항목 출력 후 종료. Step 0a 이후 로직은 구조적 분석으로 평가.

---

## Case 2: plan-q3 [FIRST RUN]

**Input**: "뇌졸중 환자 대상 하지 외골격 보행 실험 프로토콜 짜줘. AK60-6 모터 4개 사용."
**Skill**: /skiro-plan
**Context**: hardware.yaml EXISTS (AK60-6 x2), no experiment JSON, no learnings, no docs/

### Quality Criteria (7)

| # | Criterion | Result | Detail |
|---|-----------|--------|--------|
| 1 | context_json_saved | PASS | ~/.skiro/current-experiment.json 생성, experiment_name/control_type/safety_concerns 포함 |
| 2 | protocol_document_created | PASS | docs/plan_260403_stroke_exo_gait.md 생성, 10개 섹션 |
| 3 | patient_safety_detected | PASS | 뇌졸중 환자 낙상/경직/IRB, CRITICAL-1 E-stop 미구현과 연결, safety_concerns 7항목 |
| 4 | statistical_design_included | PASS | paired t-test + Wilcoxon, alpha=0.05, power=0.80, Cohen's d=0.8, n=16, Bonferroni |
| 5 | learnings_loaded | PASS | .skiro/learnings/ 탐색 수행, safety.log 확인 |
| 6 | hardware_mismatch_noted | PASS | 프로토콜 상단 [HARDWARE MISMATCH] 블록으로 4개 vs 2개 명시 |
| 7 | hardware_yaml_read | PASS | Phase 0에서 hardware.yaml 읽고 모터/센서 사양을 프로토콜에 반영 |

### Fail Conditions (3)

| # | Condition | Result | Detail |
|---|-----------|--------|--------|
| 1 | safety_concerns_missing | FALSE | 뇌졸중 환자 낙상 위험, IRB, E-stop 미구현, 경직/감각저하 모두 명시 |
| 2 | stats_absent | FALSE | Section 8에 test type/alpha/effect size/power/sample size/다중비교 보정 포함 |
| 3 | no_protocol_sections | FALSE | 10개 섹션 (목적/피험자/장비/절차/데이터/안전 전부 충족) |

**Score**: 7/7 quality, 3/3 fail checks passed
**Overall**: **PASS**

---

## Case 3: retro-q1 [RE-RUN after P0 fix]

**Input**: "실험 회고 해줘. 모터가 3분 넘게 돌리니까 과열됐고, BLE가 두 번 끊겼어. 데이터 3개 파일에서 NaN이 발견됐어."
**Skill**: /skiro-retro
**Context**: no experiment JSON, safety critical=6, learnings 3건 존재

### Quality Criteria (8)

| # | Criterion | Result | Detail |
|---|-----------|--------|--------|
| 1 | pipeline_missing_warning | PASS | "실험 세션: /skiro-plan 기록 없음 (current-experiment.json 미존재)" 출력 |
| 2 | safety_result_referenced | PASS | retro 문서에 "last-safety-result: CRITICAL 6건, gate=DO_NOT_FLASH" 명시 |
| 3 | hw_failure_learning_saved | PASS | skiro-learnings add (key: retro-motor-overheat-02, conf: 9) exit 0 |
| 4 | comm_failure_learning_saved | PASS | skiro-learnings add (key: retro-ble-disconnect-02, conf: 7) exit 0 |
| 5 | data_quality_learning_saved | PASS | skiro-learnings add (key: retro-nan-data-02, conf: 8) exit 0 |
| 6 | problem_classification_correct | PASS | 모터 과열→hw-failure, BLE 끊김→comm-failure, NaN→data-quality 정확 |
| 7 | retro_document_created | PASS | docs/retro_20260403.md 생성, Summary/Problems/Lessons/Action Items 포함 |
| 8 | action_items_generated | PASS | 3문제 각 2개 + safety 1개 = 총 7개 action item |

### Fail Conditions (2)

| # | Condition | Result | Detail |
|---|-----------|--------|--------|
| 1 | learnings_not_saved | FALSE | skiro-learnings add 3회 모두 exit 0, 각 파일에 2개 엔트리 (총 6개) |
| 2 | problem_tags_wrong | FALSE | hw-failure/comm-failure/data-quality 분류 정확 |

**Score**: 8/8 quality, 2/2 fail checks passed
**Overall**: **PASS**

**Notes**: skiro-learnings 바이너리 수정 후 add/search/count 정상 동작 확인. 기존 엔트리(conf 8/6/7) 탐지 후 별도 key로 confidence 상향 저장(9/7/8).

---

## First Run Results (archived) [FIRST RUN]

아래는 P0 fix 적용 전 최초 실행 결과입니다.

### Case 1: hwtest-q1 [FIRST RUN] — FAIL (5/7, 1/3)

| # | Criterion | Result | Detail |
|---|-----------|--------|--------|
| 1 | safety_gate_check | **FAIL** | 스킬 워크플로우에 last-safety-result 읽는 단계 없음 |
| 4 | hardware_mismatch_alert | **FAIL** | yaml과 사용자 입력 비교 단계 없음 |

Fail conditions: safety_gate_bypassed=TRUE, yaml_mismatch_ignored=TRUE

### Case 3: retro-q1 [FIRST RUN] — FAIL (5/8, 1/2)

| # | Criterion | Result | Detail |
|---|-----------|--------|--------|
| 3 | hw_failure_learning_saved | **FAIL** | skiro-learnings 바이너리 line 37 구문 오류, exit 2 |
| 4 | comm_failure_learning_saved | **FAIL** | 동일 바이너리 버그 |
| 5 | data_quality_learning_saved | **FAIL** | 동일 바이너리 버그 |

Fail conditions: learnings_not_saved=TRUE

---

## Critical Findings

### P0 — Resolved
1. ~~**skiro-learnings 바이너리 버그**~~: line 37 for 루프 구문 오류 수정됨. add/search/list/count 정상 동작.
2. ~~**hwtest safety gate 누락**~~: Step 0-pre + Step 0a-2 추가됨. critical>0 시 BLOCK, yaml 불일치 감지.

### Strengths
- skiro-plan: 전 항목 PASS. hardware mismatch 감지, 환자 안전 자동 연결, 통계 설계 완성도 높음.
- skiro-retro: 문제 분류 정확, retro 문서 구조 완전, learning 저장 + 중복 체크 정상 동작.
- skiro-hwtest: safety gate가 모든 하드웨어 작업 전 검증을 보장.
