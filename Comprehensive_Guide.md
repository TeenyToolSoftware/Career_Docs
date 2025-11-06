# TTClickCell 종합 기술 가이드 - Part 1: 개요 및 개발 환경

- **문서 버전**: 1.1
- **파트**: 1/6

---

## 1. 개요 (Overview)

### 1.1. 프로젝트 정의
- **프로젝트명**: TTClickCell
- **설명**: Excel 파일의 반복 작업(필터, 중복 제거, 조건부 서식, 함수 연산 등)을 GUI에서 빠르게 수행하고, **AI 보조**(자연어 필터/서식/함수/요약)를 제공하는 데스크톱 애플리케이션.
- **플랫폼**: Windows (PySide6 기반 Qt GUI)

### 1.2. 설계 목적 및 핵심 가치
- **목표**: 복잡한 Excel 기능에 익숙하지 않은 사용자를 위해, 직관적인 GUI와 AI를 통해 데이터 처리 및 자동화의 진입 장벽을 낮춥니다.
- **핵심 가치**:
    - **직관성**: 복잡한 Excel 함수나 VBA 매크로 학습 없이, GUI 기반으로 데이터 처리 작업을 자동화합니다.
    - **AI 기반 자동화**: "매출이 1000 이상인 행은 노란색으로" 와 같은 자연어 명령을 통해 필터, 조건부 서식 등의 규칙을 자동으로 생성합니다.
    - **원본 서식 보존**: 데이터 편집 후 저장 시, 원본 파일의 차트, 이미지, 매크로 등 서식을 최대한 보존하여 작업의 연속성을 보장합니다.

### 1.3. 대상 사용자
- **주요 타겟**: 반복적인 데이터 정리 및 보고서 작성에 많은 시간을 소요하는 국내 중소기업 실무자.
- **확장 타겟**: 유사한 데이터 처리 패턴을 가진 공공기관 담당자 및 Excel 초중급 사용자.

---

## 2. 실행 및 개발 환경

### 2.1. 요구 사항
- Python 3.12+ (권장)
- Windows 10/11
- 필수 패키지: `PySide6`, `pandas`, `numpy`, `openpyxl`, `xlrd`, `pyarrow`, `requests`, `pywin32`

### 2.2. 개발 환경 구축
```powershell
# 1. 저장소를 복제한 디렉토리로 이동합니다.

# 2. Python 가상환경을 생성하고 활성화합니다.
python -m venv .venv
.venv\Scripts\Activate.ps1

# 3. 필수 패키지를 설치합니다.
pip install -r requirements.txt 
# (만약 requirements.txt가 없다면 아래 명령어로 설치)
# pip install PySide6 pandas numpy openpyxl xlrd pyarrow requests pywin32
```

### 2.3. 실행
```powershell
python .\TTAiClickCell.py
```

### 2.4. 빌드 (Nuitka → Standalone)
- **목표**: Python 소스 코드를 C++ 코드로 변환 후 컴파일하여 배포 가능한 실행 폴더를 생성합니다.
- **명령어**:
  ```powershell
  python -m nuitka .\TTAiClickCell.py --standalone --enable-plugin=pyside6 --include-qt-plugins=platforms,styles --output-dir=dist_win --windows-console-mode=disable
  ```
- **배포**: 생성된 `dist_win` 폴더는 이후 MSIX 패키징(Visual Studio Packaging Project 등)을 통해 Microsoft Store에 배포됩니다.

---

## 3. 디렉토리 구조

```
TTClickCell/
    TTAiClickCell.py         # 메인 애플리케이션 진입점
    core/                    # 핸들러, 유틸 등 핵심 로직
        action_handler.py    # 데이터 변경 액션 처리
        ai_handler.py        # AI 연동 처리
        file_handler.py      # 파일 입출력 및 상태 관리
        state_manager.py     # Undo/Redo 상태 관리
        ui_builder.py        # 메인 UI 구조 생성
        ...
    modules/                 # 데이터 처리 엔진 모듈
        clickcell_engine.py  # Excel 파일 처리 엔진
        clickcell_model.py   # Qt 테이블 모델
        formula_engine.py    # 함수 마법사 백엔드 로직
        ...
    resources/               # 언어팩, 아이콘 등 리소스
        clickcell_lang_en.json
        clickcell_lang_ko.json
    ui/                      # UI 컴포넌트 (탭, 다이얼로그)
        filter_tab.py
        cf_tab.py
        wizard_tab.py
        ...
```

# TTClickCell 종합 기술 가이드 - Part 2: 시스템 아키텍처

- **문서 버전**: 1.1
- **파트**: 2/6

---

## 4. 시스템 아키텍처 (System Architecture)

### 4.1. 아키텍처 다이어그램 (논리적)
TTClickCell은 역할에 따라 명확하게 분리된 4계층 아키텍처(4-Tier Architecture)를 따릅니다. 이는 각 부분이 독립적으로 작동하고 교체될 수 있도록 하여 유지보수성과 확장성을 높입니다.

