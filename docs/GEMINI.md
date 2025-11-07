# TTClickCell 코드 구현 지침 (Markdown)

> **`GEMINI.md` 업데이트 지침:** 이 `GEMINI.md` 파일을 업데이트할 때는, 전체 내용을 다시 생성하여 토큰을 낭비하는 대신 변경된 부분만 수정하는 것을 원칙으로 합니다. 특히, '파일별 상세 분석'섹션은 코드 업데이트로 인해 클래스 및 함수가 변경되는 것이 아닌 이상, 각 항목의 삭제나 생략은 하지 않습니다.
> 
> **코드 수정 시 핵심 원칙:**
> 1.  **기존 코드 보존**: 함수나 클래스를 수정할 때, **기능상 반드시 필요한 기존 코드를 임의로 삭제하거나 `pass` 등으로 대체하지 않습니다.** 수정 범위는 요청된 내용과 직접적으로 관련된 부분으로 한정하며, 다른 부분의 로직은 온전히 보존하는 것을 최우선으로 합니다.
> 2.  **실행 가능한 코드 작성**: `...` 와 같은 실행 불가능한 플레이스홀더 구문은 절대로 소스 코드 파일에 직접 작성하지 않습니다. 아직 구현되지 않은 함수를 위한 플레이스홀더가 필요할 경우에만, `pass` 키워드를 사용하거나 해당 라인을 주석 처리하여 항상 유효한 코드만 생성합니다.

> **CLI 상호작용 프로토콜:** 이 프로젝트는 Gemini와의 상호작용을 위한 별도의 프로토콜을 정의합니다. 코드 수정 및 기타 작업을 요청하기 전에 반드시 `GEMINI_CLI_PROTOCOL.md` 문서를 숙지하고 따라야 합니다.

---

### 🚨 긴급: 검증 필요 작업 목록 (2025-10-02 업데이트)

아래 목록은 1.0.0-rc.2 출시를 앞두고 최종 검증이 필요한 기능들입니다.

- **검증 필요한 핵심 기능**:
  1.  **서식 보존 저장**: 데이터 편집 후 파일을 저장했을 때, 원본의 서식(기본 서식, 조건부 서식, 셀 병합 등)이 손실되지 않고 그대로 유지되는지 검증이 필요합니다.
  2.  **실행 취소/다시 실행**: 필터, 중복 제거, 함수 적용 등 모든 주요 기능에 대해 실행 취소(Undo)와 다시 실행(Redo)이 데이터와 UI에 정확히 반영되는지 검증해야 합니다.

---

## 1. 개요

TTClickCell은 PySide6 기반의 GUI 애플리케이션으로, Excel 파일 자동화 기능을 제공합니다. 최종적으로 이 애플리케이션은 **Microsoft Store를 통해 배포될 예정**이며, 패키징은 **Nuitka**를 사용하여 네이티브 코드로 컴파일한 후 **MSIX** 형식으로 변환하여 안정적인 설치 및 업데이트를 지원합니다. 이 문서는 프로젝트의 모든 클래스와 메서드를 상세히 분석하여, 코드 수정 및 유지보수 시 참조 자료로 활용하는 것을 목적으로 합니다.

---

## 2. 프로젝트 폴더 구조 가이드

프로젝트의 효율적인 관리를 위해 폴더 구조가 다음과 같이 정리되었습니다. 파일을 생성하거나 참조할 때, 각 폴더의 목적에 맞게 사용해야 합니다.

- **`references/`**: 프로젝트의 주요 참고 자료를 보관하는 최상위 폴더입니다.
  - **`backup/`**: `GEMINI.md` 등 주요 문서의 백업 파일을 저장합니다.
  - **`cloud_run_server/`**: AI 프록시 서버의 Google Cloud Run 배포 관련 소스코드(`new_main_vXX.py`), 설정 파일, 배포 가이드 등을 관리합니다. **서버 코드를 수정할 때는 기존 파일을 직접 수정하지 않고, `new_main_v40.py`와 같이 버전을 올려 새 파일을 생성하고 파일 상단에 주석으로 버전을 명시하는 것을 원칙으로 합니다.**
  - **`directions/`**: 프롬프트 엔지니어링 방향, 테스트 방향 등 프로젝트의 상위 레벨 개발 방향성을 제시하는 문서를 저장합니다.
  - **`history/`**: 개발 과정에서 발생한 주요 이벤트, 대화 로그 등 개발 히스토리를 기록한 파일을 보관합니다.
  - **`logs/`**: Cloud Run 등 외부 서비스에서 다운로드한 원본 로그나 에러 로그를 저장합니다.
  - **`reports/`**: 기능 테스트, 이슈 분석 등 프로젝트와 관련된 분석 보고서를 저장합니다.
  - **`screen_shot/`**: 이슈가 발생한 UI에 관한 스크린 샷을 보관합니다.
  - **`suggestions/`**: 다른 AI 모델의 제안 내용이나 코드 수정 가이드라인 등 외부 제안 자료를 보관합니다.

---

## 3. 개발 환경 구성 가이드 (WSL 기준)

Windows Subsystem for Linux (WSL) 환경에서 개발 및 테스트를 진행할 때, Windows 파일 시스템(`/mnt/c`, `/mnt/d` 등)에 직접 프로젝트를 위치시키면 Python 가상 환경 생성(`venv`) 시 권한 문제로 오류가 발생할 수 있습니다. 

가장 안정적이고 확실한 해결책은 **프로젝트 전체를 WSL 네이티브 파일 시스템 내부로 복사하여 작업**하는 것입니다.

### 권장 구성 절차

1.  **WSL 터미널을 엽니다.**

2.  **WSL 홈 디렉토리에 프로젝트 폴더를 생성합니다.**
    ```bash
    mkdir ~/projects
    ```

3.  **Windows에 있는 기존 프로젝트를 새로 만든 폴더로 복사합니다.**
    ```bash
    # 예: D 드라이브에 있던 프로젝트를 복사하는 경우
    cp -r /mnt/d/TTSoft/Gemini_test/TTClickCell ~/projects/
    ```

