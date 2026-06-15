# Chat JJ

전주대학교 팀 프로젝트에서 공지 데이터 수집·정제 파이프라인을 정리한 저장소입니다. 챗봇 검색용 데이터를 만들기 위해 공지 수집, 본문 정제, 이미지 OCR, 마감일 추출, 카테고리 점수화를 다뤘습니다.

## 1. 개요

| 항목 | 내용 |
|---|---|
| 기간 | 2024.03 ~ 2024.06 |
| 형태 | 전주대학교 캡스톤디자인 팀 프로젝트 |
| 목표 | 교내 공지사항과 학사 정보를 수집·정리해 챗봇 검색 데이터로 제공 |
| 저장소 범위 | 공지 크롤링, 본문/OCR 보강, 마감일 추출, 카테고리 점수화 |

Chat JJ는 여러 게시판에 흩어진 공지 정보를 사용자가 직접 찾아야 하는 문제를 줄이기 위해 설계했습니다. 특히 이미지형 공지, 날짜 표현이 다양한 공지, 학사/장학/행사처럼 성격이 다른 공지를 챗봇이 검색할 수 있는 데이터로 바꾸는 데 집중했습니다.

## 2. 이 저장소에서 직접 다룬 작업

- `BeautifulSoup` 기반으로 통합공지·학사공지 목록과 상세 본문을 수집했습니다.
- 목록 단계에서는 `공지 제목 + 링크` 기준으로 중복을 줄이고, 병합 단계에서는 `기본 제목` 기준으로 기존 CSV와 신규 수집분을 맞췄습니다.
- 이미지형 공지를 위해 이미지 링크를 따로 수집하고 OCR 텍스트를 보강했습니다.
- 날짜 패턴을 기준으로 마감일 후보와 모집 상태를 추출했습니다.
- 제목·본문·작성 부서·OCR 텍스트를 함께 쓰는 카테고리 점수화 로직으로 검색 후보를 줄였습니다.
- 프롬프트 분기만으로는 우선순위 개선 근거가 약했던 구간을 확인하고, LLM 응답 이전 단계의 규칙 기반 전처리 비중을 더 크게 검토했습니다.

## 3. 팀 전체 구조 참고

![Chat JJ 전체 시스템 구조](docs/assets/cj-system-overview.svg)

위 도표는 팀 프로젝트 전체 연결 구조를 참고용으로 정리한 자료입니다. 이 저장소에서 직접 다루는 범위는 `CJ_Scraping`에 해당하는 공지 데이터 수집·정제 파이프라인입니다.

| 레포 | 역할 |
|---|---|
| `CJ_Front` | Flutter 앱 화면과 챗봇 UI |
| `CJ_Back` | 인증, 채팅 API, 데이터 저장 |
| `CJ_AI` | 검색 데이터 활용, 응답 생성 |
| `CJ_Scraping` | 공지 크롤링, OCR, 마감일 추출, 카테고리 점수화 |
| `CJ_Server` | 배포와 운영 자료 |

세부 연결 설명은 [팀 전체 구조 참고](docs/system-architecture.md) 문서에 정리했습니다.

## 4. 데이터 파이프라인

![Chat JJ 공지 데이터 파이프라인](docs/assets/cj-data-pipeline.svg)

데이터 파이프라인은 아래 순서로 정리했습니다.

```text
공지 목록 수집
→ 상세 본문 수집
→ 이미지 링크 추출
→ OCR 텍스트 보강
→ 마감일 추출
→ 카테고리 점수화
→ CSV 생성 및 병합
```

통합공지 CSV와 학사공지 CSV는 처음부터 하나로 합치지 않고 별도 입력 축으로 유지한 뒤, 챗봇이 사용하는 검색데이터 단계에서 함께 활용하는 구조로 정리했습니다.

현재 포함된 결과 CSV는 `04_통합공지_카테고리분류_결과.csv` 944건(일반공지 498건, 장학공지 446건)과 `05_학사공지_카테고리분류_결과.csv` 482건입니다. 행 수는 헤더를 제외한 기준입니다.

### 레포지토리 구조

| 파일 | 내용 |
|---|---|
| `01_공지_통합_크롤링_파이프라인.ipynb` | 수집부터 분류까지 한 흐름으로 묶은 통합 노트북 |
| `02_공지_본문_OCR_마감일_추출.ipynb` | 공지 목록·상세 페이지 수집, 본문 추출, 이미지 OCR, 마감일 추출 중심 노트북 |
| `03_공지_카테고리_분류_점수화.ipynb` | 카테고리 기준 정의와 점수화 로직 정리 |
| `04_통합공지_카테고리분류_결과.csv` | 통합공지 944건(일반공지 498건, 장학공지 446건) 카테고리 점수화 결과 CSV |
| `05_학사공지_카테고리분류_결과.csv` | 학사공지 482건 카테고리 점수화 결과 CSV |
| `docs/system-architecture.md` | 팀 전체 구조와 이 저장소 연결 범위 설명 |
| `docs/assets/*.svg` | 팀 전체 구조, 질문 처리 흐름, 데이터 파이프라인 도표 |

