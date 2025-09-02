# AI Research Assistant – Development Notes

本文件紀錄 **AI Research Assistant** 專案的設計思路、程式架構、待完成功能與 Mock 實作細節。  
目標：以 **爬蟲 → 結構化抽取 →（可選）LLM Function Calling → 固定邏輯分析** 的方式，完成小型研究系統。

---

## 🏗️ 整體架構與流程

```
flowchart TD
A[URL] --> Wikipedia Parser --> B[Raw Text]
B[Raw Text] --> Wikipedia Parser clean_content (HTML Parser) --> C[JSON Format]
C --> D[Agent Class]
D --> D1[Structured Extraction (Pydantic Schema)]
D --> D2[Store Extracted Articles]
D --> D3[Function Calling Layer]
D3 -->|compare_technologies() or trace_evolution()| E[External Tool / Gemini API]
D3 -->|Mock Mode| F[Basic JSON/Dict Extraction + Print Summary]
```
- **Raw Text Cleaning**：對 HTML/純文字進行初步清理（移除標籤、多餘空格等）。
- **HTML Parser → JSON Format**：解析 HTML，將結構化資訊（標題、Header、段落等）轉成 JSON/dict。
- **Agent** Class：核心管理模組，內含：
    - **Structured Extraction** (Pydantic Schema)：利用 Pydantic Schema 驗證與存放文章資料。
    - **Store Extracted Articles**：集中儲存所有已處理文章，便於後續操作。
    - **Function Calling Layer**：根據 LLM 輸入自動決定：
        - **呼叫外部工具**（如 Gemini API）
        - **或進入 Mock 模式**（以 JSON/dict 輸出並做簡單摘要）。

---

## 📖 Wikipedia WebScraper

這是一個基於 **Python + Crawl4AI + OpenAI API** 的 **非同步 Wikipedia 爬蟲**，可以批量抓取文章內容，並將內容轉換成乾淨的文字，以利後續進行 NLP 或 LLM 知識庫建構。

---

### 🚀 功能特色

- 支援 **多網址非同步爬取**（`asyncio`）
- 回傳結構化資訊（標題、字數、連結數量、處理時間）
- 內建 **Markdown → 純文字** 清理功能
- 支援 **安全 API Key 輸入** 與 **OpenAI client 驗證**
- 可擴充至 **自動 chunking / LLM-based extraction**
- **速率限制（Rate Limiting）**：避免過度請求

---

## ✅ 待完成清單（Checklist）

- [ ] WikipediaExtraction Error Case Handling
- [ ] HTML Parser：`mock_extract_from_html` 真實實作（BeautifulSoup）。
- [ ] Asynchronous Multiple Scraping
- [ ] 清理函式：`clean_content`（移除非正文、過多空白、code fences 等）。  
- [ ] `Agent`：錯誤處理與輔助方法完善（如 `remove_article`、`update_article`）。  
- [ ] `compare_technologies`：API 與 Mock 版本（輸入/輸出 JSON 字串）。  
- [ ] `trace_evolution`：API 與 Mock 版本（輸入/輸出 JSON 字串）。  
- [ ] 測試：空資料、找不到標題、category 為空、JSON 格式錯誤等。  
- [ ] 文件化：在 README 中加入使用示例與 CLI/Notebook 範例。
- [ ] `create_mock_wiki_extraction`
- [ ] `batch_extract`

### 📦 安裝需求

#### Dependencies
* crawl4ai>=0.2.0
* openai>=1.0.0
* pydantic>=2.0.0
* python-dotenv>=1.0.0
* requests>=2.25.0
* beautifulsoup4>=4.9.0

**請先安裝必要套件：**
  
```bash
pip install crawl4ai ratelimit beautifulsoup4 markdown openai pandas requests
```

#### 🛠 使用方式
1. API Key 設定
請在執行程式時輸入 API Key，或是預先在環境變數中設定：
`export OPENAI_API_KEY="your_key_here"`