1.  **UI Layer (표현 계층)**
    - **역할**: 사용자와의 상호작용을 담당합니다. 사용자의 입력을 받고 처리 결과를 시각적으로 표시합니다.
    - **구성**: `PySide6` 기반의 UI 컴포넌트들. `ui/` 폴더 내의 `*_tab.py`, `setting_dialog.py` 등이 여기에 해당합니다.

2.  **Handler/Controller Layer (제어 계층)**
    - **역할**: UI와 핵심 엔진(비즈니스 로직) 사이의 중재자 역할을 합니다. 사용자의 UI 액션을 해석하여 어떤 비즈니스 로직이 호출되어야 하는지 결정하고, 로직 처리 후 UI를 어떻게 업데이트할지 지시합니다.
    - **구성**: `core/` 폴더 내의 `action_handler.py`, `file_handler.py`, `ai_handler.py` 등.

3.  **Engine/Service Layer (엔진/서비스 계층)**
    - **역할**: 애플리케이션의 핵심 비즈니스 로직을 실제로 수행합니다. 데이터 처리, 계산, 파일 입출력 등 순수한 데이터 연산이 이 계층에서 이루어집니다.
    - **구성**: `modules/` 폴더 내의 `clickcell_engine.py`, `formula_engine.py` 등.

4.  **Data Layer (데이터 계층)**
    - **역할**: 애플리케이션 내에서 사용되는 데이터를 정의하고 관리합니다. 메모리 내 데이터 구조를 포함합니다.
    - **구성**: `pandas.DataFrame`, 그리고 이를 Qt 모델로 감싸는 `ExcelTableModel` (`modules/clickcell_model.py`).

### 4.2. 주요 데이터 흐름 및 컴포넌트 상호작용 (예: 필터 적용)
사용자가 필터 기능을 사용하는 경우, 데이터는 아래와 같은 흐름으로 각 계층을 통과하며 처리됩니다.

1.  **사용자 입력 (UI Layer)**
    - 사용자가 `ui/filter_tab.py` 로 구현된 필터 탭에서 조건을 입력하고 '적용' 버튼을 클릭합니다.

2.  **신호 발생 및 전달 (UI → Handler)**
    - 필터 탭은 `apply_filters_requested` 라는 신호(Signal)를 발생시킵니다. 이 신호는 `core/ui_builder.py`에서 `core/action_handler.py`의 특정 메소드(슬롯)에 연결되어 있습니다.

3.  **흐름 제어 (Handler Layer)**
    - `ActionHandler`는 신호를 받아, `filter_tab.get_conditions()` 메소드를 호출하여 UI에 입력된 모든 필터 조건들을 수집합니다.

4.  **비동기 처리**
    - `ActionHandler`는 데이터 처리 중 UI가 멈추는 현상을 방지하기 위해, `QThread` 기반의 `Worker`(`core/worker.py`)를 생성합니다.
    - 이 워커는 `modules/clickcell_engine.py`의 `apply_filters` 함수를 백그라운드 스레드에서 실행하도록 설정됩니다.

5.  **핵심 로직 수행 (Engine Layer)**
    - `ExcelEngine`의 `apply_filters` 메소드는 현재 `DataFrame`과 `ActionHandler`로부터 전달받은 필터 조건들을 인자로 받아, Pandas를 사용하여 실제 필터링 연산을 수행하고 결과로 새로운 `DataFrame`을 생성합니다.

6.  **상태 관리**
    - `ActionHandler`는 워커의 작업이 성공적으로 완료되면, `core/state_manager.py`의 `push_state_after_action()` 메소드를 호출합니다. 이 메소드는 변경 이전의 데이터 상태를 깊은 복사(deepcopy)하여 Undo 스택에 저장합니다.

7.  **UI 갱신 (Handler → Data → UI)**
    - `ActionHandler`는 `modules/clickcell_model.py`로 구현된 `ExcelTableModel`을 필터링이 완료된 새로운 `DataFrame`으로 업데이트합니다.
    - 모델이 업데이트되면, `layoutChanged` 신호를 발생시켜 자신을 보여주고 있는 모든 뷰(`QTableView`)에게 화면을 새로 그리도록 알려, 최종적으로 사용자는 필터링된 결과를 화면에서 보게 됩니다.

# TTClickCell 종합 기술 가이드 - Part 3: 핵심 데이터 처리 로직

- **문서 버전**: 1.1
- **파트**: 3/6

---

## 5. Excel 처리 파이프라인 (핵심)

### 5.1. 로딩 (`load_workbook`)
- **설계 목표**: Excel 파일을 읽어 단순한 데이터뿐만 아니라, 사용자가 시각적으로 인지하는 서식 정보(스타일, 병합, 조건부 서식 등)까지 내부 데이터 구조로 변환하여 최대한 원본과 유사한 경험을 제공합니다.
- **지원 형식**: `.xlsx` (openpyxl), `.xls` (xlrd)
- **상세 로직**:
    1.  **헤더 자동 탐지**: `_detect_header` 메소드가 파일의 상위 15행을 스캔하여, 문자열 비율, 고유성, 밀도 및 다음 행의 데이터 타입 등을 기준으로 가장 가능성 높은 헤더 행을 자동으로 선정합니다. (`header_row_idx`)
    2.  **데이터 모델화**: 각 시트의 정보를 다음 내부 데이터 구조에 저장합니다.
        - `raw_data_grid`: 각 셀의 값, 정렬, 스타일 객체 등을 포함하는 2차원 리스트.
        - `merge_map`: `(row, col)`을 키로, `(row_span, col_span)`을 값으로 가지는 셀 병합 정보 딕셔너리.
        - `column_widths`: openpyxl이 인식한 열 너비 리스트.
        - `cf_rules_raw`: `modules/format_loader.py`가 원본 파일에서 직접 추출한 조건부 서식 규칙의 목록.