## 5. 사용 기술

### 이 저장소에서 직접 확인할 수 있는 기술

| 영역 | 기술 |
|---|---|
| 데이터 수집 | Python, requests, BeautifulSoup |
| 병렬 처리 | ThreadPoolExecutor |
| 데이터 처리 | pandas, 정규표현식, CSV |
| 이미지 보강 | OCR, 이미지 링크 추출 |
| 분류 | 카테고리 기준 설계, 점수화 로직 |

### 팀 전체 연동 참고 기술

| 영역 | 기술 |
|---|---|
| 검색/응답 | ChromaDB, Ollama |
| 서비스 | Flutter, Spring Boot |

## 6. 구현 포인트

### 공지 데이터 수집

전주대학교 통합공지와 학사공지 게시판을 순회하며 제목, 부서, 등록일, 상세 링크를 수집했습니다. 목록 페이지에서는 게시판 유형별 URL을 만들고, `BeautifulSoup`으로 행 단위 데이터를 읽은 뒤 중복 제목·링크를 제거했습니다.

```python
# 02_공지_본문_OCR_마감일_추출.ipynb
url = f"https://www.jj.ac.kr/jj/community/notice{NOTI}.do?mode=list&&articleLimit={limit}&article.offset={offset}"
res = requests.get(url, timeout=5)
res.encoding = 'utf-8'
soup = bs(res.text, "html.parser")
box = soup.find("tbody")

with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(lambda offset: crawl_page(limit, offset, end), offsets))

df = pd.DataFrame(all_rows)
df.drop_duplicates(subset=["공지 제목", "링크"], inplace=True)
```

`01_공지_통합_크롤링_파이프라인.ipynb`와 `02_공지_본문_OCR_마감일_추출.ipynb`에서는 목록 수집 뒤 상세 URL을 다시 열어 본문과 메타데이터를 붙이는 흐름으로 정리했습니다. 병렬 수집으로 페이지별 목록을 먼저 모으고, 일반공지·장학공지 수집 결과를 하나의 데이터프레임으로 합쳐 후속 정제 단계로 넘겼습니다.

### 상세 본문과 메타데이터 보강

목록 페이지에 없는 본문, 작성 부서, 등록일은 상세 URL을 다시 요청해 채웠습니다. 목록 수집 결과만으로는 챗봇 검색 데이터에 필요한 본문 텍스트가 부족했기 때문에, 상세 페이지의 본문 영역과 메타 영역을 따로 읽었습니다.

```python
# 02_공지_본문_OCR_마감일_추출.ipynb
def extract_notice_body(url):
    res = requests.get(url, timeout=5)
    res.encoding = 'utf-8'
    soup = bs(res.text, "html.parser")

    content_div = soup.select_one("div.b-content-box div.fr-view")
    content = content_div.get_text(separator="", strip=True) if content_div else None

    date_span = soup.select_one("li.b-date-box span:nth-of-type(2)")
    dept_span = soup.select_one("li.b-writer-box span:nth-of-type(2)")
    return {
        'content': content,
        'dept': dept_span.get_text(strip=True) if dept_span else None,
        'date': date_span.get_text(strip=True) if date_span else None,
    }
```

이 단계에서 제목·링크 중심의 목록 데이터를 본문·부서·등록일이 포함된 검색용 원천 데이터로 바꿨습니다.

### 이미지형 공지 보강

상세 본문 안의 이미지 링크를 따로 모으고, OCR을 적용해 본문에 없는 텍스트를 `이미지 텍스트` 열에 보강했습니다.

```python
# 02_공지_본문_OCR_마감일_추출.ipynb
def extract_image_links(url):
    res = requests.get(url, timeout=5)
    soup = BeautifulSoup(res.text, "html.parser")
    img_tags = soup.select("div.b-content-box div.fr-view img")
    return ", ".join(urljoin("https://www.jj.ac.kr", img.get("src")) for img in img_tags if img.get("src"))

def ocr_image_links(img_urls):
    img_list = img_urls.split(", ") if isinstance(img_urls, str) else [img_urls]
    texts = []
    for img_url in img_list:
        response = requests.get(img_url.strip())
        img = Image.open(BytesIO(response.content))
        result = reader.readtext(np.array(img), detail=0)
        texts.append(" ".join(result))
    return " ".join(texts) if texts else None
```