2. 建立 Scraper 並執行
```python
import asyncio
from scraper import WikipediaScraper  # 假設你把 class 存在 scraper.py

urls = [
    "https://en.wikipedia.org/wiki/Natural_language_processing",
    "https://en.wikipedia.org/wiki/Machine_learning"
]

scraper = WikipediaScraper(base_urls=urls)

results = asyncio.run(scraper.scrape_multiple())

for r in results:
    print(r["title"], r["markdown_length"], "chars")
```

#### 📂 回傳結果格式

`scrape_article` 回傳的結果會是 JSON-like dict：
```bash
{
  "title": "Natural language processing",
  "html": "...",
  "markdown": "...",
  "html_length": 15234,
  "markdown_length": 8921,
  "links_found": 245,
  "crawl_time": 1735712456.2381
}
```

#### 🧹 清理內容
你可以使用 clean_content() 將 Markdown 轉換成乾淨的純文字：
```python
raw_markdown = results[0]["markdown"]
clean_text = scraper.clean_content(raw_markdown)

print(clean_text[:500])  # 顯示前 500 字
```

`clean_content` 回傳的結果會是 JSON-like string：
```bash
{
    "title": "Natural language processing",
    "content" {
        "History": "...",
        "See also": "...",
    }
}
```
#### 🔒 OpenAI Client 建立與測試

此專案內建 `create_secure_openai_client()`，會：
- 從環境變數讀取 API key
- 測試是否能正確連線
- 回傳一個 OpenAI client

範例：
```python
from scraper import create_secure_openai_client

client = create_secure_openai_client()

if client:
    response = client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[{"role": "user", "content": "Hello, world!"}]
    )
    print(response.choices[0].message)
```

---

## 📦 Data Contracts（Pydantic Schemas）

### Article Schema
```python
from pydantic import BaseModel, Field
from typing import List

class WikipediaExtraction(BaseModel):
    title: str = Field(description="Article's Title")
    description: str = Field(description="Article's Summary or Description or Overview")
    advantages: List[str] = Field(description="The advantages of the topic mentioned in the article")
    disadvantages: List[str] = Field(description="Known challenges or limitations")
    related_concepts: List[str] = Field(description="Related technology, see also")
    notable_methods: List[str] = Field(description="Notable methods, models, or techniques in this area")
```
> 註：實際欄位可依作業最終 schema（如 `summary`, `key_concepts`, `applications`）調整。

---

## 🧠 Agent Class（狀態與行為）