### 5.2. 조건부 서식(CF) 구조 및 처리
- **설계 목표**: 원본 파일의 조건부 서식을 읽어서 보여주고, 사용자가 UI를 통해 추가한 규칙과 병합하여 적용하며, 저장 시 두 규칙의 출처를 모두 보존합니다.
- **규칙의 종류**:
    - **원본 규칙 (Raw)**: `format_loader`가 생성. Excel의 규칙을 거의 그대로 표현. `{"type": "cellIs", "formula":["1000"], "ranges":["A1:A10"], ...}`
    - **UI 규칙 (Simple)**: `ui/cf_tab.py`가 생성. 사용자가 보기 쉬운 형태. `{"col":"매출", "op":"보다 큼", "val":"1000", ...}`
    - **모델 내부 규칙 (Internal)**: `ExcelTableModel`이 UI 규칙을 받아, 실제 평가에 사용하기 위해 변환한 형태. (열 이름 → 열 인덱스, 연산자 문자열 → 연산자 키)
- **평가**: `modules/cf_evaluator.py`의 함수들이 특정 셀 값에 대해 규칙(cellIs, expression, colorScale 등)을 평가하여 참/거짓을 반환합니다.

### 5.3. 저장 (`save_workbook`)
- **설계 목표**: '보존형 저장'을 수행하여 사용자가 TTClickCell에서 지원하지 않는 기능(차트, 이미지, 매크로 등)을 사용했더라도 원본 파일의 해당 요소들이 유실되지 않도록 합니다.
- **상세 로직**:
    1.  **원본 복사**: 원본 파일을 지정된 저장 경로에 그대로 복사합니다. 이 과정에서 차트, 이미지, 매크로 등 모든 미지원 요소가 보존됩니다. (`.xlsm`의 경우 `keep_vba=True` 옵션 사용)
    2.  **데이터 덮어쓰기**: `openpyxl`로 복사된 파일을 열고, 변경된 시트의 데이터만 `DataFrame`의 내용으로 셀 단위로 덮어씁니다.
    3.  **조건부 서식 재적용**: `ws.conditional_formatting`을 한번 비운 뒤, 아래 두 종류의 규칙을 순서대로 다시 적용합니다.
        - (A) **원본 규칙 재방출**: 로딩 시 저장해두었던 `cf_rules_raw`를 다시 적용.
        - (B) **UI 규칙 적용**: 사용자가 UI를 통해 추가/수정한 규칙을 openpyxl 규칙 객체로 변환하여 추가.
    4.  **메타데이터 저장**: UI에서 생성된 규칙들의 고유 서명(해시)을 `_TTCC_META` 라는 숨겨진 시트에 저장합니다. 이를 통해 다음 로딩 시, 어떤 규칙이 원본 규칙이고 어떤 규칙이 UI를 통해 추가되었는지 식별할 수 있습니다.

---

## 6. 필터, 중복 제거, 함수 로직

### 6.1. 필터 (`ExcelEngine.apply_filters`)
- **입력**: `conditions` (`[{col, op, val}]` 형태의 리스트)
- **로직**: 조건 리스트를 순회하며 Pandas `DataFrame`에 필터를 순차적으로 적용합니다. 이 때, 열의 데이터 타입(`datetime`, `numeric`, `string`)에 따라 각기 다른 비교 연산을 수행합니다.
    - **날짜형**: 문자열 값을 날짜로 파싱하여 비교.
    - **숫자형**: 쉼표(,) 등을 제거하고 `float`으로 변환하여 수치 비교.
    - **문자형**: `startswith`, `contains` 등 문자열 메소드를 사용하여 필터링.

### 6.2. 중복 제거 (`ExcelEngine.remove_duplicates`)
- Pandas의 `df.duplicated()` 메소드를 기반으로 작동합니다.
- `cols` 파라미터로 받은 열 리스트를 기준으로 중복을 판단하며, `keep_latest` 옵션에 따라 중복된 항목 중 첫 번째 또는 마지막 항목을 남깁니다.

### 6.3. 함수 엔진 (`modules/formula_engine.py`)
- `FormulaEngine` 클래스는 함수 마법사의 각 기능에 대한 실제 데이터 연산을 담당합니다.
- 모든 함수는 입력으로 `DataFrame`과 파라미터를 받아, 연산 후 결과가 담긴 새로운 `Series` 또는 단일 값을 반환합니다.
    - `concat_columns`: 여러 열의 텍스트를 합침.
    - `conditional_value`: IF 조건에 따라 다른 값을 반환.
    - `lookup_value`: VLOOKUP과 유사하게 다른 시트/파일에서 값을 찾아옴.
    - `calculate_summary`: 합계, 평균 등 집계 연산.