`01_공지_통합_크롤링_파이프라인.ipynb`와 `02_공지_본문_OCR_마감일_추출.ipynb`에는 OCR 라이브러리 설치 셀과 이미지/OCR 텍스트 보강 흐름이 함께 남아 있습니다. 포스터형 공지를 일반 텍스트 공지와 같은 검색 후보에 넣기 위한 처리였습니다.

### 마감일 추출

본문과 OCR 텍스트를 같이 정리한 뒤, 정규표현식 기반으로 날짜 후보를 추출하고 신청 기간·모집 마감·선착순 같은 표현을 구조화 열로 분리했습니다.

```python
# 02_공지_본문_OCR_마감일_추출.ipynb
def preprocess_text(text: str) -> str:
    text = re.sub(r'\S+@\S+', '', text)
    text = re.sub(r'\d{2,4}-\d{3,4}-\d{4}', '', text)
    text = re.sub(r'(\d+)\.\s+(\d+)', r'\1.\2', text)
    return re.sub(r'\s+', ' ', text).strip()

def extract_until_deadline(text: str, base_year: int) -> Optional[str]:
    def build_date(year, month, day):
        if year < 100:
            year += 2000
        if 2000 <= year <= 2100 and 1 <= month <= 12 and 1 <= day <= 31:
            return f"{year}-{month:02d}-{day:02d}"
        return None

    patterns = [
        (r'(\d{4})[./년\s]*(\d{1,2})[./월\s]*(\d{1,2})일?\s*까지\s*(신청|접수|마감)',
         lambda m: build_date(int(m[1]), int(m[2]), int(m[3]))),
        (r'(마감|접수종료|신청마감)[\s:：]+(\d{4})?[./년\s]*(\d{1,2})[./월\s]*(\d{1,2})일?',
         lambda m: build_date(int(m[2]) if m[2] else base_year, int(m[3]), int(m[4]))),
        (r'상시모집.*~\s*(\d{4})[./년\s]*(\d{1,2})[월]?',
         lambda m: build_date(int(m[1]), int(m[2]), calendar.monthrange(int(m[1]), int(m[2]))[1])),
    ]
```

`02_공지_본문_OCR_마감일_추출.ipynb`는 목록 수집 뒤 본문/OCR 텍스트를 보강하고, 그 결과를 기준으로 마감일 후보를 다시 계산하는 순서로 구성했습니다. 목록에 보이지 않는 일정 정보를 후단에서 보완한 흐름입니다.

### 카테고리 점수화와 후보 축소

공지 제목, 본문, 작성 부서, OCR 텍스트를 함께 사용해 카테고리 점수를 계산했습니다. 키워드 사전과 가중치를 먼저 두고 질문과 가까운 공지 후보를 앞단에서 좁히는 방식으로 정리했습니다.

```python
# 03_공지_카테고리_분류_점수화.ipynb
TITLE_WEIGHT = 3
CONTENT_WEIGHT = 2
DEPT_WEIGHT = 0
IMAGE_WEIGHT = 2

def calculate_category_scores_by_notice_type(df: pd.DataFrame) -> pd.DataFrame:
    def calculate_score(title, content, dept, image, category_keywords, department_keywords):
        scores = {k: 0 for k in category_keywords.keys()}
        for cat, keywords in category_keywords.items():
            scores[cat] += (sum(title.count(k) for k in keywords) * TITLE_WEIGHT * 10) / (len(title) + 1)
            scores[cat] += (sum(content.count(k) for k in keywords) * CONTENT_WEIGHT * 10) / (len(content) + 1)
            scores[cat] += (sum(image.count(k) for k in keywords) * IMAGE_WEIGHT * 10) / (len(image) + 1)
        return scores
```

이 점수는 질문과 관련 있는 공지를 먼저 좁히는 전처리 단계로 사용했습니다. `03_공지_카테고리_분류_점수화.ipynb`에는 카테고리별 키워드 사전과 점수화 함수가 분리돼 있어, 분류 기준과 가중치 조정 과정을 코드 기준으로 따라갈 수 있습니다.

### 기존 CSV와 신규 수집분 병합

기존 CSV와 신규 수집 결과를 붙인 뒤, 공지 제목을 기준으로 중복을 제거하고 등록일 기준으로 다시 정렬했습니다.

```python
# 02_공지_본문_OCR_마감일_추출.ipynb
def merge_notice_data(new_df: pd.DataFrame, existing_df: pd.DataFrame) -> pd.DataFrame:
    noti_df = pd.concat([
        ex_df[ex_df["등록 번호"] == "공지"],
        df6[df6["등록 번호"] == "공지"]
    ])
    noti_df = noti_df.drop_duplicates(subset=["기본 제목"], keep="first")

    duplicate_titles = set(df6_normal["기본 제목"]) & set(ex_df_normal["기본 제목"])
    df6_normal = df6_normal[~df6_normal["기본 제목"].isin(duplicate_titles)]

    combined_normal = pd.concat([df6_normal, ex_df_normal])
    combined_normal = combined_normal.drop_duplicates(subset=["기본 제목"], keep="first")
    combined_normal = combined_normal.sort_values("등록일", ascending=False)
```