4.  **새로운 프로젝트 경로로 이동하여 모든 작업을 수행합니다.**
    ```bash
    cd ~/projects/TTClickCell
    ```

5.  **새로운 경로에서 개발 환경을 구성합니다.**
    ```bash
    # 1. 가상 환경 생성
    python3 -m venv .venv

    # 2. 가상 환경 활성화 (새 터미널을 열 때마다 필요)
    source .venv/bin/activate

    # 3. 의존성 라이브러리 설치
    pip install -r 'launching test/requirements-test.txt'
    pip install ruff

    # 4. 테스트 데이터셋 생성
    python 'launching test/datasets/make_datasets.py'
    ```

이제 모든 개발 및 테스트 작업은 `~/projects/TTClickCell` 경로에서 수행해야 합니다.

### Gemini AI 작업 시 경로 규칙 (매우 중요)

이 프로젝트는 WSL에서 코드를 개발하고 Windows에서 테스트하는 이중 환경을 사용합니다. 따라서 AI 작업 시 경로 혼동을 막기 위해 아래 규칙을 반드시 준수해야 합니다.

- **개발/수정 기준 경로 (WSL)**: `/home/ttsoft/projects/TTClickCell/`
- **테스트/로그 참조 경로 (Windows)**: `d:/TTSoft/Gemini_test/TTClickCell/`

**핵심 규칙**: 내가 Windows 환경에서 발생한 로그나 에러 메시지를 제공할 경우, 당신은 **Windows 경로를 자동으로 WSL 경로로 변환하여 이해하고 작업을 수행**해야 합니다.

**예시:**
만약 내가 아래와 같은 Windows 로그를 전달하면,
> `[ERROR] File not found: d:/TTSoft/Gemini_test/TTClickCell/src/main.py`

당신은 이것을 아래와 같이 해석하고, 해당 **WSL 경로의 파일을 수정**해야 합니다.
> `[ERROR] File not found: /home/ttsoft/projects/TTClickCell/src/main.py`

---

## 4. 파일별 상세 분석

### 📄 `TTAiClickCell.py`

메인 애플리케이션 진입점입니다. `ExcelAutomationApp` 메인 윈도우 클래스를 정의하고 애플리케이션을 실행합니다.

#### 🏠 **Class: `ExcelAutomationApp(QMainWindow)`**
- `__init__(self, config, language_pack, session_log_file)`: 메인 윈도우의 모든 구성요소(UI, 핸들러, 엔진)를 초기화하고, AI 프록시 서버의 콜드 스타트 방지를 위한 `warm_up_server`를 호출합니다.
- `zoom_in(self)` / `zoom_out(self)` / `zoom_reset(self)`: **[신규]** 테이블 뷰의 표시 배율을 조절(확대/축소/초기화)하고 상태 표시줄의 줌 레벨을 업데이트합니다.
- `closeEvent(self, event)`: 사용자가 창을 닫을 때 호출됩니다. **데이터 변경 여부를 확인하여 저장 프롬프트를 띄우고**, 모든 활성 백그라운드 스레드(AI, 캐싱 등)를 안전하게 종료시킨 후, 윈도우 상태(크기, 위치)를 저장합니다.
- `_restore_window_state(self)`: 이전 윈도우 상태를 복원합니다.
- `add_log(self, message)`: 로그 창에 메시지를 추가하고, `logging` 모듈을 통해 파일에도 기록합니다.
- `save_log_to_file(self)`: 단축키(Ctrl+Shift+L)로 호출되어 현재 세션의 전체 로그를 타임스탬프가 찍힌 별도 파일로 저장합니다.

---

### 📄 `TTAiClickCell_CLI.py`

GUI 없이 핵심 데이터 처리 기능을 사용할 수 있는 명령줄 인터페이스(CLI) 버전입니다.

- **`handle_filter(...)`**: 필터링 명령을 처리합니다.
- **`handle_deduplicate(...)`**: 중복 제거 명령을 처리합니다.
- **`handle_transpose(...)`**: 데이터 전치 명령을 처리합니다.
- **`handle_formula(...)`**: 함수 마법사(Concat, If, Aggregate) 관련 명령을 처리합니다.
- **`main()`**: `argparse`를 사용하여 CLI 인자(입력/출력 파일, 시트, 명령어, 하위 옵션 등)를 파싱합니다. 적절한 `handle_*` 함수를 호출하여 작업을 수행하고, `ExcelEngine`을 통해 결과를 파일로 저장합니다.

---

### 📁 `core/`

#### 📄 `action_handler.py`
- **Class `ActionHandler`**
  - `apply_all_filters(self, conditions)`: 필터 조건을 `ExcelEngine`에 전달하여 데이터 필터링을 수행하는 워커를 시작합니다.
  - `remove_duplicates(self, selected_cols, keep_latest)`: 중복 제거를 `ExcelEngine`에 요청하는 워커를 시작합니다.
  - `apply_conditional_formatting(self, rules, mode)`: 조건부 서식 규칙을 현재 시트에 적용하고, `state_manager`를 통해 히스토리를 기록합니다.
  - `apply_ai_wizard_tasks(self, tasks)`: AI 마법사가 생성한 작업 목록을 `on_apply_formula`를 통해 순차적으로 실행합니다.
  - `on_apply_formula(self, task_name, params, from_ai)`: 함수 마법사의 작업을 `FormulaEngine`을 통해 실행하고, 결과를 DataFrame에 새 열로 추가하거나 요약 행을 추가합니다. 작업 후, **UI 일관성을 위해 `raw_data_grid`를 재생성**합니다.
  - `transpose_sheet(self)`: `ExcelEngine`을 통해 현재 시트 데이터를 전치(Transpose)하고 UI를 업데이트합니다.
  - `adjust_header(self, amount)`: 지정된 헤더 행 인덱스를 변경하고 `ExcelEngine`을 통해 DataFrame을 다시 읽어옵니다.
  - `on_process_finished(self, result)`: 데이터 처리 워커(필터, 중복 제거) 완료 시의 메인 콜백. **필터링 시 셀 스타일이 보존되도록, 필터링된 인덱스를 기반으로 원본 그리드에서 행을 재구성**하여 UI를 업데이트합니다.
  - `clear_cells(self, model, indexes)`: 선택된 셀들의 내용을 지웁니다 (모델과 DataFrame 동시 업데이트).
  - `paste_cells(self, model, start_index, text_data)`: 클립보드의 내용을 특정 셀부터 붙여넣습니다 (모델과 DataFrame 동시 업데이트).
  - `clear_cf_rules_for_selection(self, indexes)`: **선택된 열에 적용된 모든 조건부 서식 규칙(UI 및 원본 파일 규칙 포함)을 제거**합니다.

