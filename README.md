# AnthropicのMCPによるWebスクレイピング

[![Bright Data Promo](https://github.com/bright-jp/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.jp/)

このガイドでは、オンデマンドのデータ抽出のためにMCPサーバーをセットアップし、開発ツールと接続し、Bright Dataを活用してAI互換のWeb情報を即座に取得する方法を説明します。

- [制約の理解：なぜLLMは現実世界との相互作用に支援が必要なのか](#understanding-the-limitation-why-llms-need-help-with-real-world-interaction)
- [MCPの重要性](#the-importance-of-mcp)
- [Model Context Protocolの理解](#understanding-model-context-protocol)
- [MCPアーキテクチャの解説](#mcp-architecture-explained)
- [独自のMCPサーバーを開発する](#developing-your-own-mcp-server)
- [MCPサーバーを接続する](#connecting-your-mcp-server)
- [プロフェッショナルなWebデータ抽出のためにBright DataのMCPを使う](#using-bright-datas-mcp-for-professional-web-data-extraction)
- [参考資料](#further-reading)

## Understanding the Limitation: Why LLMs Need Help with Real-World Interaction

Large Language Models（LLM）は、広範な学習データセットからテキストを処理・生成することに優れています。しかし、重大な制約があります。それは、外部世界と自然に相互作用できないことです。つまり、ローカルファイルへのアクセス、カスタムスクリプトの実行、Webサイトから最新情報を取得するといった機能が標準では備わっていません。

基本的な例として、Claudeに稼働中のAmazon商品ページから詳細を抽出するよう依頼しても、追加ツールなしでは不可能です。なぜでしょうか。インターネットを閲覧したり外部アクションをトリガーしたりする固有の能力がないためです。

![claude-without-mcp](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/claude-without-mcp.png)

補助ツールがなければ、LLMはリアルタイムデータや外部システムとの統合に依存する実用的なタスクを実行できません。

そこで価値を発揮するのが、[AnthropicのModel Context Protocol（MCP）](https://www.anthropic.com/news/model-context-protocol)です。MCPは、データ抽出ツール、API、スクリプトなどの外部ツールと、LLMが安全かつ標準化された方法で通信できるようにします。

実際の違いは次のとおりです。カスタムMCPサーバーを統合した後、Claudeから直接、構造化されたAmazon商品情報を抽出することに成功しました。

![claude-amazon-product-data-extraction-results](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/claude-amazon-product-data-extraction-results.png)

## The Importance of MCP

- **標準化:** MCPは、LLMベースのシステムが外部ツールやデータに接続するための統一インターフェースを提供します。Web統合をAPIが標準化したのと同様です。これによりカスタム統合の必要性が大幅に減り、開発が加速します。
- **柔軟性とスケーラビリティ:** 開発者は、LLMやホスティングプラットフォームを置き換えても、ツール統合を書き直す必要がありません。MCPは`stdio`など複数の通信方式をサポートしており、さまざまな構成に適応できます。
- **LLM機能の強化:** MCPでLLMを最新データや外部ツールに接続することで、静的な回答を超えられます。文脈に基づいて最新で関連性の高い情報を提供し、現実世界のアクションをトリガーできるようになります。

> **たとえ話**:
> 
> MCPはLLMのためのUSBインターフェースだと考えてください。USBが（キーボード、プリンター、外付けドライブなどの）異なるデバイスを特別なドライバーなしで互換マシンに接続できるようにするのと同様に、MCPは標準化されたプロトコルを用いてLLMを幅広いツールに接続できます。毎回カスタム統合を行う必要はありません。

## Understanding Model Context Protocol

Model Context Protocol（MCP）は、Anthropicが開発したオープンスタンダードで、大規模言語モデル（LLM）が外部ツール、API、データソースと一貫性があり安全な方法で相互作用できるようにします。ユニバーサルコネクターとして機能し、Webサイトデータの抽出、データベースのクエリ、スクリプトの実行といった現実世界のタスクをLLMが行えるようにします。

Anthropicが導入したものですが、MCPはオープンかつ拡張可能であり、誰でも標準を実装・貢献できます。[Retrieval-Augmented Generation（RAG）](https://brightdata.jp/blog/web-data/rag-explained)を扱ったことがある方なら、この概念を理解しやすいはずです。MCPはそのアイデアを発展させ、軽量なJSON-RPCインターフェースによって相互作用を標準化し、モデルがライブデータへアクセスしてアクションを実行できるようにします。

## MCP Architecture Explained

MCPの基盤は、AIモデルと外部機能の間の通信を標準化することにあります。

**中核のアイデア:** 標準化されたインターフェース（通常は`stdio`のようなトランスポート上のJSON-RPC 2.0）により、LLM（クライアント経由）は外部サーバーが公開するツールを発見し、呼び出せるようになります。

MCPはクライアント/サーバーアーキテクチャで動作し、主要コンポーネントは3つです。

1. **MCP Host**: LLMと外部ツール間の相互作用を開始・管理する環境またはアプリケーションです。例として、_Claude Desktop_のようなAIアシスタントや、_Cursor_のようなIDEがあります。
2. **MCP Client**: host内のコンポーネントで、MCP Serverへの接続を確立・維持し、通信プロトコルを処理してデータ交換を管理します。
3. **MCP Server:** （開発者である私たちが作成する）MCPプロトコルを実装し、特定の機能セットを公開するプログラムです。MCP serverはデータベース、Webサービス、またはこのケースではWebサイト（Amazon）と連携できます。serverは機能を標準化された形で公開します:
   - **Tools:** 呼び出し可能な関数（例: _scrape\_amazon\_product_, _get\_weather\_data_）
   - **Resources:** 静的データを取得するための読み取り専用エンドポイント（例: ファイル取得、JSONレコードの返却）
   - **Prompts:** LLMがtoolsやresourcesとやり取りする際のガイドとなる事前定義テンプレート

MCPアーキテクチャ図はこちらです。

![mcp-architecture-diagram-host-client-server-connections](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/mcp-architecture-diagram-host-client-server-connections.png)

_Image Source: [Model Context Protocol](https://modelcontextprotocol.io/introduction)_

この構成では、**host**（Claude DesktopまたはCursor IDE）が**MCP client**を起動し、それが外部の**MCP server**に接続します。serverはtools、resources、promptsを公開し、AIが必要に応じてそれらと相互作用できるようにします。

要するに、ワークフローは次のように動作します。

- ユーザーが _「このAmazonリンクの商品情報を取得して。」_ のようなメッセージを送信します
- MCP clientが、そのタスクを処理できる登録済みtoolを確認します
- clientが構造化されたリクエストをMCP serverへ送信します
- MCP serverが適切なアクションを実行します（例: ヘッドレスブラウザを起動）
- serverが構造化された結果をMCP clientへ返します
- clientが結果をLLMへ渡し、LLMがユーザーに提示します

## Developing Your Own MCP Server

Amazonの商品ページからデータを抽出するPythonのMCP serverを構築しましょう。

![amazon-product-page-example](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/amazon-product-page-example.png)

このserverは2つのtoolを提供します。1つはHTMLをダウンロードするため、もう1つは整理された情報を抽出するためのものです。CursorまたはClaude DesktopのLLMクライアントを介してserverとやり取りします。

### Step 1: Preparing Your Environment

まず、[Python 3](https://www.python.org/downloads/)がインストールされていることを確認してください。次に、仮想環境を作成して有効化します。

```sh
python -m venv mcp-amazon-scraper
# On macOS/Linux:
source mcp-amazon-scraper/bin/activate
# On Windows:
.\mcp-amazon-scraper\Scripts\activate
```

必要なライブラリ（[MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)、[Playwright](https://playwright.dev/python/)、[LXML](https://lxml.de/)）をインストールします。

```sh
pip install mcp playwright lxml
# Install browser binaries for Playwright
python -m playwright install
```

これにより次がインストールされます。

- **mcp**: Model Context Protocolのserverおよびclient向けPython SDKで、JSON-RPC通信の詳細をすべて処理します
- **playwright**: ブラウザ自動化ライブラリで、JavaScriptが多いWebサイトをレンダリングしてスクレイピングするためのヘッドレスブラウザ機能を提供します
- **lxml**: 高速なXML/HTMLパースライブラリで、XPathクエリを使ってWebページから特定データ要素を簡単に抽出できます

要するに、MCP Python SDK（`mcp`）がプロトコルの詳細をすべて処理するため、ClaudeやCursorが自然言語プロンプトから呼び出せるtoolを公開できます。PlaywrightはWebページ（JavaScriptコンテンツを含む）を完全にレンダリングでき、lxmlは強力なHTMLパース機能を提供します。

### Step 2: Starting the MCP Server

`amazon_scraper_mcp.py`という名前のPythonファイルを作成します。まず、必要なモジュールをインポートし、`FastMCP` serverを初期化します。

```python
import os
import asyncio
from lxml import html as lxml_html
from mcp.server.fastmcp import FastMCP
from playwright.async_api import async_playwright

# Define a temporary file path for the HTML content
HTML_FILE = os.path.join(os.getenv("TMPDIR", "/tmp"), "amazon_product_page.html")

# Initialize the MCP server with a descriptive name
mcp = FastMCP("Amazon Product Scraper")

print("MCP Server Initialized: Amazon Product Scraper")
```

これでMCP serverのインスタンスが作成されます。次にtoolを追加します。

### Step 3: Implementing the `fetch_page` Tool

このtoolはURLを入力として受け取り、Playwrightでページへ移動し、コンテンツのロードを待ち、HTMLをダウンロードして一時ファイルへ保存します。

```python
@mcp.tool()
async def fetch_page(url: str) -> str:
    """
    Fetches the HTML content of the given Amazon product URL using Playwright
    and saves it to a temporary file. Returns a status message.
    """
    print(f"Executing fetch_page for URL: {url}")
    try:
        async with async_playwright() as p:
            # Launch headless Chromium browser
            browser = await p.chromium.launch(headless=True)
            page = await browser.new_page()
            # Navigate to the URL with a generous timeout
            await page.goto(url, timeout=90000, wait_until="domcontentloaded")
            # Wait for a key element (e.g., body) to ensure basic loading
            await page.wait_for_selector("body", timeout=30000)
            # Add a small delay for any dynamic content rendering via JavaScript
            await asyncio.sleep(5)

            html_content = await page.content()
            with open(HTML_FILE, "w", encoding="utf-8") as f:
                f.write(html_content)

            await browser.close()
            print(f"Successfully fetched and saved HTML to {HTML_FILE}")
            return f"HTML content for {url} downloaded and saved successfully to {HTML_FILE}."
    except Exception as e:
        error_message = f"Error fetching page {url}: {str(e)}"
        print(error_message)
        return error_message
```

この非同期関数は、Amazonページで起こり得るJavaScriptレンダリングをPlaywrightで処理します。`@mcp.tool()`デコレーターにより、この関数はserver内で呼び出し可能なtoolとして登録されます。

### Step 4: Implementing the `extract_info` Tool

このtoolは、`fetch_page`が保存したHTMLファイルを読み込み、LXMLとXPathセレクターでパースし、抽出した商品詳細を含む辞書を返します。

```python
def _extract_xpath(tree, xpath, default="N/A"):
    """Helper function to extract text using XPath, returning default if not found."""
    try:
        # Use text_content() to get text from node and children, strip whitespace
        result = tree.xpath(xpath)
        if result:
            return result[0].text_content().strip()
        return default
    except Exception:
        return default

def _extract_price(price_str):
    """Helper function to parse price string into a float."""
    if price_str == "N/A":
        return None
    try:
        # Remove currency symbols and commas, handle potential whitespace
        cleaned_price = "".join(filter(str.isdigit or str.__eq__("."), price_str))
        return float(cleaned_price)
    except (ValueError, TypeError):
        return None

@mcp.tool()
def extract_info() -> dict:
    """
    Parses the saved HTML file (downloaded by fetch_page) to extract
    Amazon product details like title, price, rating, features, etc.
    Returns a dictionary of the extracted data.
    """
    print(f"Executing extract_info from file: {HTML_FILE}")
    if not os.path.exists(HTML_FILE):
        return {
            "error": f"HTML file not found at {HTML_FILE}. Please run fetch_page first."
        }

    try:
        with open(HTML_FILE, "r", encoding="utf-8") as f:
            page_html = f.read()

        tree = lxml_html.fromstring(page_html)

        # --- XPath Selectors for Amazon Product Details ---
        title = _extract_xpath(tree, '//span[@id="productTitle"]')
        # Handle different price structures (main price, sale price)
        price_whole = _extract_xpath(tree, '//span[contains(@class, "a-price-whole")]')
        price_fraction = _extract_xpath(
            tree, '//span[contains(@class, "a-price-fraction")]'
        )
        price_str = (
            f"{price_whole}.{price_fraction}"
            if price_whole != "N/A"
            else _extract_xpath(tree, '//span[contains(@class,"a-offscreen")]')
        )  # Fallback to offscreen if needed

        price = _extract_price(price_str)

        # Original price (strike-through)
        original_price_str = _extract_xpath(
            tree, '//span[@class="a-price a-text-price"]//span[@class="a-offscreen"]'
        )
        original_price = _extract_price(original_price_str)

        # Rating
        rating_text = _extract_xpath(tree, '//span[@id="acrPopover"]/@title')
        rating = None
        if rating_text != "N/A":
            try:
                rating = float(rating_text.split()[0])
            except (ValueError, IndexError):
                rating = None

        # Review Count
        reviews_text = _extract_xpath(tree, '//span[@id="acrCustomerReviewText"]')
        review_count = None
        if reviews_text != "N/A":
            try:
                review_count = int(reviews_text.split()[0].replace(",", ""))
            except (ValueError, IndexError):
                review_count = None

        # Availability
        availability = _extract_xpath(
            tree,
            '//div[@id="availability"]//span/text()',
        )

        # Features (bullet points)
        feature_elements = tree.xpath(
            '//div[@id="feature-bullets"]//li//span[@class="a-list-item"]'
        )
        features = [
            elem.text_content().strip()
            for elem in feature_elements
            if elem.text_content().strip()
        ]

        # Calculate Discount
        discount = None
        if price and original_price and original_price > price:
            discount = round(((original_price - price) / original_price) * 100)

        extracted_data = {
            "title": title,
            "price": price,
            "original_price": original_price,
            "discount_percent": discount,
            "rating_stars": rating,
            "review_count": review_count,
            "features": features,
            "availability": availability.strip(),
        }
        print(f"Successfully extracted data: {extracted_data}")
        return extracted_data

    except Exception as e:
        error_message = f"Error parsing HTML: {str(e)}"
        print(error_message)  # Added for logging
        return {"error": error_message}
```

この関数は、LXMLの`fromstring`を使用してHTMLをパースし、堅牢なXPathセレクターで目的の要素を見つけます。

### Step 5: Running the Server

最後に、`amazon_scraper_mcp.py`スクリプトの末尾に次の行を追加して、`stdio`トランスポート機構でserverを起動します。これはClaude DesktopやCursorのようなclientと通信するローカルMCP serverの標準です。

```python
if __name__ == "__main__":
    print("Starting MCP Server with stdio transport...")
    # Run the server, listening via standard input/output
    mcp.run(transport="stdio")
```

### Complete Source Code

```python
import os
import asyncio
from lxml import html as lxml_html
from mcp.server.fastmcp import FastMCP
from playwright.async_api import async_playwright

# Define a temporary file path for the HTML content
HTML_FILE = os.path.join(os.getenv("TMPDIR", "/tmp"), "amazon_product_page.html")

# Initialize the MCP server with a descriptive name
mcp = FastMCP("Amazon Product Scraper")

print("MCP Server Initialized: Amazon Product Scraper")

@mcp.tool()
async def fetch_page(url: str) -> str:
    """
    Fetches the HTML content of the given Amazon product URL using Playwright
    and saves it to a temporary file. Returns a status message.
    """
    print(f"Executing fetch_page for URL: {url}")
    try:
        async with async_playwright() as p:
            # Launch headless Chromium browser
            browser = await p.chromium.launch(headless=True)
            page = await browser.new_page()
            # Navigate to the URL with a generous timeout
            await page.goto(url, timeout=90000, wait_until="domcontentloaded")
            # Wait for a key element (e.g., body) to ensure basic loading
            await page.wait_for_selector("body", timeout=30000)
            # Add a small delay for any dynamic content rendering via JavaScript
            await asyncio.sleep(5)

            html_content = await page.content()
            with open(HTML_FILE, "w", encoding="utf-8") as f:
                f.write(html_content)

            await browser.close()
            print(f"Successfully fetched and saved HTML to {HTML_FILE}")
            return f"HTML content for {url} downloaded and saved successfully to {HTML_FILE}."
    except Exception as e:
        error_message = f"Error fetching page {url}: {str(e)}"
        print(error_message)
        return error_message

def _extract_xpath(tree, xpath, default="N/A"):
    """Helper function to extract text using XPath, returning default if not found."""
    try:
        # Use text_content() to get text from node and children, strip whitespace
        result = tree.xpath(xpath)
        if result:
            return result[0].text_content().strip()
        return default
    except Exception:
        return default

def _extract_price(price_str):
    """Helper function to parse price string into a float."""
    if price_str == "N/A":
        return None
    try:
        # Remove currency symbols and commas, handle potential whitespace
        cleaned_price = "".join(filter(str.isdigit or str.__eq__("."), price_str))
        return float(cleaned_price)
    except (ValueError, TypeError):
        return None

@mcp.tool()
def extract_info() -> dict:
    """
    Parses the saved HTML file (downloaded by fetch_page) to extract
    Amazon product details like title, price, rating, features, etc.
    Returns a dictionary of the extracted data.
    """
    print(f"Executing extract_info from file: {HTML_FILE}")
    if not os.path.exists(HTML_FILE):
        return {
            "error": f"HTML file not found at {HTML_FILE}. Please run fetch_page first."
        }

    try:
        with open(HTML_FILE, "r", encoding="utf-8") as f:
            page_html = f.read()

        tree = lxml_html.fromstring(page_html)

        # --- XPath Selectors for Amazon Product Details ---
        title = _extract_xpath(tree, '//span[@id="productTitle"]')
        # Handle different price structures (main price, sale price)
        price_whole = _extract_xpath(tree, '//span[contains(@class, "a-price-whole")]')
        price_fraction = _extract_xpath(
            tree, '//span[contains(@class, "a-price-fraction")]'
        )
        price_str = (
            f"{price_whole}.{price_fraction}"
            if price_whole != "N/A"
            else _extract_xpath(tree, '//span[contains(@class,"a-offscreen")]')
        )  # Fallback to offscreen if needed

        price = _extract_price(price_str)

        # Original price (strike-through)
        original_price_str = _extract_xpath(
            tree, '//span[@class="a-price a-text-price"]//span[@class="a-offscreen"]'
        )
        original_price = _extract_price(original_price_str)

        # Rating
        rating_text = _extract_xpath(tree, '//span[@id="acrPopover"]/@title')
        rating = None
        if rating_text != "N/A":
            try:
                rating = float(rating_text.split()[0])
            except (ValueError, IndexError):
                rating = None

        # Review Count
        reviews_text = _extract_xpath(tree, '//span[@id="acrCustomerReviewText"]')
        review_count = None
        if reviews_text != "N/A":
            try:
                review_count = int(reviews_text.split()[0].replace(",", ""))
            except (ValueError, IndexError):
                review_count = None

        # Availability
        availability = _extract_xpath(
            tree,
            '//div[@id="availability"]//span/text()',
        )

        # Features (bullet points)
        feature_elements = tree.xpath(
            '//div[@id="feature-bullets"]//li//span[@class="a-list-item"]'
        )
        features = [
            elem.text_content().strip()
            for elem in feature_elements
            if elem.text_content().strip()
        ]

        # Calculate Discount
        discount = None
        if price and original_price and original_price > price:
            discount = round(((original_price - price) / original_price) * 100)

        extracted_data = {
            "title": title,
            "price": price,
            "original_price": original_price,
            "discount_percent": discount,
            "rating_stars": rating,
            "review_count": review_count,
            "features": features,
            "availability": availability.strip(),
        }
        print(f"Successfully extracted data: {extracted_data}")
        return extracted_data

    except Exception as e:
        error_message = f"Error parsing HTML: {str(e)}"
        print(error_message)  # Added for logging
        return {"error": error_message}

if __name__ == "__main__":
    print("Starting MCP Server with stdio transport...")
    # Run the server, listening via standard input/output
    mcp.run(transport="stdio")
```

## Connecting Your MCP Server

serverスクリプトの準備ができたので、Claude DesktopやCursorのようなMCP clientに接続しましょう。

### Setting Up with Claude Desktop

**Step 1:** Claude Desktopを開きます。

**Step 2:** `Settings` -> `Developer` -> `Edit Config`に移動します。これにより、デフォルトのテキストエディターで`claude_desktop_config.json`ファイルが開きます。

![claude-desktop-settings-menu-navigation](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/claude-desktop-settings-menu-navigation.png)

**Step 3:** `mcpServers`キー配下にserverのエントリを追加します。`args`のパスは、`amazon_scraper_mcp.py`ファイルへの絶対パスに置き換えてください。

```json
{
  "mcpServers": {
    "amazon_product_scraper": {
      "command": "python",  // Or python3 if needed
      "args": ["/full/path/to/your/amazon_scraper_mcp.py"], // <-- IMPORTANT: Use the correct absolute path
    }
  }
}
```

**Step 4:** `claude_desktop_config.json`ファイルを保存し、変更を反映するためにClaude Desktopを完全に終了して再起動します。

**Step 5:** Claude Desktopのチャット入力エリアに、小さなtoolsアイコン（ハンマー🔨のようなもの）が表示されるはずです。

![claude-desktop-mcp-tools-icon-interface](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/claude-desktop-mcp-tools-icon-interface.png)

**Step 6:** それをクリックすると、`fetch_page`と`extract_info`のtoolsを備えた「Amazon Product Scraper」が一覧に表示されるはずです。

![claude-available-mcp-tools-dialog-amazon-scraper](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/claude-available-mcp-tools-dialog-amazon-scraper.png)

**Step 7:** 例えば次のようなプロンプトを送信します: _"Get the current price, original price, and rating for this Amazon product: [https://www.amazon.com/dp/B09C13PZX7](https://www.amazon.com/dp/B09C13PZX7)"._

**Step 8:** Claudeは外部toolsが必要だと検知し、まず`fetch_page`、次に`extract_info`を実行する許可を求めます。各toolについて「Allow for this chat」をクリックします。

![mcp-permission-dialog-fetch-page-amazon-tool](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/mcp-permission-dialog-fetch-page-amazon-tool.png)

**Step 9:** 権限を付与すると、MCP serverがtoolsを実行します。その後Claudeは構造化データを受け取り、チャット内に提示します。

![claude-amazon-product-data-extraction-results](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/claude-amazon-product-data-extraction-results-2.png)

### Setting Up with Cursor

Cursor（AIファーストIDE）の手順も同様です。

**Step 1:** Cursorを開きます。

**Step 2:** `Settings` ⚙️へ進み、`MCP`セクションに移動します。

![cursor-ide-add-new-global-mcp-server-settings](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/cursor-ide-add-new-global-mcp-server-settings.png)

**Step 3:** 「+Add a new global MCP Server」をクリックします。これにより`mcp.json`設定ファイルが開きます。serverのエントリを追加し、ここでもスクリプトへの**絶対パス**を使用してください。

![cursor-mcp-json-configuration-file-amazon-scraper](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/cursor-mcp-json-configuration-file-amazon-scraper.png)

**Step 4:** `mcp.json`ファイルを保存すると、「amazon\_product\_scraper」が一覧に表示され、起動・接続されていれば緑のドットが表示されるはずです。

![cursor-ide-configured-amazon-scraper-mcp-settings](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/cursor-ide-configured-amazon-scraper-mcp-settings.png)

**Step 5:** Cursorのチャット機能（`Cmd+l`または`Ctrl+l`）を使用します。

**Step 6:** 例えば次のようなプロンプトを送信します: "_Extract all available product data from this Amazon URL: [https://www.amazon.com/dp/B09C13PZX7](https://www.amazon.com/dp/B09C13PZX7). Format the output as a structured JSON object"_.

**Step 7:** Claude Desktopと同様に、Cursorは`fetch_page`と`extract_info`を実行する権限を求めます。これらのリクエスト（「Run Tool」）を承認します。

**Step 8:** Cursorは対話フローを表示し、MCP toolsの呼び出し、最後に`extract_info` toolが返した構造化JSONデータを提示します。

![cursor-ide-amazon-product-data-extraction-json-results](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/cursor-ide-amazon-product-data-extraction-json-results.png)
以下はCursorからのJSON出力例です。

```json
{
  "title": "Razer Basilisk V3 Customizable Ergonomic Gaming Mouse: Fastest Gaming Mouse Switch - Chroma RGB Lighting - 26K DPI Optical Sensor - 11 Programmable Buttons - HyperScroll Tilt Wheel - Classic Black",
  "price": 39.99,
  "original_price": 69.99,
  "discount_percent": 43,
  "rating_stars": 4.6,
  "review_count": 7782,
  "features": [
    "ICONIC ERGONOMIC DESIGN WITH THUMB REST — PC gaming mouse favored by millions worldwide with a form factor that perfectly supports the hand while its buttons are optimally positioned for quick and easy access",
    "11 PROGRAMMABLE BUTTONS — Assign macros and secondary functions across 11 programmable buttons to execute essential actions like push-to-talk, ping, and more",
    "HYPERSCROLL TILT WHEEL — Speed through content with a scroll wheel that free-spins until its stopped or switch to tactile mode for more precision and satisfying feedback that's ideal for cycling through weapons or skills",
    "11 RAZER CHROMA RGB LIGHTING ZONES — Customize each zone from over 16.8 million colors and countless lighting effects, all while it reacts dynamically with over 150 Chroma integrated games",
    "OPTICAL MOUSE SWITCHES GEN 2 — With zero unintended misclicks these switches provide crisp, responsive execution at a blistering 0.2ms actuation speed for up to 70 million clicks",
    "FOCUS+ 26K DPI OPTICAL SENSOR — Best-in-class mouse sensor with intelligent functions flawlessly tracks movement with zero smoothing, allowing for crisp response and pixel-precise accuracy",
    // ... (other features)
  ],
  "availability": "In Stock"
}
```

これはMCPの柔軟性を示しています。同じserverが、異なるclientアプリケーションでもシームレスに動作します。

## Using Bright Data's MCP for Professional Web Data Extraction

Bright Dataのエンタープライズグレードの[Model Context Protocol（MCP）](https://github.com/bright-jp/brightdata-mcp)ソリューションは、自己管理のMCP serverに伴う複雑さ（プロキシ管理、[アンチボット回避のナビゲーション](https://brightdata.jp/blog/web-data/anti-scraping-techniques)、スケーリングの課題など）を解消し、[AI agents](https://brightdata.jp/use-cases/apps-agents)およびLLMとのシームレスな統合を提供します。

Bright DataのMCPに接続すると、SERP結果や到達が難しいサイトを含むパブリックWebデータへ即時アクセスでき、AIワークフロー向けに最適化されます。

MCPは、[Web Unlocker](https://brightdata.jp/products/web-unlocker)、[SERP API](https://brightdata.jp/products/serp-api)、[Web Scraper API](https://brightdata.jp/products/web-scraper)、[Scraping Browser](https://brightdata.jp/products/scraping-browser)などのツールによって強力なWeb抽出フレームワークを解放し、次を提供します。

- **[AI-Ready Data](https://brightdata.jp/use-cases/data-for-ai):** 事前に構造化されたコンテンツで、前処理は不要です。
- **スケーラビリティと信頼性:** 高ボリュームでも速度低下なく対応します。
- **ブロックおよびCAPTCHA回避:** 高度なアンチボット機能です。
- **グローバルIPカバレッジ:** [Bright Data proxies](https://brightdata.jp/proxy-types)で195か国からアクセス可能です。
- **シームレスな統合:** どのMCP clientでも迅速にセットアップできます。

### Prerequisites for Bright Data MCP

Bright DataのMCP統合を開始する前に、以下が揃っていることを確認してください。

1. **Bright Dataアカウント:** [brightdata.com](https://brightdata.jp/)で登録してください。初回ユーザーにはテスト用の無料クレジットが付与されます。
2. **API Token:** Bright Dataアカウント設定からAPI tokenを安全に取得してください（[User Settings Page](https://brightdata.jp/cp/setting/users)）。
3. **Web Unlocker Zone:** Bright Dataのコントロールパネルで[Web Unlocker proxy](https://docs.brightdata.com/scraping-automation/web-unlocker/quickstart) zoneを作成してください。`mcp_unlocker`のような覚えやすい識別子を選びます（必要に応じて、後で環境変数で変更できます）。
4. **(Optional) Scraping Browser Zone:** 高度なブラウザ自動化機能（例: 複雑なJavaScript操作やスクリーンショット）が必要な場合は、[Scraping Browser zone](https://docs.brightdata.com/scraping-automation/scraping-browser/quickstart)を作成してください。このzoneに提供される認証情報（UsernameとPassword。**Overview**タブ内）を控えておきます。一般的に`brd-customer-ACCOUNT_ID-zone-ZONE_NAME:PASSWORD`の形式です。

### Quickstart: Configuring Bright Data MCP for Claude Desktop

**Step 1:** Bright DataのMCP serverは通常`npx`で実行します（Node.jsに同梱）。必要に応じて[公式サイト](https://nodejs.org/en/download)からNode.jsをインストールしてください。

**Step 2:** Claude Desktop -> `Settings` -> `Developer` -> `Edit Config`（`claude_desktop_config.json`）を開きます。

**Step 3:** `mcpServers`配下にBright Data server設定を挿入します。プレースホルダーを実際の認証情報に置き換えてください。

```json
{
  "mcpServers": {
    "Bright Data": { // Choose a name for the server
      "command": "npx",
      "args": ["@brightdata/mcp"],
      "env": {
        "API_TOKEN": "YOUR_BRIGHTDATA_API_TOKEN", // Paste your API token here
        "WEB_UNLOCKER_ZONE": "mcp_unlocker",     // Your Web Unlocker zone name
        // Optional: Add if using Scraping Browser tools
        "BROWSER_AUTH": "brd-customer-ACCOUNTID-zone-YOURZONE:PASSWORD"
      }
    }
  }
}
```

**Step 4:** 設定ファイルを保存し、Claude Desktopを再起動します。

**Step 5:** Claude Desktopのハンマーアイコン（🔨）にカーソルを合わせます。複数のMCP toolsが利用可能になっているはずです。

![claude-desktop-interface-with-mcp-tools-available](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/claude-desktop-interface-with-mcp-tools-available.png)

スクレイパーを制限する可能性があることで知られるWebサイトであるZillowからデータを抽出してみましょう。Claudeに次のようにプロンプトを入力します: "_Extract key property data in JSON format from this Zillow URL: [https://www.zillow.com/apartments/arverne-ny/the-tides-at-arverne-by-the-sea/ChWHPZ/](https://www.zillow.com/apartments/arverne-ny/the-tides-at-arverne-by-the-sea/ChWHPZ/)_"

![bright-data-mcp-zillow-property-extraction-process](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/bright-data-mcp-zillow-property-extraction-process.png)

必要なBright DataのMCP toolsをClaudeが利用することを許可してください。Bright DataのMCP serverが、基盤となる複雑さ（プロキシローテーション、必要に応じたScraping BrowserによるJavaScriptレンダリング）を管理します。

Bright Dataのserverが抽出を実施し、構造化データを返し、Claudeがそれを提示します。

![zillow-property-data-json-structure-bright-data-mcp](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/zillow-property-data-json-structure-bright-data-mcp.png)

以下は想定される出力のサンプルです。

```json
{
  "propertyInfo": {
    "name": "The Tides At Arverne By The Sea",
    "address": "190 Beach 69th St, Arverne, NY 11692",
    "propertyType": "Apartment building",
    // ... more info
  },
  "rentPrices": {
    "studio": { "startingPrice": "$2,750", /* ... */ },
    "oneBed": { "startingPrice": "$2,900", /* ... */ },
    "twoBed": { "startingPrice": "$3,350", /* ... */ }
  },
  // ... amenities, policies, etc.
}
```

**別の例: Hacker Newsの見出し**

よりシンプルなクエリとして、次のように依頼します: "_Give me the titles of the latest 5 news articles from Hacker News_".

![hacker-news-latest-articles-mcp-extraction-results](https://github.com/bright-jp/web-scraping-with-mcp/blob/main/images/hacker-news-latest-articles-mcp-extraction-results.png)

これは、Bright DataのMCP serverが、動的なコンテンツや強固に保護されたWebコンテンツであっても、AIワークフロー内で直接アクセスすることをどのように簡素化するかを示しています。

## Further Reading

より深い知識のために、AIおよび大規模言語モデル（LLM）に関する過去のガイドを厳選してご紹介します。

- [Top Sources for Finding LLM Training Data](https://brightdata.jp/blog/web-data/llm-training-data)
- [Web Scraping with LLaMA 3: Turn Any Website into Structured JSON](https://brightdata.jp/blog/web-data/web-scraping-with-llama-3)
- [Web Scraping With LangChain and Bright Data](https://brightdata.jp/blog/web-data/web-scraping-with-langchain-and-bright-data)
- [How To Create a RAG Chatbot With GPT-4o Using SERP Data](https://brightdata.jp/blog/web-data/build-a-rag-chatbot)

## Conclusion

AnthropicのModel Context Protocolは、AIシステムが外部世界と相互作用する方法における根本的な転換点を示しています。特定タスク向けにカスタムMCP serverを構築できます。Bright DataのMCP統合はこれをさらに強化し、アンチボット保護を回避し、[AI-readyな構造化データ](https://brightdata.jp/use-cases/data-for-ai)を供給するエンタープライズグレードのWebスクレイピング機能を提供します。

今すぐ[AI solutions](https://brightdata.jp/ai)に登録して、無料でお試しください！