이 병합 단계 때문에 기존 공지는 유지하고 신규 수집분만 덧붙일 수 있었습니다. 결과 CSV를 반복 생성해도 같은 공지가 계속 누적되지 않도록 만든 부분입니다.

## 7. 트러블슈팅

### 1) 이미지형·비정형 공지를 검색 가능한 데이터로 정규화

#### 문제 상황
학교 공지에는 이미지나 포스터 형태로 올라오는 글이 섞여 있었고, 신청 기간·모집 마감·선착순 같은 표현도 공지마다 달랐습니다. 목록 페이지 제목만 모아서는 챗봇 검색에 필요한 정보가 부족했고, 포스터 안 문구는 텍스트 검색에서 빠졌습니다.

#### 원인 탐색
초기 수집 단계에서 안정적으로 모이는 값은 제목과 링크 중심이었고, 실제 질문에 필요한 본문·작성 부서·등록일은 상세 페이지를 다시 열어야 확인할 수 있었습니다. 마감일도 한 가지 패턴으로 고정되지 않아 단순 문자열 처리만으로는 일정 정보를 같은 형식의 열로 정리하기 어려웠습니다.

#### 해결 방안
공지 목록에서 링크를 수집한 뒤 상세 URL을 다시 열어 본문, 부서, 등록일을 보강하는 2단계 수집 흐름을 두었습니다. 본문 안 이미지 링크도 따로 모아 OCR 텍스트를 추출했고, 정규표현식과 날짜 파싱 후처리로 본문 우선, OCR 보조 순서의 마감일 후보를 구조화했습니다. 누락 컬럼은 먼저 채워 두고, 수집 단계와 병합 단계에서 중복 공지를 걸러 반복 실행 시 CSV 형태가 무너지지 않게 맞췄습니다. LLM에 바로 넘기기 전에 공지를 같은 형식의 데이터로 정리하는 쪽이 먼저라고 봤습니다.

#### 적용 결과
제목만 있던 공지가 본문, 작성 부서, 등록일, 이미지 텍스트까지 포함한 행으로 정리됐고, 일정·마감 질문에 필요한 값도 후속 단계에서 다시 쓰기 쉬운 형태로 모을 수 있었습니다. 반복 수집 이후에도 누락 컬럼이나 중복 공지 때문에 결과 CSV가 흔들리지 않도록 정리했습니다.

### 2) 질문과 공지의 연관성을 먼저 좁히기 위한 카테고리 점수화

#### 문제 상황
공지 수가 늘어나자 사용자 질문과 직접 관련 없는 글까지 한 번에 비교하게 되어 답변 후보 범위가 넓어졌습니다. 관련 공지를 앞단에서 줄이지 않으면, LLM이 후단에서 질문과 공지의 관계를 한 번에 판단해야 했습니다.

#### 원인 탐색
프롬프트를 더 잘게 나누는 시도만으로는 검색 후보 자체가 넓은 문제를 줄이기 어려웠습니다. 저장소에서 직접 확인되는 중심 로직도 LLM 튜닝보다 제목·본문·작성 부서·OCR 텍스트를 정리하고 점수화하는 전처리 쪽에 가까웠습니다.

#### 해결 방안
제목, 본문, 작성 부서, OCR 텍스트를 함께 읽되 필드별 가중치를 다르게 두는 카테고리 점수화 로직을 넣었습니다. 질문 의도와 가까운 공지 후보를 먼저 좁힌 뒤, 그 결과를 후단 LLM 응답 단계로 넘기도록 역할을 나눴습니다. LLM에게 모든 관련성을 한 번에 맡기기보다 사람이 정한 카테고리 기준으로 검색 범위를 먼저 줄이는 쪽을 택했습니다.

#### 적용 결과
이 저장소의 직접 구현 범위는 LLM 자체 튜닝이 아니라 검색 후보를 좁히는 데이터 전처리와 점수화로 정리됐습니다. README에서도 프롬프트 실험을 앞세우지 않고, 규칙 기반 후보 축소 뒤 LLM이 응답을 잇는 흐름으로 설명할 수 있게 바꿨습니다.

## 관련 문서

- [팀 전체 구조 참고](docs/system-architecture.md)
- [질문 처리 시퀀스 SVG](docs/assets/cj-runtime-sequence.svg)