```python
class Agent:
    def __init__(self, model: str = "gpt-4o-mini"):
        self.articles: List[WikipediaExtraction] = []
        self.model = model
        self.create_secure_openai_client()

    def create_secure_openai_client(self):
    """
    Create OpenAI client with secure API key handling.

    This function:
    1. Looks for OPENAI_API_KEY in environment variables
    2. Tests the connection with a simple API call
    3. Returns the client or None if setup fails
    """
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        print("❌ No OPENAI_API_KEY found in environment variables")
        print("💡 Set environment variable: OPENAI_API_KEY=your_key")
        return None
    try:
        self.client = OpenAI(api_key=api_key)
        # Test connection with a simple API call
        models = client.models.list()
        print("✅ OpenAI client created and tested successfully")

    except Exception as e:
        print(f"❌ OpenAI client creation failed: {e}")
        print("🔍 Check your API key and internet connection")
        self.client = None

    # ---- Data In/Out ----
    def add_article(self, article: WikipediaExtraction) -> None:
        self.articles.append(article)

    # def list_titles(self) -> List[str]:
    #     return [a.title for a in self.articles]

    # def get_article_by_title(self, title: str) -> WikipediaExtraction:
    #     for a in self.articles:
    #         if a.title == title:
    #             return a
    #     raise ValueError(f"Article not found: {title}")

    # def get_articles_by_category(self, category: str) -> List[WikipediaExtraction]:
    #     # category 可能對應 related_concepts，需自定義
    #     results = [a for a in self.articles if category in a.related_concepts]
    #     if not results:
    #         raise ValueError(f"No articles found for category: {category}")
    #     return results

    # ---- Extraction ----
    def extract_structured_data(self, content: str, model: str = "gpt-4o-mini") -> WikipediaExtraction:
        """Use OpenAI structured output to extract data"""
        """
        Extract structured data from raw text using OpenAI's Structured Outputs.

        This replaces the old pattern of:
        response.json()  # Hope it works!

        With guaranteed schema compliance.
        """

        if not client:
            print("⚠️ Demo mode: Would extract structured data with OpenAI API")
            return create_mock_wiki_extraction()
        
        # Create the schema for OpenAI
        schema = {
            "name": "wiki_extraction",
            "schema": WikipediaExtraction.model_json_schema(),
            "strict": False  # This enforces strict schema compliance
        }

        try:
            # The magic happens here - response_format enforces our schema
            response = client.chat.completions.create(
                model=self.model,  # Only gpt-4o and gpt-4o-mini support structured outputs
                messages=[
                    {
                        "role": "system",
                        "content": "Please carefully read and comprehend the entire article content provided below. Extract all relevant information and structure it exactly according to the provided schema."
                    },
                    {
                        "role": "user",
                        "content": f"Please carefully analyze the article {content} below. Even if some fields within the schema are not explicitly provided by the article, use inference to fill in any missing information where reasonable. Then, extract and organize the data from the article according to the schema, ensuring completeness and accuracy based on both explicit statements and logical inference."
                    }
                ],
                response_format={
                    "type": "json_schema",
                    "json_schema": schema
                }
            )

            # Direct validation - no parsing errors possible!
            wiki_extraction = WikipediaExtraction.model_validate_json(response.choices[0].message.content)

            print("✅ Structured extraction successful!")
            print(f"📚 Processed article: {wiki_extraction.title}")
            print(f"💰 Topic's Description: {wiki_extraction.description}")
            print(f"⭐ Topic's Advantages: {wiki_extraction.advantages}")
            print(f"🏷️ Topic's Disadvantages: {wiki_extraction.disadvantages}")

            return wiki_extraction

        except Exception as e:
            print(f"❌ Extraction failed: {e}")
            return self.create_mock_wiki_extraction(content)
        
    def batch_extract(self, articles: List[Dict]) -> List[WikipediaExtraction]:
        """Process multiple articles"""
        extracted = []
        for article in articles:
            extracted.append(self.extract_structured_data(article['markdown']))
        
        return extracted
        

    def create_mock_wiki_extraction(self, raw_content: str):
        """Create mock data for demonstration when API is not available."""
        return WikipediaExtraction(
            title="Null",
            description="Null",
            advantages=[],
            disadvantages=[],
            related_concepts=[],
            notable_methods=[]
        )
```

### Function Calling Layer (對外介面)
```python
def ask_ai(self, query: str, assistant: Agent):
    """
    Function Calling 層，AI Assistant 與外部對話的唯一入口。
    - query: 使用者的任務 (ex: "compare_technologies: LLM, RNN")
    - assistant: 已經擁有 structured articles 的 Agent
    """
    if query.startswith("compare_technologies"):
        # Example: compare_technologies: LLM, RNN
        techs = query.split(":")[1].split(",")
        results = [assistant.get_article_by_title(t.strip()) for t in techs]
        return results

    elif query.startswith("trace_evolution"):
        # Example: trace_evolution: Deep Learning
        topic = query.split(":")[1].strip()
        return assistant.get_articles_by_category(topic)

    else:
        raise ValueError(f"Unknown query: {query}")
```

**Error Handling**：  
- 若 `articles` 為空或找不到標題 → `ValueError`。  
- `category` 可為 `None`；使用時需做空值檢查。  
- 輸入 JSON → 以 `Article.model_validate()` 驗證。  

---

## 🔧 Function Calling Tools（外部工具規格）

