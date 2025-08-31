# AI Research Assistant – Development Notes

本文件紀錄 **AI Research Assistant** 專案的設計思路、程式架構、待完成功能與 Mock 實作細節。  
目標：以 **爬蟲 → 結構化抽取 →（可選）LLM Function Calling → 固定邏輯分析** 的方式，完成小型研究系統。

---

## 🏗️ 整體架構與流程

```
flowchart TD
    A[Raw Data (HTML/Plain Text)] --> B[Raw Text Cleaning]
    B --> C[HTML Parser → JSON Format]
    C --> D[AIResearchAssistant Class]
    D --> D1[Structured Extraction (Pydantic Schema)]
    D --> D2[Store Extracted Articles]
    D --> D3[Function Calling Layer]
    D3 -->|compare_technologies() or trace_evolution()| E[External Tool / Gemini API]
    D3 -->|Mock Mode| F[Basic JSON/Dict Extraction + Print Summary]
```
- **Raw Text Cleaning**：對 HTML/純文字進行初步清理（移除標籤、多餘空格等）。
- **HTML Parser → JSON Format**：解析 HTML，將結構化資訊（標題、Header、段落等）轉成 JSON/dict。
- **AIResearchAssistant** Class：核心管理模組，內含：
- **Structured Extraction** (Pydantic Schema)：利用 Pydantic Schema 驗證與存放文章資料。
- **Store Extracted Articles**：集中儲存所有已處理文章，便於後續操作。
- **Function Calling Layer**：根據 LLM 輸入自動決定：
    - **呼叫外部工具**（如 Gemini API）
    - **或進入 Mock 模式**（以 JSON/dict 輸出並做簡單摘要）。
---

## 📦 Data Contracts（Pydantic Schemas）

```python
from typing import List, Optional
from pydantic import BaseModel, Field

class Article(BaseModel):
    title: str = Field(description="Article title")
    content: str = Field(description="Cleaned plain text or markdown")
    category: Optional[str] = Field(default=None, description="Domain/topic label, optional")

class ArticleComparison(BaseModel):
    article_a_title: str
    article_b_title: str
    article_a_summary: str
    article_b_summary: str
    common_concepts: List[str] = []
    unique_to_a: List[str] = []
    unique_to_b: List[str] = []
    notes: Optional[str] = None
```
> 註：實際欄位可依作業最終 schema（如 `summary`, `key_concepts`, `applications`）調整。

---

## 🧠 AIResearchAssistant Class（狀態與行為）

```python
from typing import List, Dict
from pydantic import ValidationError

class AIResearchAssistant:
    def __init__(self):
        # 以 Pydantic 物件管理（避免使用 global）
        self.articles: List[Article] = []

    # ---- Data In/Out ----
    def add_article(self, article: Article) -> None:
        self.articles.append(article)

    def list_titles(self) -> List[str]:
        return [a.title for a in self.articles]

    def get_article_by_title(self, title: str) -> Article:
        for a in self.articles:
            if a.title == title:
                return a
        raise ValueError(f"Article not found: {title}")

    def get_articles_by_category(self, category: str) -> List[Article]:
        results = [a for a in self.articles if a.category == category]
        if not results:
            raise ValueError(f"No articles found for category: {category}")
        return results

    # ---- Mocked Structured Extraction（無 API 時）----
    def mock_extract_from_html(self, html: str) -> Dict:
        """
        使用 HTML parser（e.g., BeautifulSoup）擷取 <h1>/<h2>/<p>，
        回傳 dict：{'title': str, 'content': str, 'category': Optional[str]}
        """
        # TODO: implement with BeautifulSoup
        return {"title": "Extracted Title", "content": "Extracted Content", "category": None}
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
4. **Pydantic** → 驗證 → `Article` 物件 → 存入 `AIResearchAssistant.articles`。  
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

## ✅ 待完成清單（Checklist）

- [ ] HTML Parser：`mock_extract_from_html` 真實實作（BeautifulSoup）。  
- [ ] 清理函式：`clean_content`（移除非正文、過多空白、code fences 等）。  
- [ ] `AIResearchAssistant`：錯誤處理與輔助方法完善（如 `remove_article`、`update_article`）。  
- [ ] `compare_technologies`：API 與 Mock 版本（輸入/輸出 JSON 字串）。  
- [ ] `trace_evolution`：API 與 Mock 版本（輸入/輸出 JSON 字串）。  
- [ ] 測試：空資料、找不到標題、category 為空、JSON 格式錯誤等。  
- [ ] 文件化：在 README 中加入使用示例與 CLI/Notebook 範例。  

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