---

## 7. 테이블 모델 및 뷰

### 7.1. `ExcelTableModel(QAbstractTableModel)`
- **역할**: Pandas `DataFrame`이라는 데이터 소스를 `QTableView`가 이해할 수 있는 Qt의 모델-뷰 아키텍처에 맞게 변환해주는 핵심 어댑터 클래스입니다.
- **주요 메소드**:
    - `data()`: 뷰가 특정 셀의 데이터를 요청할 때 호출됩니다. (값, 배경색, 글자색, 정렬 등)
    - `setData()`: 사용자가 뷰에서 셀을 편집할 때 호출되어 `DataFrame`의 값을 변경합니다.
    - `headerData()`: 헤더(열 이름) 정보를 제공합니다.
    - `update_rules()`: UI로부터 새로운 조건부 서식 규칙을 받아 내부 규칙으로 변환하고, 뷰를 갱신하도록 신호를 보냅니다.

### 7.2. `CustomTableView(QTableView)`
- **역할**: 기본 `QTableView`를 상속받아, Excel과 유사한 사용자 경험을 제공하기 위한 커스텀 기능을 추가합니다.
- **주요 기능**:
    - **병합 셀 선택 보정**: 병합된 셀의 일부만 클릭해도 전체 병합 영역이 선택되도록 선택 모델을 보정합니다.
    - **컨텍스트 메뉴**: 오른쪽 클릭 시 '복사', '붙여넣기', '서식 지우기' 등의 메뉴를 제공합니다.

# TTClickCell 종합 기술 가이드 - Part 4: AI 연동 및 애플리케이션 설정

- **문서 버전**: 1.1
- **파트**: 4/6

---

## 8. AI 연동

### 8.1. 아키텍처
- TTClickCell 클라이언트는 AI 모델을 직접 호출하지 않고, Google Cloud Run에 배포된 **프록시 서버**를 통해 AI 기능을 사용합니다.
- **장점**:
    - **프롬프트 관리**: 프롬프트가 변경되거나 개선될 때, 클라이언트 앱을 업데이트하지 않고 서버 측 코드만 수정하면 됩니다.
    - **인증 및 사용량 제어**: 사용자 인증, API 키 관리, 사용량 추적 등 공통 로직을 서버에서 중앙 관리할 수 있습니다.
    - **모델 유연성**: 향후 다른 AI 모델(e.g., GPT, Llama)로 교체하거나 여러 모델을 조합하는 로직을 서버에 쉽게 구현할 수 있습니다.

### 8.2. 프록시 엔드포인트
- **Warm-up**: `GET https://ttclickcell-ai-proxy-oveqdgxtba-du.a.run.app/warmup` (서버의 콜드 스타트 방지)
- **Main**: `POST https://ttclickcell-ai-proxy-oveqdgxtba-du.a.run.app`
  - **Headers**: `Content-Type: application/json`, `Authorization: Bearer <token>`

### 8.3. API 데이터 계약 (Contract)
- **요청 페이로드** (`core/ai_handler.py`의 `_build_ai_input_contract` 참조)
  ```json
  {
    "prompt": {
      "language": "ko",
      "user_id": "<uuid>",
      "source_schema": { 
        "sheet": "Sheet1", 
        "headers": [ 
          { "alias": "c1", "header": "Name", "type": "string", "unique_values": ["Apple", "Banana"] }
        ]
      },
      "task": { 
        "type": "filter", 
        "goal_dsl": "과일 이름이 A로 시작하는 것만 보여줘"
      }
    }
  }
  ```
- **응답 페이로드** (프록시 서버가 AI 응답을 정규화하여 반환)
  ```json
  // 예: 필터 규칙 생성 시
  {
    "ai_response_text": "{\"rules\":[{\"col\":\"Name\",\"op\":\"시작 문자\",\"val\":\"A\"}]}"
  }
  ```

### 8.4. 클라이언트 측 적용 흐름
1.  `AIHandler`가 `_build_ai_input_contract`를 통해 요청 JSON을 생성합니다.
2.  `Worker` 스레드에서 `_call_gemini_api`를 호출하여 프록시 서버에 POST 요청을 보냅니다.
3.  요청이 완료되면 `on_ai_*_finished` 콜백 함수 (예: `on_ai_filter_finished`)가 실행됩니다.
4.  콜백 함수는 응답 JSON을 파싱하여 `filter_tab.set_conditions()` 와 같은 UI 업데이트 함수를 호출합니다.
5.  최종적으로 사용자는 AI가 생성한 규칙이 UI에 채워진 것을 확인합니다.

---

## 9. 구성, 로깅, 보안

### 9.1. 설정 파일 (`config.json`)
- **경로**: `%APPDATA%\TTClickCell\config.json` (Windows 기준)
- **참조**: `core/app_utils.py`의 `load_config()`, `save_config()`
- **주요 키**:
  - `language`: UI 언어 ("ko" 또는 "en")
  - `api_key`: AI 기능 사용을 위한 API 키 (암호화되어 저장)
  - `recent_files`: 최근에 열었던 파일 경로 목록
  - `favorite_folders`: 즐겨찾기 폴더 경로 목록
  - `user_id`: 사용자를 식별하기 위한 고유 UUID

