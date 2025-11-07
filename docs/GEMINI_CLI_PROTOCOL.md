# GEMINI_CLI_PROTOCOL.md

## 1. 목적

이 문서는 **Gemini CLI 전용 운영 규칙**을 정의한다. `GEMINI.md`가 코드와 프로젝트 전반의 "기본 헌장"이라면, 본 문서는 **CLI 환경에서 Gemini를 어떻게 운용할지**를 구체화한다.

---

## 2. 원칙

1. **`GEMINI.md` 우선**: 모든 작업은 `GEMINI.md`의 규칙(코드 삭제 금지, placeholder 금지 등)을 최상위로 따른다.
2. **자동 수정 기반**: Gemini는 diff 출력을 하지 않고, 소스 파일을 직접 수정한다.
3. **추적성 확보**: 직접 수정의 단점을 보완하기 위해 **Changelog 기록**과 **백업 저장**을 필수화한다.

---

## 3. 답변 포맷

Gemini는 파일 수정 후, 반드시 아래 3단계를 출력한다.

### 3.1 [CHANGELOG]

*   파일별 변경 사항을 간결하게 기록한다.
*   예:

    ```
    [CHANGELOG]
    - core/action_handler.py: clear_cells에서 UI 동기화 로직 보강
    - modules/clickcell_engine.py: patternType None 처리 수정
    ```

### 3.2 [TEST]

*   수정된 기능을 검증할 최소 실행 단위를 제시한다.
*   예:

    ```bash
    pytest -q tests/test_engine.py::test_cf_rules
    ```
*   GUI 관련 이슈일 경우, **QTableModel 계약 점검 루틴**을 반드시 포함한다.

### 3.3 [BACKUP]

*   Gemini는 `[CHANGELOG]` 블록의 내용을 기반으로, `references/history/` 경로에 마크다운 형식의 변경 이력 파일을 **직접 생성**한다.
*   이 작업은 `write_file` 도구를 통해 자동으로 수행된다.
*   파일명은 `YYYY-MM-DD_작업_요약.md` 형식을 따른다.
*   예시: `print(default_api.write_file(file_path='/home/ttsoft/projects/TTClickCell/references/history/2025-10-02_action_handler_fix.md', content='[CHANGELOG]\n- core/action_handler.py: ...'))`

---

## 4. UI/렌더링 이슈 디버그 루틴

테이블 뷰가 표시되지 않거나 값이 비어 보이는 경우, 항상 다음 절차를 수행한다:

1.  `rowCount()`, `columnCount()` 로그 확인
2.  `data(index, role)`에서 DisplayRole/AlignmentRole 반환 값 점검
3.  `headerData()` 및 `flags()` 확인
4.  원본 DataFrame → `raw_data_grid` 변환 과정 검증

---

## 5. 다국어(i18n) 키 검증

1.  새로운 로그 키나 UI 문자열이 추가되면, `resources/clickcell_lang_*.json` 모두에 존재하는지 확인한다.
2.  누락 시 자동으로 추가하거나, 최소한 `default="[MISSING:KEY]"` 처리한다.

---

## 6. 백업 및 기록 규칙

*   Gemini CLI는 파일을 직접 수정하므로, **diff 기록은 남지 않는다.**
*   따라서 반드시 아래를 병행한다:

    1.  **Changelog**: 답변 내 `[CHANGELOG]` 블록 출력
    2.  **백업 파일**: `[BACKUP]` 규칙에 따라 `references/history/`에 자동 생성
    3.  **Git 사용 시**: 자동 수정 후 `git add -p`로 변경 범위 확인

---

## 7. 예시 세션

사용자: "조건부 서식 저장 시 CF merge 오류 발생. 로그 첨부."

Gemini 답변:

```
[CHANGELOG]
- modules/clickcell_engine.py: save_workbook 내 CF 병합 로직 수정 (CellIsRule dxf 호환성)

[TEST]
pytest -q tests/test_engine.py::test_save_cf_rules

[BACKUP]
print(default_api.write_file(file_path='/home/ttsoft/projects/TTClickCell/references/history/2025-10-02_cf_merge_fix.md', content='''[CHANGELOG]
- modules/clickcell_engine.py: save_workbook 내 CF 병합 로직 수정 (CellIsRule dxf 호환성)
'''))
```

---

## 8. 결론

이 프로토콜은 Gemini CLI가 직접 파일을 수정하는 워크플로에서 **안정성, 추적성, 검증 가능성**을 확보하기 위한 최소한의 장치이다. `GEMINI.md`와 함께 항상 참조되어야 하며, 프로젝트 전 생애주기에 적용된다.