#### 📄 `ai_handler.py`
- **Class `AIHandler`**
  - `cancel_ai_task(self)`: 진행 중인 모든 AI 작업을 취소합니다.
  - `warm_up_server(self)`: **서버의 콜드 스타트를 방지**하기 위해 프록시 서버에 GET 요청을 보내는 워커를 시작합니다.
  - `run_ai_*` 메서드들: 각 AI 기능(필터, 조건부 서식, 함수 마법사, 요약)에 대한 워커를 시작합니다.
  - `on_ai_filter_finished(self, result, query)`: AI 필터 작업 완료 후 콜백. 생성된 규칙을 파싱하여 필터 UI에 추가합니다.
  - `on_ai_cf_finished(self, result, query)`: AI 조건부 서식 작업 완료 후 콜백. **설정된 적용 모드(`replace`/`append`)에 따라** UI 규칙을 교체하거나 추가합니다. **"빨간색"과 같은 색상 이름을 Hex 코드로 변환**하여 규칙을 생성합니다.
  - `on_ai_wizard_finished(self, result, query)`: AI 함수 마법사 작업 완료 후 콜백. **AI 응답이 리스트 형태일 때도 정상 처리**하도록 수정되었으며, 생성된 작업 목록을 UI에 표시합니다.
  - `_call_gemini_api(self, prompt)`: AI 프록시 서버에 **`Authorization` 헤더에 인증 토큰을 담아** POST 요청을 보내는 핵심 함수입니다.
  - `_build_ai_input_contract(self, task_type, ...)`: AI에 보낼 JSON 데이터 구조를 생성합니다. **'요약' 기능의 경우 통합 문서 전체 시트의 스키마를 포함**하도록 개선되었습니다.
  - `check_license(self)` / `check_usage(self)` / `increment_usage(self)`: **라이선스 및 사용량 체크 로직이 변경**되었습니다. 클라이언트는 UI 잠금 여부만 확인하고, 실제 사용량 체크 및 차감은 AI 요청 시 서버에서 직접 처리합니다. `increment_usage`는 작업 성공 후 UI를 새로고침하는 역할만 합니다.
  - `_get_usage_from_server(self)`: **자체 백엔드 서버(`/usage/{user_id}`)에서 현재 AI 사용량 정보를 가져와** UI에 표시합니다.

#### 📄 `file_handler.py`
- **Class `FileHandler`**
  - `on_file_loaded(self, result, file_path)`: 파일 로딩 완료 후 메인 콜백. **`cf_meta` 모듈을 사용하여 Excel 파일에 저장된 메타데이터를 읽고, 원본 조건부 서식(`baseline_cf_rules`)과 UI에서 관리하는 조건부 서식(`cf_rules`)을 분리**하여 세션 간 규칙 보존을 구현합니다.
  - `_prompt_to_save_if_dirty(self)`: 파일이 수정(`is_dirty=True`)되었을 경우, 사용자에게 변경사항 저장 여부를 묻는 대화상자를 표시합니다.
  - `load_file(self, path)`: 파일 로드 전, `_prompt_to_save_if_dirty`를 호출하여 미저장 변경사항을 확인합니다.
  - `save_file_dialog(self)`: 파일 저장 대화상자를 열고, **현재 UI에 설정된 조건부 서식 규칙(`ui_rules_by_sheet`)을 `ExcelEngine.save_workbook`에 전달**하여 파일에 함께 저장하도록 합니다.
  - `update_table_view(self, sheet_info)`: `ExcelTableModel`을 새로 생성하고 UI에 적용합니다. **모델이 생성된 후, `baseline_cf_rules`와 `cf_rules`를 모두 모델에 적용하여** 화면에 올바른 서식이 표시되도록 합니다.
  - `update_ui_from_data(self)`: 파일 로드, 시트 변경 등 데이터에 변화가 생겼을 때 전체 UI(시트 탭, 테이블 뷰, 컨트롤 패널)를 새로고침하는 메인 함수입니다.
  - `_start_column_data_caching(self)`: AI 프롬프트에 컨텍스트로 제공될 열의 고유값 스캔을 위한 백그라운드 워커를 시작합니다.

#### 📄 `state_manager.py`
- **Class `StateManager`**
  - `undo(self)` / `redo(self)`: 현재 시트의 상태를 이전/다음 단계로 되돌리고 UI를 업데이트합니다.
  - `push_state_after_action(self)`: 데이터 변경 작업 후, 현재 시트 상태의 깊은 복사본을 히스토리 스택에 추가합니다. 이 때, **메모리 과다 사용을 방지하기 위해 히스토리 스택의 최대 크기(`MAX_HISTORY_SIZE`)를 유지**하고, `is_dirty` 플래그를 `True`로 설정합니다.
  - `clear_history(self)`: 새 파일을 열 때 모든 시트의 실행 취소/다시 실행 히스토리를 초기화합니다.
  - `update_action_states(self)`: 히스토리 스택 상태에 따라 실행 취소/다시 실행 UI(버튼, 메뉴)의 활성화 상태를 업데이트합니다.

---

### 📁 `modules/`