### 9.2. 로깅
- **경로**: `%APPDATA%\TTClickCell\logs\log_pyside_YYYY-MM-DD_HH-MM-SS.txt`
- **설정**: `core/app_utils.setup_logging()`에서 로그 레벨(DEBUG, INFO) 및 포맷을 설정합니다.
- **UI**: `ui/log_window.py`를 통해 UI 내 별도의 창으로 로그를 실시간 확인할 수 있습니다.

### 9.3. 보안
- **API 키 암호화**: `core/app_utils.py`의 `_encrypt_key`, `_decrypt_key` 함수가 Windows 데이터 보호 API(DPAPI)를 사용하여 로컬 설정 파일에 저장된 API 키를 암호화/복호화합니다. 이를 위해 `pywin32` 라이브러리의 `win32crypt` 모듈을 사용합니다.

---

## 10. 기타 주요 기능

### 10.1. Undo/Redo
- `core/state_manager.py`의 `StateManager` 클래스가 담당합니다.
- 데이터에 변경을 가하는 모든 액션(필터, 중복 제거, 편집 등)이 완료된 후, `push_state_after_action()`이 호출되어 변경 후의 `DataFrame` 상태를 `history_stack`에 저장합니다.
- 스택의 최대 크기는 `MAX_HISTORY_SIZE=20`으로 제한됩니다.

### 10.2. 캐싱 및 성능
- **열 데이터 캐싱**: 시트 로딩 후, AI 프롬프트에 컨텍스트를 제공하거나 UI의 콤보 박스 제안을 위해, 문자열 타입 열의 고유값(상위 200개)을 미리 스캔하여 메모리에 캐싱합니다. (`core/file_handler.py`의 `_start_column_data_caching`)
- **대용량 파일 모드**: UI 토글을 통해 대용량 파일 처리 모드를 활성화/비활성화할 수 있으며, 활성화 시에는 파일의 일부 행만 미리보기로 로드하여 메모리 사용량을 조절합니다.

### 10.3. 국제화 (i18n)
- `resources/` 폴더의 `clickcell_lang_ko.json`, `clickcell_lang_en.json` 파일이 모든 UI 텍스트와 내부 문자열을 관리합니다.
- 애플리케이션은 시작 시 설정된 언어에 맞는 JSON 파일을 로드하여 `lang` 객체로 사용합니다.

### 10.4. 예외 처리
- 파일 포맷 오류, 시트 읽기 실패, AI 네트워크 오류 등 예상 가능한 예외는 `try/except` 블록으로 감싸 사용자에게 `QMessageBox`를 통해 알림을 표시하고 로그를 기록합니다.

# TTClickCell 종합 기술 가이드 - Part 5: 유지보수, 테스트, 배포

- **문서 버전**: 1.1
- **파트**: 5/6

---

## 11. 확장 및 유지보수 가이드

이 섹션은 새로운 기능을 추가하거나 기존 기능을 수정할 때 따라야 할 절차를 설명합니다.

### 11.1. 새 필터 연산자 추가 (예: 'is empty')
1.  **언어팩 추가**: `resources/clickcell_lang_*.json` 파일에 `"op_is_empty": "비어 있음"` 과 같은 키와 라벨을 추가합니다.
2.  **엔진 로직 구현**: `modules/clickcell_engine.py`의 `apply_filters()` 메소드에 새로운 연산자 키(`op_is_empty`)에 대한 처리 로직을 추가합니다. (예: `df[col].isnull()`)
3.  **UI 업데이트**: `ui/ui_utils.py`의 `update_operator_and_value_combos()` 함수를 수정하여, 특정 데이터 타입의 열이 선택되었을 때 '비어 있음' 연산자가 콤보 박스에 나타나도록 합니다.

### 11.2. 새 조건부 서식 유형 추가 (예: '상위 10개')
1.  **UI 추가**: `ui/cf_tab.py`에 '상위 10개' 규칙을 위한 UI 요소를 추가합니다.
2.  **모델 변환 로직 추가**: `ExcelTableModel.update_rules()` 메소드에 새로운 UI 규칙을 내부 형식으로 변환하는 로직을 추가합니다.
3.  **평가기 구현**: `modules/cf_evaluator.py`에 '상위 10개'를 평가하는 새로운 함수를 구현합니다. (예: `eval_topN(...)`)
4.  **저장 로직 추가**: `ExcelEngine._apply_cf_rules()` 메소드에 새로운 내부 규칙 형식을 `openpyxl`의 `Rule` 객체로 변환하는 로직을 추가합니다.

### 11.3. 새 함수 마법사 추가 (예: '텍스트 자르기')
1.  **엔진 함수 구현**: `modules/formula_engine.py`에 `split_text(...)` 와 같은 실제 데이터 처리 함수를 구현합니다.
2.  **UI 위젯 생성**: `ui/wizard_tab.py`에 `_create_split_text_widget()` 메소드를 만들어 필요한 UI(자를 열, 구분자, 가져올 순서 등)를 생성합니다.
3.  **흐름 연결**: `wizard_tab`의 `on_task_changed`, `_on_apply_clicked` 등의 메소드에 새로운 함수 위젯을 연결하고, `ActionHandler.on_apply_formula()`에 새로운 `task_name` 분기를 추가합니다.