> Function Calling 精神：**LLM 決策**是否呼叫工具，但工具本身是**固定、可控**的邏輯；若無 API → 進入 **Mock Mode**。  
> 這兩個函式作為「外部工具」，不封裝在 Class 內（由 Class 提供資料給它們）。

### 1) `compare_technologies`
**Signature**
```python
def compare_technologies(article_a_json: str, article_b_json: str) -> str:
    """
    比較兩個技術文章（以 JSON 字串輸入）並回傳 JSON 字串結果。

    Parameters:
        article_a_json (str): JSON string of Article
        article_b_json (str): JSON string of Article

    Returns:
        str: JSON string of ArticleComparison
    """
```
**輸入/輸出格式**  
- **輸入**：`Article` 物件序列化後的 **JSON 字串**（便於 LLM 或跨模組傳遞）。  
- **輸出**：`ArticleComparison` 物件序列化後的 **JSON 字串**。  

**行為（API / Mock）**  
- **API 可用**：可呼叫 Gemini/OpenAI 對兩篇文章做對比摘要與欄位填充。  
- **Mock**：擷取 `title`、`content` 前 N 字製作摘要；`common_concepts/unique_*` 可基於簡單關鍵字集合比較。  

### 2) `trace_evolution`
**Signature**
```python
from typing import List
def trace_evolution(topic: str, articles_json: List[str]) -> str:
    """
    追蹤單一主題在多篇文章中的演進（以 JSON 字串陣列輸入、JSON 字串輸出）。

    Parameters:
        topic (str): 技術主題
        articles_json (List[str]): list of JSON string of Article

    Returns:
        str: JSON string (e.g., {'timeline': [...], 'key_innovations': [...], 'notes': str})
    """
```
**行為（API / Mock）**  
- **API 可用**：請 LLM 依時間/段落推測演進脈絡與里程碑。  
- **Mock**：以標題排序 + 取每篇第一段作為「里程碑摘要」。  

---

## 🔄 Pipeline 建議（避免 global，清楚資料流）

1. **爬蟲** → 取得 raw HTML / text。  
2. **清理** → `clean_content`（移除雜訊、保留段落）。  
3. **HTML Parser** → 擷取標題/段落 → dict。  
4. **Pydantic** → 驗證 → `Article` 物件 → 存入 `Agent.articles`。  
5. **Function Calling**：
   - 從 Class 取出 `Article`，轉成 **JSON 字串**，作為工具的 **input arguments**。  
   - 呼叫 `compare_technologies` / `trace_evolution`（API 或 Mock）。  
6. **結果呈現**：印出 / 存檔 / 可視化。  

---

## ⚠️ Error Handling 策略

- **沒有任何 Article**：在呼叫比較/演進前先檢查，否則 `ValueError("No articles available")`。  
- **找不到指定標題**：`get_article_by_title` 拋出 `ValueError`。  
- **Category 為空**：允許 `None`；在檢索特定分類時顯示警告或跳過。  
- **JSON 序列化/反序列化**：使用 `Article.model_validate_json()` 與 `model_dump_json()`。  

---

## 🔗 JSON 格式範例（Article）

```json
{
  "title": "Radiosity (Computer Graphics)",
  "content": "Radiosity is a global illumination algorithm ...",
  "category": "Rendering"
}
```

---

## 📎 使用建議
- 內部運算建議以 **dict/Pydantic** 進行；與 LLM 或工具層交互時再轉 **JSON string**。  
- 保持工具（函式）輸入/輸出嚴格定義，方便測試與替換（API ↔ Mock）。  
- 避免使用 global state；由 Class 統一管理狀態與資料。

🔮 未來擴充建議
- 資料儲存：將結果存入 SQLite / MongoDB / CSV
- 自動 Chunking：使用 RegexChunking / SlidingWindowChunking
- 資訊擷取：搭配 LLMExtractionStrategy 自動生成結構化知識
- 前端展示：整合成一個小型 Web NotebookLM Demo
