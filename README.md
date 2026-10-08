## 작업용 저장소 구조

이 저장소는 A·B 담당자가 DataLoader를 함께 구현할 때 참고하는 내부 작업용 저장소입니다.

```text
bus-data-loader/
├── README.md                # A·B 역할, 구현 순서, 공통 데이터 규칙
├── requirements.txt         # 필요한 패키지
├── .gitignore
├── main.py                  # 경로·기준 날짜 설정, 실행·결과 저장
│
├── src/
│   └── data_loader.py       # DataLoader 클래스 구현
│
├── data/
│   ├── raw/                 # 원본 데이터
│   │   ├── bus_stops.csv #버스정류소위치정보 데이터
│   │   ├── living_population.csv #자치구별 서울 생활인구 데이터
│   │   └── districts/ #자치구지도 데이터
│   │       ├── districts.shp
│   │       ├── districts.shx
│   │       ├── districts.dbf
│   │       ├── districts.prj
│   │       └── districts.cpg
│   │
│   └── processed/           # 실행 후 생성되는 결과
│       ├── bus_stops.csv
│       ├── districts.geojson
│       └── validation_report.json
│
└── tests/
    └── test_data_loader.py  # 데이터 결합·집계 결과 검증
```

- `main.py`는 실행 설정과 결과 저장을 담당하고, 데이터 읽기·정제·결합은 `DataLoader`에서 처리합니다.
- `raw/`에는 원본을 보관하고 직접 수정하지 않습니다. 위 CSV 이름에 맞춰 파일을 준비하거나 실행 경로를 조정합니다.
- 자치구 경계의 `.shp`, `.shx`, `.dbf`, `.prj`, `.cpg` 파일은 함께 보관합니다.
- `processed/` 내부 파일은 구현 완료 후 실행하면서 생성합니다. 경계 정보가 포함된 자치구 결과는 GeoJSON으로 저장합니다.

## A·B 작업 분담

| 담당 | 작업 범위 |
|---|---|
| A | 원본 파일 준비, DataLoader의  `__init__()`, `load_data()`, 필요한 패키지 정리 |
| B | `clean_data()`, `merge_data()` |
| 공동 | `run()`, `main.py`, README, 결과 검증 및 오류 처리 기준 합의 |

## 메서드별 작업과 결과

별도의 검증 메서드 없이 원본 데이터 검증은 `clean_data()`에서, 자치구 매핑·집계 결과 검증은 `merge_data()` 마지막에서 수행합니다.

| 메서드 | 작업 | 결과 |
|---|---|---|
| `__init__()` | 파일 경로·기준 날짜 저장, 내부 변수 초기화 | 데이터 변수는 `None`, 검증 기록은 빈 사전 |
| `load_data()` | CSV 2개와 압축 해제된 SHP 파일 읽기 | 원본 표 3개와 원본 행 수 기록 |
| `clean_data()` | 열 이름·자료형 정리, 날짜 선택, 중복·누락·경계 검증 | 정제된 표 3개 |
| `merge_data()` | 정류장 자치구 매핑, 면적 계산, 구별 정류장 집계, 생활인구 연결 및 결과 검증 | 최종 표 2개와 검증 기록 |
| `run()` | 읽기 → 정제 → 병합 순서로 실행 | 정류장 표, 자치구 표, 검증 기록 반환 |

`load_data()`, `clean_data()`, `merge_data()`는 결과를 별도로 반환하지 않고 `self` 속성에 저장합니다. `run()`이 마지막에 결과를 반환합니다. Python 틀의 본문은 `pass`로 남겨두었으므로 이 순서에 맞게 직접 구현합니다.

## 공통 변수명 및 열 이름

- `self.bus_stop_data`: 정류장 표. `stop_id`, `stop_name`, `longitude`, `latitude`를 사용하고 병합 후 `district_code`, `district_name`을 추가합니다.
- `self.population_data`: 선택 날짜의 자치구별 생활인구 표. `date`, `district_code`, `district_name`, `living_population`을 사용합니다. 병합 단계에서는 수정하지 않고 인구 값을 가져다 사용합니다.
- `self.district_data`: 처음에는 자치구 경계 표이며, 병합 후 자치구별 최종 요약표로 저장합니다. 최종 열은 `district_code`, `district_name`, `geometry`, `area_km2`, `living_population`, `bus_stop_count`입니다.
- `self.validation_report`: 원본 행 수, 제외·중복·충돌 행 수, 미배정 정류장 수, 인구가 연결되지 않은 자치구 목록 등을 기록하는 사전입니다.

최종 데이터 표는 **정류장 표와 자치구별 요약표 두 개**입니다. 생활인구 표는 요약표를 만드는 중간 데이터이며 따로 반환하지 않습니다. 원본 파일은 수정하지 않습니다.

## DataLoader 의사코드

아래 코드는 실행 코드가 아니라 전체 흐름을 보여주는 의사코드이며, 지표 계산은 `District` 담당에게 넘기는 구조입니다.