#### 📄 `clickcell_engine.py`
- **Class `ExcelEngine`**
  - `load_workbook(self, ...)`: **[성능 개선]** `.xlsx` 파일 로딩 시, 기존의 `openpyxl` 대신 새로 구현된 `modules/fast_excel_reader.py` 모듈을 사용하여 더 빠른 속도로 파일을 읽습니다. `.xls` 파일은 `xlrd`를 사용합니다. 이 과정에서 헤더를 자동 감지하고, 서식, 병합 정보, 조건부 서식 규칙 등을 상세하게 추출합니다.
  - `save_workbook(self, path, data_dict, ...)`: **[핵심] '보존형 저장'**을 수행하는 메인 함수. 원본 파일을 새 경로에 복사한 뒤, `openpyxl`로 열어 변경된 데이터만 덮어씁니다. **특히, 조건부 서식의 경우 `cf_mode` 파라미터('preserve', 'replace', 'append')를 통해 저장 방식을 제어**하며, 원본 규칙(`baseline`)과 UI 규칙(`ui_rules`)을 병합하고 **`cf_meta` 모듈을 이용해 UI 규칙의 서명을 파일 내 숨은 공간에 저장**하여 다음 로드 시 구분할 수 있도록 합니다.
  - `apply_filters(self, df, conditions, ...)`: **데이터 타입(날짜, 숫자, 문자열)을 인지하여 각 타입에 맞는 방식으로 필터 조건을 적용**하도록 개선되었습니다.
  - `reset_header_and_get_df(self, original_grid, new_header_idx, lang)`: 사용자가 헤더 위치를 변경했을 때, 해당 인덱스를 기준으로 DataFrame을 재생성합니다.
  - `dataframe_to_raw_grid(self, df)`: DataFrame을 내부 `raw_data_grid` 형식으로 변환합니다.

#### 📄 `formula_engine.py`
- **Class `FormulaEngine`**
  - `concat_columns(self, df, ...)`: 여러 열의 텍스트를 주어진 구분자로 연결하여 새 Series를 반환합니다.
  - `conditional_value(self, df, ...)`: IF 조건을 평가하여, 조건이 참일 때와 거짓일 때 각각 다른 값을 가지는 새 Series를 생성합니다.
  - `lookup_value(self, df_left, ...)`: 두 DataFrame 간에 VLOOKUP과 유사한 조회를 수행합니다. **조인 키(key)의 이름이 같을 때 발생하는 `KeyError`를 해결**하기 위해, pandas가 자동으로 생성하는 `_y` 접미사를 감지하여 처리하도록 수정되었습니다.
  - `calculate_summary(self, df, ...)`: 지정된 열의 합계(SUM) 또는 평균(AVERAGE)을 계산하여 단일 숫자 값을 반환합니다.

#### 📄 `clickcell_model.py`
- **Class `ExcelTableModel(QAbstractTableModel)`**
  - `__init__(self, ...)`: 테이블 모델을 초기화하고, 메인 윈도우 참조를 저장합니다. 조건부 서식 규칙을 `priority`에 따라 정렬합니다.
  - `data(self, index, role)`: 특정 셀에 대한 데이터를 역할에 따라 반환합니다. **`_eval_cf_for_cell`을 호출하여 얻은 조건부 서식(`cf_fmt`)과 셀의 기본 서식(`base_fmt`)을 병합**하여 최종 서식을 결정합니다. **[성능 개선]** 렌더링 성능 향상을 위해 `QColor`, `QFont` 등 자주 사용되는 객체를 캐싱하여 재사용합니다.
  - `setData(self, index, value, role)`: 사용자가 셀 데이터를 편집할 때 호출됩니다. 모델과 연결된 DataFrame의 값을 변경하고, `state_manager`를 통해 변경 내역을 히스토리에 저장합니다.
  - `update_rules(self, simple_rules, append)`: UI나 AI로부터 받은 단순 형식의 규칙을 `openpyxl`이 이해할 수 있는 내부 형식으로 변환합니다. `append` 값에 따라 기존 규칙에 추가하거나 전체를 교체한 후, `layoutChanged` 신호를 보내 화면 전체를 새로 그리도록 합니다.
  - `_build_cf_index(self)`: 조건부 서식 규칙에 포함된 범위 문자열(예: 'A1:D10')을 파싱하여 내부적으로 사용할 수 있는 좌표 튜플 리스트로 변환합니다.
  - `_eval_cf_for_cell(self, ...)`: **특정 셀에 적용될 최종 조건부 서식을 평가하는 핵심 함수**입니다. 모든 규칙을 순회하며, 셀이 규칙 범위에 해당하면 `eval_cellIs`, `eval_expression` 등을 호출하여 조건을 평가하고, 그 결과를 캐시에 저장하여 성능을 최적화합니다.
  - `clear_formatting(self, indexes)`: **[비활성화]** 현재 버전에서는 기능이 주석 처리되어 있습니다.

---

### 📁 `ui/`

- **`ai_tab.py`**: 'AI 분석 (Beta)' 탭 UI. 라이선스 상태에 따라 잠금/해제 위젯을 전환하고, AI 사용량 표시 및 'AI 데이터 요약' 기능 UI를 생성합니다.
- **`cf_tab.py`**: 조건부 서식 규칙을 수동 또는 AI로 생성/관리하는 UI. 규칙 개수를 표시하는 배지(`update_rule_count_badge`) 기능이 포함되어 있습니다.
- **`dedup_tab.py`**: '중복 제거' 기능 UI. 중복 제거 기준 열과 옵션을 선택합니다.
- **`favorites_tab.py`**: 즐겨찾기 폴더 목록 관리 UI.
- **`filter_tab.py`**: 필터 조건을 수동 또는 AI로 생성/관리하는 UI. '빠른 필터'와 '지정 필터' 두 가지 AI 모드를 제공합니다.
- **`log_window.py`**: 로그 표시를 위한 독립적인 플로팅 창.
- **`setting_dialog.py`**: 설정 대화상자 UI. 언어, 시작 화면 표시 여부, 로그 창 자동 실행 등 주요 설정을 변경합니다.
- **`subscription_dialog.py`**: AI 기능 구독 요금제 선택 대화상자 UI.
- **`summary_dialog.py`**: AI 요약 결과를 표시하는 전용 대화상자 UI.
- **`welcome_widget.py`**: 프로그램 시작 시 나타나는 웰컴 스크린 UI. '빠른 시작 가이드'를 포함합니다.
- **`wizard_tab.py`**: '함수 마법사 (Beta)' 기능 UI. `QStackedWidget`을 사용하여 여러 함수 UI를 제공하며, **사용자가 파라미터를 입력할 때마다 결과 예시를 실시간으로 보여주는 기능**이 포함되어 있습니다.