---

## 12. 테스트 체크리스트

새로운 버전을 배포하기 전, 아래 항목들을 점검하여 회귀(regression) 버그가 없는지 확인해야 합니다.

- [ ] **파일 I/O**: 다양한 헤더 위치, 병합 셀, 여러 시트, 서식/차트가 포함된 `.xlsx` 및 `.xls` 파일 로딩 및 저장.
- [ ] **서식 보존**: 저장 후 다시 열었을 때 값, 셀 스타일, 조건부 서식, 병합, 열 너비가 모두 보존되는지 확인.
- [ ] **핵심 기능**:
    - **필터**: 문자열, 숫자, 날짜 타입에 대한 모든 연산자 정상 동작 및 오류 처리.
    - **중복 제거**: 단일/다중 컬럼 기준, '마지막 항목 유지' 옵션 동작 확인.
    - **조건부 서식**: 원본 규칙 보존, UI 규칙 추가/교체 모드, 여러 규칙 우선순위 적용 확인.
    - **함수 마법사**: Concat, IF, Lookup, 집계 기능으로 새 열이 정상적으로 생성되는지 확인.
- [ ] **사용자 경험 (UX)**:
    - **Undo/Redo**: 모든 데이터 변경 작업에 대해 Undo/Redo가 정확히 동작하는지 확인.
    - **Copy/Paste**: 테이블 뷰 내/외부 복사/붙여넣기 동작 확인.
    - **UI 상태**: 작업 진행 중 UI 비활성화, 완료 후 상태 라벨 업데이트 등.
- [ ] **AI 기능**:
    - **기능별 AI**: 필터, 조건부 서식, 함수 마법사, 데이터 요약 기능이 다양한 자연어 입력에 대해 안정적으로 동작하는지 확인.
    - **예외 처리**: 서버 장애, 인증 토큰 누락, 응답 파싱 실패 시 사용자에게 적절한 알림을 주는지 확인.
- [ ] **기타**:
    - **다국어**: 한국어/영어 전환 시 모든 UI 텍스트가 누락 없이 잘 표시되는지 확인.
    - **템플릿**: 작업 저장 및 불러오기 기능이 모든 컨트롤의 상태를 정확히 복원하는지 확인.

---

## 13. 라이선스 및 배포 유의사항

- **PySide6 (LGPLv3)**: 이 라이선스는 TTClickCell이 상업용闭源软件(closed-source software)으로 배포될 때, 사용자가 PySide6 라이브러리 부분을 직접 수정하고 교체할 수 있도록 보장할 것을 요구합니다. Nuitka의 `--standalone` 모드는 이 요구사항(동적 링크)을 자연스럽게 만족시킵니다.
- **라이선스 고지**: 애플리케이션 내 '정보(About)' 메뉴에 PySide6 사용 및 LGPLv3 라이선스 하에 있음을 명시해야 합니다.
- **파일 포함**: 배포되는 패키지 안에는 `LGPL-3.0.txt` 전문과, `requests`, `numpy` 등 다른 주요 라이브러리들의 라이선스 파일을 `LICENSES/` 와 같은 별도 폴더에 포함하는 것이 좋습니다.
- **아이콘**: `TT_clickcell.ico` 파일이 빌드 결과물에 포함되어 실행 파일 아이콘으로 정상적으로 표시되는지 확인해야 합니다.

# TTClickCell 종합 기술 가이드 - Part 6-1: 모듈 레퍼런스 (Main)

- **문서 버전**: 1.1
- **파트**: 6-1/6

---

## 14. 모듈별 상세 레퍼런스

> 아래는 각 파일의 클래스/함수 시그니처와 간단 설명(존재 시 Docstring)을 나열합니다. (세부 동작은 소스 코드 주석/구현 참조)

### `TTAiClickCell.py`

**Classes**
- **ExcelAutomationApp** (QMainWindow)  
  - Methods:
    - `__init__(self, config, language_pack, session_log_file)`
    - `closeEvent(self, event)` — [MOD] 창을 닫기 전, 모든 백그라운드 스레드를 안전하게 종료합니다.
    - `_restore_window_state(self)` — 이전에 저장된 창 상태(크기, 위치)를 복원합니다.
    - `add_log(self, message)` — 로그 창에 메시지를 추가하고 파일에도 기록합니다.
    - `save_log_to_file(self)` — Saves the content of the session log file to a new timestamped file in the project's logs directory.

# TTClickCell 종합 기술 가이드 - Part 6-2: 모듈 레퍼런스 (Core)

- **문서 버전**: 1.1
- **파트**: 6-2/6

---

### `core/action_handler.py`

