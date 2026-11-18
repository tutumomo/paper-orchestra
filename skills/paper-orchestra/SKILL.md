---
name: paper-orchestra
description: 將零散研究筆記與實驗數據轉化為可編譯的學術論文 LaTeX 與 PDF 完稿。支援兩種運作模式：(1) 調用 PaperOrchestra 本地多 Agent CLI 管線（paper_writing_cli.py）；(2) 由當前對話 Agent 借鏡本專案 5-Agent 工作流與提示詞輕量逐步執行，避免額外消耗大量外部 API 額度。當使用者要求把筆記/實驗結果寫成論文、整理無幻覺 BibTeX 文獻、自動畫統計圖並套入 CVPR/ICLR LaTeX 模板時使用。
---

# PaperOrchestra Skill

本技能讓 AI Coding Agent 能夠利用 **PaperOrchestra** 的自動化論文寫作流程，將使用者手邊零散的構想筆記與實驗紀錄轉換為符合頂會模板格式的 LaTeX 與 PDF 論文初稿。

> **安裝方式（將本技能啟用於你的 Agent 環境）**
> 將整個 `skills/paper-orchestra/` 資料夾複製或建立符號連結至全域或專案技能目錄：
> - **Antigravity / Gemini 全域**：`~/.gemini/config/skills/paper-orchestra/`
> - **Claude Code 全域 / 專案**：`~/.claude/skills/paper-orchestra/` 或 `<project>/.claude/skills/paper-orchestra/`

---

## 執行前決策：選擇運作模式

開始執行前，請先根據使用者的環境與需求選擇（或詢問確認）以下兩種模式之一：

| 模式 | 適用情境 | 優缺點 |
| :--- | :--- | :--- |
| **模式一：完整框架模式 (Full Pipeline Mode)** | 已設定 `.env` API Key（Gemini/Vertex AI + Semantic Scholar）、想直接跑完整無人值守生成與 VLM 審稿迭代 | 完全自動化，保留所有中間產物與多輪審稿紀錄；但單篇執行會消耗較多外部 API 額度且需時約 30–40 分鐘 |
| **模式二：輕量原生模式 (Lightweight Native Agent Mode)** | 不想額外呼叫大量外部 LLM API、僅希望由當前對話中的 Agent 帶頭逐步產出大綱、查證文獻、寫圖表與編譯 LaTeX | 透明可控、省去外部批改 API 費用，可隨時由人類介入微調；需要當前 Agent 逐階段執行 |

---

## 模式一：呼叫完整 PaperOrchestra CLI 框架（Full Pipeline Mode）

### Step 1：整理原料資料夾 (`raw_materials_dir`)
向使用者收集零散筆記、方法草稿、實驗數據表格與圖片（若有），在工作目錄下建立輸入資料夾（例如 `workspace/raw_materials/`），並產出：
1. `idea_sparse.md`：整理核心動機、研究問題定義、方法架構與主要貢獻（可用 `--idea_filename` 指定自訂檔名）。
2. `experimental_log.md`：整理實驗設置、Benchmark 數據表、消融實驗結果與分析觀察（可用 `--experimental_log_filename` 指定自訂檔名）。
3. `figures/`（選填）：若使用者已備妥 `.pdf` / `.png` / `.jpg` 圖表且不打算自動繪圖（`--use_plotting false`），將圖檔放進此子目錄。

### Step 2：環境就緒檢核
執行前先透過終端機檢查必要依賴，缺少任一項即回報並協助補齊：
1. **LaTeX 編譯器**：確認 `pdflatex --version` 與 `bibtex --version` 皆可正常執行。
2. **Python 環境**：確認目標 Python 環境（建議 Python 3.11，`paper_orchestra` Conda 環境）已安裝 `requirements.txt` 套件。
3. **`.env` 憑證**：檢查 repo 根目錄下是否存在 `.env`，且已設定 `GEMINI_API_KEY`（或 `VERTEX_AI_PROJECT` + `VERTEX_AI_LOCATION`），並建議設定 `SEMANTIC_SCHOLAR_API_KEY`。**嚴禁印出或暴露實際 Key 值。**

### Step 3：執行 `paper_writing_cli.py` 與追蹤產物
確認目標模板目錄存在（例如 `templates/cvpr2025` 或 `templates/iclr2025`），執行主程式：

```bash
python paper_writing_cli.py \
  --raw_materials_dir <path_to_raw_materials> \
  --latex_template_dir templates/cvpr2025 \
  --output_dir <path_to_output_dir> \
  --use_plotting true
```