---

### ☁️ `references/cloud_run_server/new_main_v39.py` (Cloud Run Proxy)

- **[FIX]** `v39` 버전에서는 AI 'filter' 작업 시, 조건 텍스트(예: `'>= 10'`)에서 실제 값( `10`)을 추출할 때 따옴표 등을 제거하는 `_clean_value` 로직이 누락되었던 버그를 수정했습니다.
- **`check_and_increment_usage(user_id)`**: Firestore DB를 사용하여 사용자의 API 사용량을 확인하고, **사용량이 한도 내에 있을 때만 카운트를 1 증가**시킵니다. 한도를 초과하면 에러를 반환합니다.
- **`ttclickcell_proxy(request)`**: HTTP 요청을 처리하는 메인 함수. **`Authorization` 헤더에서 인증 토큰을 확인**하고, `check_and_increment_usage`를 호출하여 사용량을 검증한 뒤, `task.type`에 따라 최적화된 프롬프트를 생성하여 AI 모델을 호출합니다.
- **`/usage/{user_id}` 엔드포인트**: **[신규]** 클라이언트가 현재 사용량과 한도를 조회할 수 있는 GET 엔드포인트를 제공합니다.
- **`/warmup` 엔드포인트**: **[신규]** 클라이언트가 서버의 콜드 스타트를 방지하기 위해 호출하는 간단한 GET 엔드포인트입니다.

---

## 5. 기타
  - Ai 모델은 현재 (25.09.02)기준 최신 모델인 gemini-2.5-flash 모델을 사용한다.

---

## 6. 향후 개선 후보 기능

MVP 출시 이후, 사용자 피드백과 시장 요구에 따라 다음 기능들을 순차적으로 도입하는 것을 검토합니다.

### 1. 데이터 클린징 툴킷
- **목표**: 반복적인 데이터 정제 작업을 자동화하여 작업 효율을 극대화합니다.
- **주요 기능**:
  - **공백 일괄 제거**: 셀 값의 앞/뒤/중간에 있는 불필요한 공백을 제거합니다.
  - **텍스트 서식 변환**: 대소문자 변환, 첫 글자만 대문자로 변경 등 일관된 서식을 적용합니다.
  - **텍스트 나누기/병합**: 특정 구분자를 기준으로 하나의 셀을 여러 셀로 나누거나, 여러 셀을 하나로 합칩니다.
  - **데이터 형식 변환**: 숫자 형식의 텍스트를 실제 숫자로, 또는 그 반대로 변환합니다.
  - **특수 문자 처리**: 전화번호, 주민번호 등에서 숫자만 남기거나 특정 패턴의 문자를 제거합니다.

### 2. 서식 및 편집 기능 강화
- **목표**: 기본적인 서식 편집 기능을 제공하여 Excel로 돌아가지 않고 간단한 수정을 완료할 수 있도록 합니다.
- **주요 기능**:
  - **서식 일괄 초기화**: 선택 영역의 모든 서식(글꼴, 색상 등)을 한 번에 제거합니다.
  - **강조 프리셋**: 클릭 한 번으로 행 전체에 노란색 배경을 적용하는 등 자주 쓰는 서식을 빠르게 적용합니다.
  - **직접 서식 편집**: 배경색, 글자색, 굵게, 이탤릭, 밑줄 등 기본적인 서식을 직접 편집하는 기능을 추가합니다. (단, 조건부 서식과의 충돌 해결 정책 필요)

### 3. 데이터 요약 및 분석
- **목표**: 간단한 클릭만으로 데이터에서 인사이트를 얻을 수 있는 요약/분석 기능을 제공합니다.
- **주요 기능**:
  - **요약 마법사**: 특정 열을 기준으로 합계, 평균, 개수 등을 자동으로 계산하여 새 테이블로 생성합니다.
  - **피벗 테이블 지원 강화**: 기존 피벗 테이블의 구조를 최대한 보존하여 불러오고, 저장 시에도 유지되도록 지원합니다.

### 4. 고급 AI 기능
- **목표**: 단순 작업 자동화를 넘어, AI가 데이터 처리의 전 과정을 주도하는 고차원 기능을 제공합니다.
- **주요 기능**:
  - **AI 데이터 요약**: 데이터의 핵심 내용을 파악하여 보고서용 텍스트 초안을 자동으로 생성합니다.
  - **AI 풀 오토마타**: "A시트와 B시트를 합친 후, 부서별로 매출 합계를 내고, 결과만 새 시트로 만들어줘"와 같은 복합적인 명령을 한 번에 처리합니다.
  - **AI 기반 작업 제안**: 사용자가 선택한 데이터의 맥락을 AI가 파악하여, "이 데이터는 중복 제거가 필요해 보입니다" 와 같이 다음에 할 작업을 추천합니다.

### 5. 편의성 및 기술적 개선
- **목표**: 사용자의 작업 흐름을 개선하고, 고급 사용자를 위한 옵션을 제공합니다.
- **주요 기능**:
  - **우클릭 메뉴 확장**: 행/열 삽입 및 삭제, 빠른 필터 및 정렬 등 자주 쓰는 기능을 우클릭 메뉴에 추가합니다.
  - **데이터 마스킹**: 주민번호, 전화번호 등 민감 정보를 익명화(*) 처리합니다.
  - **Excel 애드인 버전**: 독립 실행형 앱과 더불어, Excel 내에서 직접 실행할 수 있는 애드인 버전을 병행 개발합니다.
  - **BYOK (Bring Your Own Key)**: 사용자가 자신의 AI API 키를 등록하여 구독료 부담 없이 AI 기능을 사용하고, 사용량을 직접 관리할 수 있는 옵션을 제공합니다.