**Classes**
- **ActionHandler**   
  - Methods:
    - `__init__(self, main_win)`
    - `apply_all_filters(self, conditions)` — 필터 탭의 모든 조건을 현재 시트에 적용합니다.
    - `remove_duplicates(self, selected_cols, keep_latest)` — 중복 제거 탭의 설정을 현재 시트에 적용합니다.
    - `apply_conditional_formatting(self, rules)`
    - `apply_ai_wizard_tasks(self, tasks)` — AI가 생성한 작업 목록을 순차적으로 실행합니다.
    - `on_apply_formula(self, task_name, params, from_ai)` — 함수 마법사 탭의 설정을 현재 시트에 적용합니다.
    - `transpose_sheet(self)`
    - `adjust_header(self, amount)`
    - `toggle_header_highlight(self)`
    - `on_process_finished(self, result)`
    - `clear_cells(self, model, indexes)`
    - `paste_cells(self, model, start_index, text_data)`
    - `clear_cell_formatting(self, indexes)` — 선택된 셀들의 서식을 지우도록 모델에 요청합니다.
    - `clear_cf_rules_for_selection(self, indexes)` — 선택된 열에 적용된 모든 조건부 서식 규칙을 제거합니다.

### `core/ai_handler.py`

**Classes**
- **AIHandler**   
  - Methods:
    - `__init__(self, main_win)`
    - `cancel_ai_task(self)`
    - `warm_up_server(self)`
    - `run_ai_filter(self, query, **kwargs)`
    - `run_ai_cf(self, query)`
    - `run_ai_wizard(self, task_display_name, query)`
    - `run_ai_summary(self)`
    - `_call_gemini_api(self, prompt)`
    - `_build_ai_input_contract(self, task_type, goal_dsl, wizard_task)`
    - `check_license(self)`
    - `purchase_ai_feature(self)`

### `core/ai_parser.py`

**Module-level functions**
- `extract_json_from_response(text)` — 응답 텍스트에서 JSON 객체 부분만 안정적으로 추출합니다.
- `normalize_and_validate_rule(rule, available_cols)` — 단일 규칙을 검증하고 동의어 사전을 이용해 값을 표준화합니다.
- `parse_ai_response(raw_response, available_cols)` — AI의 원시 응답을 받아 JSON을 추출하고, 각 규칙을 검증/정규화합니다.

### `core/app_utils.py`

**Module-level functions**
- `_encrypt_key(key)` — Windows DPAPI를 사용하여 API 키를 암호화합니다.
- `_decrypt_key(encrypted_key)` — Windows DPAPI를 사용하여 암호화된 API 키를 복호화합니다.
- `resource_path(relative_path)` — PyInstaller로 패키징되었을 때와 개발 환경 모두에서 리소스 경로를 올바르게 찾습니다.
- `get_app_data_dir()` — 운영체제에 맞는 애플리케이션 데이터 저장 경로를 반환합니다.
- `load_config()` — 설정 파일(config.json)을 로드하고 API 키를 복호화합니다.
- `save_config(config)` — API 키를 암호화한 후, 현재 설정을 파일(config.json)에 저장합니다.
- `load_language(lang_code)`
- `setup_logging(level)`
- `restart_program()`

### `core/cache.py`

**Module-level functions**
- `get_cache_dir()`
- `compute_cache_key(...)`
- `has_cache(key)`
- `save_cache_df(key, df)`
- `load_cache_df(key)`

### `core/cf_meta.py`

**Module-level functions**
- `_rule_to_signature(rule)` — 규칙 딕셔너리에서 고유한 서명(해시)을 생성합니다.
- `load_ui_signatures(workbook)` — openpyxl 워크북 객체에서 숨겨진 메타 시트를 읽어 UI 규칙 서명 집합을 반환합니다.
- `save_ui_signatures(workbook, signatures)` — openpyxl 워크북 객체에 UI 규칙 서명 집합을 숨겨진 메타 시트에 저장합니다.

### `core/file_handler.py`

**Classes**
- **FileHandler**
  - Methods:
    - `on_file_loaded(self, result, file_path)`
    - `load_file(self, path)`
    - `save_file_dialog(self)`
    - `on_sheet_tab_changed(self, index)`
    - `update_ui_from_data(self)`
    - `...` (and many others for UI and state updates)

### `core/state_manager.py`

**Classes**
- **StateManager**
  - Methods:
    - `undo(self)`
    - `redo(self)`
    - `push_state_after_action(self)`
    - `clear_history(self)`
    - `update_action_states(self)`

### `core/ui_builder.py`

**Classes**
- **UIBuilder**
  - Methods:
    - `setup_main_window(self)`
    - `setup_ui_components(self)`
    - `connect_signals(self)`
    - `show_toast(self, message, duration)`

### `core/worker.py`

**Classes**
- **Worker(QThread)**: 백그라운드 작업을 처리하기 위한 QThread 클래스.
- **StreamingWorker(QThread)**: 스트리밍 응답을 처리하기 위한 QThread 클래스.

# TTClickCell 종합 기술 가이드 - Part 6-3: 모듈 레퍼런스 (Modules)

- **문서 버전**: 1.1
- **파트**: 6-3/6

---

### `modules/cf_evaluator.py`

**Module-level functions**
- `eval_cellIs(op, formulas, cell_value, ...)`: `cellIs` 타입의 조건부 서식 규칙을 평가합니다.
- `eval_expression(expr, ...)`: `expression` 타입의 수식 기반 규칙을 평가합니다.
- `pick_color_from_colorscale(...)`: `colorScale` 타입의 규칙에 대해 셀 값의 분포에 따른 색상을 계산합니다.

