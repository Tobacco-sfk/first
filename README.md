import time
import random
import requests
from bs4 import BeautifulSoup
from openai import OpenAI

# -------------------------- 1. 基础防封禁请求配置 --------------------------
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36",
    "Accept-Language": "zh-CN,zh;q=0.9",
    "Referer": "https://www.douyin.com/"
}

# 基础全局防频繁触发：随机延时，降低重试
def random_sleep(min_t=2, max_t=5):
    time.sleep(random.uniform(min_t, max_t))

# -------------------------- 2. 大模型解析配置（对应你日志里的Kimi/豆包） --------------------------
# 豆包 Seed-1.8 客户端
doubao_client = OpenAI(
    api_key="你的豆包API_KEY",
    base_url="https://ark.cn-beijing.volces.com/api/v3",
)

# Kimi k2.5 客户端
kimi_client = OpenAI(
    api_key="你的Kimi_API_KEY",
    base_url="https://api.moonshot.cn/v3",
)

def ai_parse_content(raw_text, use_model="doubao"):
    """AI结构化清洗提取抖音内容，失败自动兜底降级，减少无效重试扣费"""
    prompt = f"""
    请把以下抖音网页原始内容，提取为干净结构化数据：
    标题、文案内容、发布者、点赞数、评论数、发布时间
    没有的数据标注：无
    原始内容：
    {raw_text}
    """
    try:
        if use_model == "doubao":
            resp = doubao_client.chat.completions.create(
                model="seed-1-8",
                messages=[{"role":"user", "content":prompt}],
                temperature=0.1
            )
        else:
            resp = kimi_client.chat.completions.create(
                model="kimi-k2-5",
                messages=[{"role":"user", "content":prompt}],
                temperature=0.1
            )
        return resp.choices[0].message.content
    except Exception as e:
        print(f"模型{use_model}调用失败，自动兜底重试一次")
        random_sleep(3,6)
        # 第一次失败，切换另一个模型兜底
        return ai_parse_content(raw_text, use_model="kimi" if use_model=="doubao" else "doubao")

# -------------------------- 3. 单作品基础获取（合规公开页面） --------------------------
def get_douyin_basic_page(video_url, max_retry=2):
    """限制最大重试次数，避免无限重试疯狂扣费"""
    for attempt in range(1, max_retry+1):
        try:
            random_sleep()
            resp = requests.get(video_url, headers=HEADERS, timeout=15)
            resp.raise_for_status()
            soup = BeautifulSoup(resp.text, "html.parser")
            page_text = soup.get_text(strip=True)
            print("页面获取成功，开始AI解析")
            structured_data = ai_parse_content(page_text)
            return structured_data
        except Exception as e:
            print(f"第{attempt}次获取失败: {str(e)}")
            random_sleep(4,8)
    print("达到最大重试次数，直接放弃本次任务，避免无效消耗")
    return None

# -------------------------- 4. 运行测试 --------------------------
if __name__ == "__main__":
    # 仅填写公开可访问的抖音作品测试链接
    test_video_url = "https://www.douyin.com/video/xxxxxxx"
    
    result = get_douyin_basic_page(test_video_url)
    if result:
        print("最终结构化结果：")
        print(result)