### 6. 셀 테두리 표시 기능 (고급 렌더링)
- **목표**: Excel 파일에 저장된 셀 테두리(선 스타일, 색상)를 TTClickCell의 테이블 뷰에 그대로 표시하여 원본과의 시각적 일관성을 극대화합니다.
- **핵심 과제**: Qt의 기본 `QTableView`는 셀별 개별 테두리 렌더링을 지원하지 않으므로, `QStyledItemDelegate`를 상속받아 `paint` 이벤트를 직접 처리하는 고수준의 커스텀 렌더링 구현이 필요합니다.
- **최적 구현 전략 (GPT 제안 종합)**:
  - **'엣지(Edge) 기반' 희소 데이터 모델**: '셀' 단위로 모든 테두리 정보를 저장하는 대신, '선(Edge)'을 기준으로 데이터를 저장하여 메모리 사용량을 최적화하고, 인접 셀이 동일한 경계선을 중복으로 그리는 문제를 원천적으로 방지합니다.
    - 예: `h_edges[(r, c)]` (셀 (r,c)의 하단 수평선), `v_edges[(r, c)]` (셀 (r,c)의 우측 수직선)
  - **단계적 도입**: 1단계 '경량 모드'에서는 실선/굵기/색상 위주로 구현하고, 2단계 '정밀 모드'에서 점선, 이중선 등 복잡한 스타일을 지원합니다.
- **상세 구현 단계**:
  1.  **데이터 모델 수정 (`modules/clickcell_engine.py`)**: `_read_xlsx` 함수에서 `cell.border` 정보를 읽어, 위에서 설명한 '엣지 기반' 데이터 구조로 변환하여 저장하는 로직을 추가합니다. 셀 병합 시 내부 엣지는 제외하고 외곽선 엣지만 저장합니다.
  2.  **커스텀 렌더링 델리게이트 구현 (`ui/border_delegate.py`)**: `QStyledItemDelegate`를 상속받는 `BorderDelegate` 클래스를 생성하고 `paint` 메서드를 재정의합니다.
      - 각 셀은 자신의 위쪽과 왼쪽 테두리만 그리도록 규칙을 정해 중복 그리기를 방지합니다.
      - `QPen` 객체를 캐싱하여 `paint` 이벤트 호출 시의 성능 저하를 최소화합니다.
      - Excel의 테두리 스타일('dotted', 'double' 등)을 Qt의 `PenStyle`로 변환하는 매핑 로직을 구현합니다.
  3.  **UI 적용 및 연동 (`core/file_handler.py`, `modules/clickcell_model.py`)**:
      - `file_handler`에서 테이블 뷰에 새로 만든 `BorderDelegate`를 적용합니다.
      - `clickcell_model`의 `data` 함수에 `BorderRole`과 같은 커스텀 역할을 추가하여, 델리게이트가 각 셀의 엣지 정보를 조회할 수 있도록 합니다.
---

## 7. 향후 작업 목록 (To-Do List)

- **MS 스토어 API 연동**:
  - `ai_handler._get_license_status_from_store` 함수 내에 MS 스토어 API 연동 코드를 구현하여, 실제 사용자의 라이선스(체험판, 영구 라이선스, 구독 등) 상태를 확인해야 합니다.
  - `ai_handler.purchase_ai_feature` 함수를 실제 MS 스토어 구매 API 호출 로직으로 교체해야 합니다.
- **자체 백엔드 서버 구축 및 연동**:
  - AI 사용량의 정확한 추적과 관리를 위해 사용자 인증을 처리하는 백엔드 서버 구축이 필요합니다.
  - `ai_handler._get_usage_from_server` 및 `increment_usage` 함수 내에 이 서버와 통신하는 API 코드를 구현해야 합니다.

---

## 8. 라이선스 분석 및 MS 스토어 출시 가이드

### 7.1. 라이브러리 의존성 및 라이선스 정책

TTClickCell의 상용 배포를 위해, 소스 코드에서 사용된 주요 외부 라이브러리의 라이선스를 분석한 결과는 다음과 같습니다. 모든 라이선스는 상업적 이용에 큰 제약이 없는 `Permissive` 또는 `Weak Copyleft` 유형입니다.

| 라이브러리 | 라이선스 | 주요 의무 및 참고사항 |
|---|---|---|
| **PySide6** | **LGPL v3** | **(가장 중요)** 동적 링크 필수, 라이선스 고지 및 소스 코드 제공 의무(Qt/PySide6 부분), 역공학 허용. | 
| **pandas** | BSD-3-Clause | Permissive. 라이선스 사본 포함. |
| **numpy** | BSD-3-Clause | Permissive. 라이선스 사본 포함. |
| **openpyxl** | MIT | Permissive. 라이선스 사본 포함. |
| **pyarrow** | Apache 2.0 | Permissive. 라이선스 및 NOTICE 파일 포함. |
| **requests** | Apache 2.0 | Permissive. 라이선스 및 NOTICE 파일 포함. |
| **xlrd** | BSD-3-Clause | Permissive. 라이선스 사본 포함. |
| **pywin32** | PSF License | Permissive. 라이선스 사본 포함. |

### 7.2. 핵심: PySide6 (LGPLv3) 라이선스 준수 사항

PySide6는 LGPLv3 라이선스를 따르므로, TTClickCell이 상업용闭源软件(closed-source software)으로 배포되기 위해서는 다음 조건을 반드시 준수해야 합니다.

1.  **동적 링크 (Dynamic Linking)**
    -   사용자가 PySide6 라이브러리 부분을 직접 수정하고 교체할 수 있도록 보장해야 합니다.
    -   Nuitka 컴파일 시, PySide6 관련 파일들(`.pyd`, `.dll`)이 최종 실행 파일(`.exe`)에 포함되지 않고 별도의 파일로 배포 폴더에 존재해야 합니다. (`--standalone` 모드의 기본 동작)