```text
CLASS DataLoader:

    METHOD __init__(bus_file, population_file, district_file, target_date):
        저장: 파일 경로 3개
        저장: 분석 기준 날짜

        bus_stop_data = 없음
        population_data = 없음
        district_data = 없음
        validation_report = 빈 기록


    METHOD load_data():
        bus_stop_data =
            버스정류소 CSV 읽기
            (한글 인코딩 확인, 정류장 ID는 문자열로 읽기)

        population_data =
            생활인구 CSV 읽기
            (자치구 코드는 문자열로 읽기)

        district_data =
            자치구 경계 파일 읽기
            (압축 해제된 districts.shp 및 같은 폴더의 관련 파일 사용)

        원본 행 수를 validation_report에 기록
        파일이 없거나 읽기에 실패하면 오류 발생


    METHOD clean_data():
        각 데이터에 필요한 원본 열이 있는지 확인
        필수 열이 없으면 오류 발생

        # 1. 버스정류소 정리
        버스 데이터의 열 이름을 공통 이름으로 변경:
            stop_id, stop_name, longitude, latitude

        longitude, latitude를 숫자로 변환
        ID 누락 또는 좌표 변환 실패 행을 별도로 기록하고 제외

        완전히 동일한 중복 행 제거
        동일 ID인데 정보가 다른 행은 충돌로 기록하고 확인
        # 이름이 같다는 이유만으로 삭제하지 않음

        # 2. 생활인구 정리
        인구 데이터의 열 이름을 공통 이름으로 변경:
            date, district_code, district_name, living_population

        날짜와 인구의 자료형 변환
        서울시 전체 합계 행 제외
        target_date에 해당하는 행만 선택

        자치구별로 한 행인지 확인
        선택한 날짜에 서울 25개 자치구가 모두 있는지 확인
        누락값, 중복, 0 이하 값이 있으면 기록하고 확인
        # 인구 누락을 0으로 채우지 않음

        # 3. 자치구 경계 정리
        경계 데이터의 열 이름을 공통 이름으로 변경:
            district_code, district_name, geometry

        자치구 코드와 이름 형식을 통일
        좌표계 정보와 경계 도형의 유효성 확인
        서울 25개 자치구 경계인지 확인
        인구와 경계의 자치구 코드 체계를 맞추고 코드가 일치하는지 확인

        제외·중복·충돌 행 수와 이유를 validation_report에 기록
        해결되지 않은 필수 데이터 오류가 있으면 오류 발생


    METHOD merge_data():
        # 1. 정류장의 위치를 공간 데이터로 만들기
        각 정류장의 (longitude, latitude)로 점 생성
        원본 좌표에 맞는 좌표계 지정

        # 2. 정류장을 자치구에 배정
        정류장 점과 자치구 경계의 좌표계를 맞춤
        각 점을 포함하는 자치구를 찾아 공간 결합

        자치구가 없는 정류장 또는 여러 구에 걸린 정류장을 기록
        해당 행의 위치와 경계를 확인한 뒤 배정 여부 결정
        # 임의로 가장 가까운 구에 넣지 않음

        bus_stop_data에 district_code, district_name 추가

        # 3. 자치구 면적 계산
        경계를 적절한 미터 단위 투영 좌표계로 변환
        area_km2 = 경계 면적 / 1,000,000

        # 4. 정류장 수 집계
        stop_counts =
            배정된 정류장을 district_code별로 세기

        # 5. 자치구별 최종 데이터 생성
        district_data =
            경계 데이터에 생활인구를 자치구 코드로 결합
            그 결과에 stop_counts를 자치구 코드로 결합

        정류장이 없는 구의 bus_stop_count는 0으로 처리
        생활인구가 연결되지 않은 구는 오류로 기록

        지도 출력용 경계는 위도·경도 좌표계로 변환
        # 계산한 area_km2 값은 유지


        # 6. 매핑·집계 결과 검증 (merge_data 내부에서 수행)
        확인: 자치구 코드가 중복 없이 25개인가?
        확인: 모든 자치구의 면적과 생활인구가 양수인가?
        확인: 정류장 한 개가 여러 자치구에 중복 배정되지 않았는가?
        확인:
            자치구별 bus_stop_count 합계
            == 자치구가 배정된 정류장 수

        validation_report에 다음 내용 저장:
            원본 정류장 수
            제외된 행 수와 이유
            중복 및 충돌 행 수
            자치구 미배정 정류장 수
            인구가 연결되지 않은 자치구 목록

        해결되지 않은 필수 데이터 오류가 있으면:
            오류 발생


    METHOD run():
        load_data()
        clean_data()
        merge_data()

        RETURN bus_stop_data, district_data, validation_report
```

다른 팀원이 사용하는 흐름은 이렇게 연결됩니다.

```text
loader = DataLoader(
    버스정류소 파일,
    생활인구 파일,
    자치구 경계 파일,
    분석 기준 날짜
)

stops, districts, report = loader.run()

# C·D 담당
정류장 데이터로 BusStop 객체 생성
자치구 데이터로 District 객체 생성
District 객체에서 밀도와 형평성 지표 계산

# E·F 담당
계산된 District 데이터를 Visualizer에 전달
지도와 차트 표시
```

실제 구현 전에 **각 파일의 원본 열 이름과 자치구 코드 체계가 서로 맞는지** 확인하면 됩니다. 코드 체계가 다르면 `merge_data()` 전에 대응표로 통일하는 과정이 필요합니다.