*若在 Windows PowerShell 執行，請將反斜線 `\` 換成反引號 `` ` ``。*
*若使用者已提供現成圖檔於 `raw_materials/figures/`，可移除 `--use_plotting true`（預設為 `False`）。*

執行完成後檢查並向使用者回報：
- 最終 PDF 是否成功產出：`<output_dir>/final_paper.pdf`
- 可編譯的 LaTeX 專案目錄：`<output_dir>/latex_writeup/`
- 若有執行繪圖與審稿：檢查 `<output_dir>/time_breakdown.json` 與 `<output_dir>/content_refinement_workdir/peer_reviews/` 摘要。

---

## 模式二：輕量級本地 Agent 流程移植（Lightweight Native Agent Mode）

當使用者不希望啟動外部批改迴圈消耗額外 API，或希望把這套 5-Agent 思路直接用在當前對話（甚至改寫成技術／工程報告）時，由當前 Agent 嚴格依下列五階段步驟逐步完成：

### 階段 1：大綱與圖表藍圖規劃（借鏡 `methods/prompts/outline_agent.py`）
1. 讀取使用者的構想筆記、實驗紀錄與目標 `templates/<conf>/template.tex`、`guidelines.md`。
2. 輸出結構化大綱（建議存為 `outline.json`），明確列出：
   - 各章節標題與段落核心論點
   - 需要製作的圖表清單（區分統計數據圖 `plot` 與架構概念圖 `diagram`、預期長寬比與資料來源）
   - 需要檢索的文獻關鍵字與方向

### 階段 2：無幻覺文獻查證與 BibTeX 建立（借鏡 `methods/prompts/literature_review_agent.py`）
1. **嚴禁憑記憶捏造引用**。使用網頁搜尋、Semantic Scholar API（可參考 `utils/scholar_utils.py` 中的 `s2_title_search`）或論文查詢工具核對每一篇候選論文的真實標題、作者、發表年份與會議／期刊名稱。
2. 若使用者指定了時間截點（Cutoff Date），過濾掉該日期之後才發表的文獻。
3. 將驗證過的文獻寫入 `references.bib`，並建立清楚的 Citation Key 對照清單（相當於 `citation_map.json`），先完成 Introduction 與 Related Work 的 LaTeX 段落。

### 階段 3：程式化繪圖與視覺檢查（借鏡 `methods/agents/plotting_agent.py`）
1. 針對每個統計數據圖（`plot`），撰寫可重現的 Python `matplotlib` 腳本讀取真實實驗數據繪製並輸出高解析 `.png` 或 `.pdf` 至 `figures/`，**不得臆測或篡改原始數據**。
2. 針對架構概念圖（`diagram`），可使用向量繪圖（如 Mermaid / Graphviz / LaTeX TikZ）或交由圖像工具生成，再透過多模態視覺讀取工具檢查標籤是否清楚、有無重疊截斷。

### 階段 4：LaTeX 模板填寫與章節撰寫（借鏡 `methods/prompts/section_writing_agent.py`）
1. 將目標模板複製到工作區（如 `latex_writeup/`）。
2. 依照大綱逐節補齊 Methodology、Experiments、Ablation Study 與 Conclusion：
   - 所有 `\cite{...}` 必須嚴格對應階段 2 寫入 `references.bib` 的 Key。
   - 所有 `\includegraphics{...}` 必須對應階段 3 實際產出的圖檔路徑與圖說。
   - 表格數值必須與原始實驗紀錄完全一致。

### 階段 5：編譯與視覺排版檢核（借鏡 `methods/prompts/content_refinement_agent.py` 與 `format_agent.py`）
1. 依序執行完整 LaTeX 編譯流程確保引用編號正確：
   ```bash
   pdflatex -interaction=nonstopmode template.tex
   bibtex template
   pdflatex -interaction=nonstopmode template.tex
   pdflatex -interaction=nonstopmode template.tex
   ```
2. 若編譯有誤，讀取 `.log` 定位語法或缺少套件問題並修復。
3. 將產出的 PDF 頁面轉換為圖片（可利用 `utils/pdf_utils.py` 的 `pdf_to_grid_images` 或 PyMuPDF），以視覺檢查確認：
   - 是否有表格超出欄寬（Overfull `\hbox`）
   - 圖片與文字是否有壓字、空白過大或浮動體錯位
   - 頁數與格式是否符合 `guidelines.md` 限制
4. 修正排版問題後重新編譯驗證，最後將最終 `.pdf` 與 `.tex` 原始檔交付給使用者。