2.  **라이선스 고지**
    -   애플리케이션 내 '정보(About)' 또는 '도움말' 메뉴에 PySide6를 사용했음과 LGPLv3 라이선스 하에 있음을 명시해야 합니다.
    -   배포되는 패키지 안에 `LGPL-3.0.txt` 전문을 포함해야 합니다.

3.  **소스 코드 제공 (Qt/PySide6 한정)**
    -   사용자가 요청할 경우, TTClickCell이 사용한 **PySide6 라이브러리의 소스 코드**를 제공하거나, 소스 코드를 얻을 수 있는 위치(예: 공식 웹사이트)를 안내해야 합니다. (TTClickCell 전체 코드가 아닌, 라이브러리 소스 코드에 한정됩니다.)

4.  **역공학 허용**
    -   최종 사용자 라이선스 계약(EULA)에 "디버깅 목적의 역공학을 금지한다"와 같은 조항을 포함해서는 안 됩니다.

### 7.3. Nuitka 컴파일 및 패키징 가이드

Nuitka를 사용하여 라이선스를 준수하는 배포 패키지를 생성하는 권장 절차입니다.

1.  **`onedir` 모드로 컴파일 (권장)**:
    -   `--standalone` 옵션을 사용하여 모든 의존성을 하나의 폴더에 모읍니다. 이 방식은 LGPLv3의 동적 링크 요구사항을 자연스럽게 만족시킵니다.
    -   **PowerShell 예시 명령어**:
        ```powershell
        python -m nuitka ./TTAiClickCell.py --standalone --enable-plugin=pyside6 --include-qt-plugins=platforms,styles --output-dir=dist_win --windows-console-mode=disable
        ```

2.  **라이선스 파일 포함**: 
    -   컴파일 결과물 폴더(`dist_win/TTAiClickCell.dist`) 내에 `LICENSES` 라는 새 폴더를 생성합니다.
    -   `venv/Lib/site-packages` 경로에 있는 각 라이브러리 폴더에서 `LICENSE`, `LICENCE`, `NOTICE` 등의 파일을 찾아 `LICENSES` 폴더에 복사합니다.
    -   PySide6의 `LGPL-3.0.txt` 파일도 반드시 포함합니다.

3.  **고지문 작성 및 포함**:
    -   배포 폴더 최상단에 `THIRD_PARTY_NOTICES.txt` 파일을 생성하여, 사용된 모든 오픈소스 라이브러리 목록과 라이선스 유형을 명시합니다.

### 7.4. MS 스토어 등록 및 제출 절차

Nuitka로 빌드한 앱을 MS 스토어에 제출하는 절차는 다음과 같습니다.

1.  **사전 준비**:
    -   **MS 파트너 센터 개발자 계정**: 유료 등록이 필요합니다.
    -   **Visual Studio**: 최신 버전의 Visual Studio에 "MSIX Packaging Tools"와 "C++ build tools"를 설치합니다.

2.  **MSIX 패키징**:
    -   Nuitka로 컴파일한 `dist_win` 폴더 전체를 MSIX로 패키징해야 합니다. 이 방식은 사용자에게 안정적인 설치/제거 경험을 제공하며, 스토어 정책을 준수하는 가장 확실한 방법입니다.
    -   Visual Studio에서 **"Windows Application Packaging Project"** 템플릿으로 새 프로젝트를 생성합니다.
    -   생성된 프로젝트의 `Dependencies`에 Nuitka로 빌드한 `TTAiClickCell.exe`를 참조로 추가합니다.
    -   `Package.appxmanifest` 파일을 열어 앱 아이콘, 버전, 게시자 정보 등 필수 메타데이터를 입력합니다.
    -   프로젝트를 빌드하여 스토어 제출용 `.msixupload` 파일을 생성합니다.

3.  **파트너 센터 제출**:
    -   파트너 센터에서 앱 이름을 예약하고 새 제출을 시작합니다.
    -   `Packages` 섹션에 생성된 `.msixupload` 파일을 업로드합니다.
    -   스토어 목록에 표시될 설명, 스크린샷, 가격, 연령 등급 등 모든 정보를 상세히 기입합니다.
    -   개인정보처리방침(Privacy Policy) URL을 반드시 제공해야 합니다.
    -   모든 정보 입력 후, **"Submit to the Store"**를 클릭하여 인증을 요청합니다. 인증은 보통 며칠이 소요됩니다.

---
## 9. MVP 범위 정의 (Keep / Backlog / Cut)

MVP(Minimum Viable Product)의 성공적인 출시를 위해, 기능 범위를 명확히 정의하고 개발 우선순위를 설정합니다.

-   **Keep (MVP 핵심 기능)**:
    -   **독립 실행형 애플리케이션**: PySide6 기반의 독립 실행형 앱으로, 4대 핵심 기능(필터, 중복 제거, 조건부 서식, 함수 마법사)에 집중합니다.
    -   **AI 기반 보조 기능**: 자연어 입력을 통해 필터, 조건부 서식, 함수 조건을 생성하는 AI 기능을 핵심 구독 모델로 제공합니다.
    -   **안정성 및 사용성**: 대용량 파일 처리 시의 경고 및 스트리밍 로딩, 실행 취소(Undo/Redo), 로그 표시 등 안정적인 사용 경험을 보장합니다.
    -   **배포 및 온보딩**: MS 스토어를 통한 단일 배포 채널, 간단한 30초 튜토리얼 및 샘플 파일 제공.

