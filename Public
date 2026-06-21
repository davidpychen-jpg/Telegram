"""
Gradio 介面：抓取「公开资讯观测站（MOPS）」重大讯息公告
            -> 整理成文字 -> 透过 Telegram Bot 发送

与先前 CLI 版本的差异：
  - 不再使用环境变数 TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID
  - 改为在 Gradio 介面上手动输入 Token 与 Chat ID
  - 抓取与发送拆成两个按钮，方便先预览内容再决定是否送出

需要套件：
  pip install gradio requests beautifulsoup4 pandas
"""

import requests
import pandas as pd
from bs4 import BeautifulSoup
import gradio as gr

# ========== Step 1：爬取 MOPS 重大讯息公告 ==========

MOPS_URL = "https://mopsov.twse.com.tw/mops/web/t05sr01_1"
HEADERS = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/124.0.0.0 Safari/537.36"
    ),
    "Referer": "https://mopsov.twse.com.tw/mops/web/index",
    "Accept-Language": "zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7",
}


def fetch_mops_news() -> pd.DataFrame:
    """抓取公开资讯观测站重大讯息公告，回传整理好的 DataFrame"""
    response = requests.get(MOPS_URL, headers=HEADERS, timeout=15)
    response.encoding = "utf-8"
    soup = BeautifulSoup(response.text, "html.parser")

    records = []
    for table in soup.find_all("table"):
        for row in table.find_all("tr"):
            cells = row.find_all("td")
            texts = [c.get_text(strip=True) for c in cells]

            # 栏位顺序通常为：公司代号、公司简称、发言日期、发言时间、主旨
            # 用「公司代号为 4 位数字」过滤掉表头与无效列
            if len(texts) >= 5 and texts[0].isdigit() and len(texts[0]) == 4:
                records.append({
                    "公司代号": texts[0],
                    "公司简称": texts[1],
                    "发言日期": texts[2],
                    "发言时间": texts[3],
                    "主旨":     texts[4],
                })

    return pd.DataFrame(records, columns=["公司代号", "公司简称", "发言日期", "发言时间", "主旨"])


# ========== Step 2：把 DataFrame 组成文字 ==========

def build_message(df: pd.DataFrame) -> str:
    """把 DataFrame 内容整理成适合发送的文字"""
    if df.empty:
        return "📭 目前没有抓到任何重大讯息公告。"

    lines = [f"📢 公开资讯观测站 最新重大讯息（共 {len(df)} 笔）\n"]
    for _, row in df.iterrows():
        lines.append(
            f"【{row['公司代号']} {row['公司简称']}】{row['发言日期']} {row['发言时间']}\n"
            f"{row['主旨']}\n"
        )
    return "\n".join(lines)


# ========== Step 3：发送 Telegram 讯息 ==========

TELEGRAM_LIMIT = 4000  # 单则讯息上限约 4096 字元，留一点余裕避免超过


def send_telegram_message(token: str, chat_id: str, text: str) -> str:
    """将文字透过 Telegram Bot 送出，过长自动切割分批，回传每段的发送结果"""
    if not token or not chat_id:
        return "❌ 请先填写 Telegram Bot Token 与 Chat ID"
    if not text:
        return "❌ 没有可发送的内容，请先按「抓取重大讯息」"

    url = f"https://api.telegram.org/bot{token}/sendMessage"
    results = []
    for i in range(0, len(text), TELEGRAM_LIMIT):
        chunk = text[i:i + TELEGRAM_LIMIT]
        try:
            resp = requests.post(url, data={"chat_id": chat_id, "text": chunk}, timeout=15)
            data = resp.json()
            ok = data.get("ok", False)
            results.append(
                f"片段 {i // TELEGRAM_LIMIT + 1}：{'✅ 成功' if ok else '❌ 失败 - ' + str(data)}"
            )
        except requests.RequestException as e:
            results.append(f"片段 {i // TELEGRAM_LIMIT + 1}：❌ 发送失败 - {e}")

    return "\n".join(results)


# ========== Gradio 介面逻辑 ==========

def on_fetch():
    """按下「抓取重大讯息」：爬取资料，回传表格、文字预览与状态"""
    try:
        df = fetch_mops_news()
    except requests.RequestException as e:
        return pd.DataFrame(), "", f"❌ 抓取失败：{e}"

    message = build_message(df)
    status = f"✅ 共抓到 {len(df)} 笔重大讯息"
    return df, message, status


def on_send(token, chat_id, message):
    """按下「发送至 Telegram」：把目前预览框里的文字送出"""
    return send_telegram_message(token, chat_id, message)


with gr.Blocks(title="MOPS 重大讯息 → Telegram 通知") as demo:
    gr.Markdown("# 📢 公开资讯观测站重大讯息 → Telegram 通知机器人")
    gr.Markdown(
        "1️⃣ 先按「抓取重大讯息」预览内容　"
        "2️⃣ 填入 Telegram Bot Token / Chat ID　"
        "3️⃣ 按「发送至 Telegram」送出（送出前可在文字框内手动修改内容）"
    )

    with gr.Row():
        token_input = gr.Textbox(
            label="Telegram Bot Token",
            placeholder="例如：123456789:AAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
            type="password",
        )
        chatid_input = gr.Textbox(
            label="Telegram Chat ID",
            placeholder="例如：8024792301（群组通常为负数）",
        )

    fetch_btn = gr.Button("🔍 抓取重大讯息", variant="primary")
    status_box = gr.Textbox(label="抓取状态", interactive=False)
    df_output = gr.Dataframe(label="重大讯息列表", interactive=False, wrap=True)
    message_box = gr.Textbox(
        label="整理后文字内容（将作为 Telegram sendMessage 的 text 参数）",
        lines=12,
        interactive=True,
    )

    send_btn = gr.Button("📤 发送至 Telegram", variant="stop")
    send_result = gr.Textbox(label="发送结果", interactive=False)

    fetch_btn.click(
        fn=on_fetch,
        inputs=None,
        outputs=[df_output, message_box, status_box],
    )

    send_btn.click(
        fn=on_send,
        inputs=[token_input, chatid_input, message_box],
        outputs=send_result,
    )


if __name__ == "__main__":
    demo.launch()