### `modules/clickcell_engine.py`

**Classes**
- **ExcelEngine**
  - Methods:
    - `_get_column_type(self, series)`
    - `_detect_header(self, data_grid, max_scan_rows)`
    - `load_workbook(self, file_path, ...)`
    - `apply_filters(self, df, conditions, ...)`
    - `remove_duplicates(self, df, cols, ...)`
    - `save_workbook(self, path, data_dict, ...)`: 원본 보존형 저장 파이프라인.
    - `reset_header_and_get_df(self, original_grid, new_header_idx, ...)`
    - `transpose_dataframe(self, df)`

### `modules/clickcell_model.py`

**Classes**
- **ExcelTableModel** (QAbstractTableModel)
  - Methods:
    - `__init__(self, ...)`
    - `_eval_cf_for_cell(self, row1, col1, cell_value)`: 특정 셀의 조건부 서식을 평가하고 결과를 캐싱합니다.
    - `data(self, index, role)`: Qt 뷰가 셀 데이터를 요청할 때 호출됩니다.
    - `setData(self, index, value, role)`: 사용자가 뷰에서 셀을 편집할 때 호출됩니다.
    - `update_rules(self, simple_rules)`: UI로부터 받은 규칙을 내부 형식으로 변환하고 모델을 업데이트합니다.
    - `clear_formatting(self, indexes)`

### `modules/format_loader.py`

**Module-level functions**
- `load_cf_rules(xlsx_path, sheet_name)`: `openpyxl`을 사용하여 Excel 파일에서 직접 조건부 서식 규칙을 로드합니다.

### `modules/formula_engine.py`

**Classes**
- **FormulaEngine**: 함수 마법사 기능의 실제 데이터 연산을 담당합니다.
  - Methods:
    - `concat_columns(self, df, ...)`
    - `conditional_value(self, df, ...)`
    - `lookup_value(self, df_left, ...)`
    - `calculate_summary(self, df, ...)`

### `modules/locale_number.py`

**Module-level functions**
- `normalize_number_str(s, profile)`: 지역화된 숫자 형식(예: 1,234.56)을 표준 숫자로 변환합니다.
- `detect_best_profile(values)`: 값 샘플을 기반으로 가장 가능성 있는 숫자 형식을 감지합니다.

# TTClickCell 종합 기술 가이드 - Part 6-4: 모듈 레퍼런스 (UI)

- **문서 버전**: 1.1
- **파트**: 6-4/6

---

### `ui/ai_tab.py`

**Classes**
- **AITab(QWidget)**: AI 데이터 요약 및 구독 관리 UI.

### `ui/cf_tab.py`

**Classes**
- **CFTab(QWidget)**: 조건부 서식 규칙을 수동 또는 AI로 생성/관리하는 UI.
  - `get_rules()`: UI에 입력된 규칙 정보를 수집하여 반환.
  - `add_multiple_rules(rules)`: AI가 생성한 여러 규칙을 UI에 한 번에 추가.

### `ui/custom_table_view.py`

**Classes**
- **CustomTableView(QTableView)**: 셀 병합, 컨텍스트 메뉴 등 확장 기능을 추가한 테이블 뷰.

### `ui/dedup_tab.py`

**Classes**
- **DedupTab(QWidget)**: '중복 제거' 기능 UI.

### `ui/favorites_tab.py`

**Classes**
- **FavoritesTab(QWidget)**: 즐겨찾기 폴더 목록 관리 UI.

### `ui/filter_tab.py`

**Classes**
- **FilterTab(QWidget)**: 필터 조건을 수동 또는 AI로 생성/관리하는 UI.

### `ui/log_window.py`

**Classes**
- **LogWindow(QWidget)**: 로그 표시를 위한 독립적인 플로팅 창.

### `ui/setting_dialog.py`

**Classes**
- **SettingsDialog(QDialog)**: 언어, 시작 화면 등 주요 설정 변경 대화상자.
- **FirstRunLanguageDialog(QDialog)**: 최초 실행 시 언어 선택 대화상자.

### `ui/subscription_dialog.py`

**Classes**
- **SubscriptionDialog(QDialog)**: AI 기능 구독 요금제 선택 대화상자.

### `ui/summary_dialog.py`

**Classes**
- **SummaryDialog(QDialog)**: AI 요약 결과를 표시하는 전용 대화상자.

### `ui/ui_utils.py`

**Module-level functions**
- `update_operator_and_value_combos(...)`: 선택된 열의 데이터 타입에 따라 필터의 연산자 및 값 목록을 동적으로 업데이트.

### `ui/welcome_widget.py`

**Classes**
- **WelcomeWidget(QWidget)**: 프로그램 시작 시 나타나는 웰컴 스크린.

### `ui/wizard_tab.py`

**Classes**
- **FunctionWizardWidget(QWidget)**: '함수 마법사' 기능 UI.
  - `display_tasks(tasks)`: AI가 생성한 함수 작업 목록을 UI에 자동으로 채움.