-   **Backlog (다음 버전 검토)**:
    -   **Excel 애드인 버전**: 독립 실행형 앱의 시장 반응을 확인한 후, v1.3 이상 버전에서 가설 검증을 거쳐 개발을 검토합니다.
    -   **피벗 테이블/차트 보존**: 기술적 복잡성이 높아 MVP 범위에서는 제외하며, 향후 기능으로 검토합니다.
    -   **고급 AI 기능**: 여러 작업을 한 번에 처리하는 'AI 풀 오토마타' 기능은 v1.2 이후 단계적으로 도입합니다.
    -   **BYOK (Bring Your Own Key)**: AI 기능 사용량이 많은 고급 사용자의 요청이 누적될 경우, 자체 API 키를 사용할 수 있는 옵션을 도입합니다.

-   **Cut (제외 대상)**:
    -   **모든 Excel 기능 구현**: '엑셀의 모든 기능'을 대체하려는 시도는 MVP의 핵심 가치(단순함, 초보자 중심)와 충돌하므로 명확히 제외합니다.
    -   **전문가용 고급 기능**: 고급 피벗, 매크로 편집 등은 대상 사용자와 맞지 않으므로 제외합니다.

---

## 10. 시장 경쟁력 분석 및 전략

경쟁 제품 분석을 통해 TTClickCell의 시장 내 위치와 핵심 전략을 다음과 같이 정의합니다.

-   **강점 (Strengths)**:
    -   **한국어 친화적 UI 및 AI**: 개발 초기부터 한국어 환경을 고려하여 설계된 UI와 한국어 자연어 처리에 특화된 AI 기능은 국내 시장에서 강력한 차별화 포인트입니다.
    -   **쉬운 사용성 (Low Learning Curve)**: 복잡한 Excel 함수나 VBA 매크로 없이 GUI 기반으로 데이터 처리 작업을 자동화할 수 있어, Excel 초보자 및 중급 사용자에게 매력적입니다.
    -   **독립 실행형 앱**: Excel이 설치되지 않은 환경에서도 작동하며, Excel 버전에 구애받지 않는다는 장점이 있습니다. MS 스토어를 통한 간편한 설치 및 업데이트 또한 강점입니다.

-   **약점 (Weaknesses)**:
    -   **통합성 부족**: Excel 애드인 형태의 경쟁 제품(예: Kutools, FormulaBot)과 달리 독립된 앱으로 작동하여, 사용자가 Excel과 앱 사이를 오가며 작업해야 하는 불편함이 있습니다.
    -   **제한된 기능 범위**: 300개 이상의 기능을 제공하는 경쟁 제품 대비 핵심 기능에만 집중하여 기능의 폭이 좁습니다.
    -   **Excel 고유 기능 미지원**: 피벗 테이블, 차트, VBA 매크로 등 Excel의 고급 기능을 지원하지 않아, 전문 사용자의 요구를 충족시키기 어렵습니다.

-   **핵심 전략 (Strategy)**:
    -   **틈새 시장 공략**: 모든 Excel 사용자가 아닌, '반복적인 데이터 정리 및 보고서 작성에 많은 시간을 소요하는 국내 중소기업 실무자 및 공공기관 담당자' 등 특정 사용자 그룹을 초기 타겟으로 설정합니다.
    -   **'쉬운 AI 자동화' 가치 제안**: '복잡한 기능의 집합'이 아닌, 'AI를 통해 가장 빈번한 작업을 가장 쉽게 해결해주는 도구'라는 점을 핵심 가치로 내세웁니다.
    -   **점진적 기능 확장**: 초기 버전의 성공적인 안착 후, 사용자 피드백과 요청에 기반하여 국내 사용자에게 가장 필요한 기능(예: 텍스트 처리, 데이터 마스킹)부터 순차적으로 추가하며 제품을 발전시킵니다.

---

## 11. 향후 아키텍처 개선 과제 (장기)

제품의 완성도와 전문성을 높이기 위한 장기적인 기술 개선 과제 목록입니다.

1.  **CF 적용 모드 토글 (UI/UX)**: 조건부 서식 UI에 '규칙 추가'와 '규칙 교체' 모드를 명시적으로 선택할 수 있는 토글을 제공하여 사용자 오해를 방지합니다.
2.  **Undo/Redo 메모리 최적화**: 현재의 전체 상태 복사 방식(`deepcopy`)은 대용량 데이터에서 메모리 부담이 큽니다. 향후 변경된 내용만 기록하는 'Diff' 기반으로 전환하여 메모리 사용량을 최적화합니다.
3.  **AI 네트워크 안정화**: API 타임아웃, 재시도 로직(Exponential Backoff), 서킷 브레이커 등을 도입하여 네트워크 오류 발생 시에도 사용자 경험(UX)이 저하되지 않도록 방어 로직을 강화합니다.
4.  **보안 강화**: API 키 등 민감 정보를 OS의 자격증명 관리자(Credential Store)나 DPAPI를 통해 더 안전하게 저장하고, 로그에 민감 정보가 노출되지 않도록 마스킹 처리를 추가합니다.
5.  **대용량 파일 처리 성능 개선**: `openpyxl`의 스타일 복사 로직을 최적화하고, 향후 더 빠른 파일 파서(예: `Arrow`, `pyxlsb`) 도입을 검토하여 대용량 파일 로딩 및 저장 속도를 개선합니다.
6.  **테스트 자동화**: 서식/매크로 보존, CF 모드, .xls 변환 등 주요 기능에 대한 회귀 테스트 세트를 표준화하고, CLI와 GUI를 통합하는 테스트 시나리오를 스크립트화하여 빌드 안정성을 확보합니다.
7.  **패키징 및 라이선스 고지 자동화**: 빌드 파이프라인에서 `LICENSE` 및 `NOTICE` 파일을 자동으로 패키지에 포함시켜, 라이선스 고지 의무를 누락 없이 준수하도록 합니다.
8.  **의존성 관리 및 호환성 확보**: `requirements.txt` 등을 통해 라이브러리 버전을 고정하고, 다양한 OS와 오피스 버전 조합의 테스트 매트릭스를 구성하여 호환성을 확보합니다.
9.  **다국어 리소스 관리**: CI(Continuous Integration) 단계에서 한국어/영어 언어팩의 키가 동일한지 자동으로 검사하는 로직을 추가하여 번역 누락을 방지합니다.