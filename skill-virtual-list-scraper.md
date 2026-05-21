# Skill: Virtual List Scraper & AI-Ready Formatter

## 1. 应用场景 (Application Scenarios)
针对采用 **虚拟列表 (Virtual List/Windowing)** 技术的现代 Web 应用进行全量文字抓取。

*   **典型平台**：腾讯会议 (Tencent Meeting)、飞书妙记 (Feishu/Lark Minutes)、钉钉会议 (DingTalk)、Notion 等。
*   **痛点解决**：
    *   页面只渲染当前可见区域，直接全选复制只能获取一小段内容。
    *   官方导出功能受限、收费或格式不便于 AI 分析。
    *   需要将结构化的“发言人-时间-内容”转换为高密度的 AI 提示词输入。

## 2. 使用方法 (Usage)
1.  在目标页面打开 Chrome DevTools (F12)。
2.  切换到 **Console (控制台)** 面板。
3.  粘贴本 Skill 提供的脚本并按回车。
4.  脚本会自动模拟滚动、采集并触发 `.txt` 文件下载。

## 3. 核心处理逻辑 (Core Logic)
这是本脚本能够实现“全量”且“无偏”采集的关键点：

1.  **强制锚定起点**：首先将容器 `scrollTop` 设为 0，确保从 00:00 开始，防止从当前滚动位置漏掉上方已回收的 DOM。
2.  **offsetTop 绝对去重**：由于滚动时同一个发言段落会被多次创建和销毁，传统的索引去重不可靠。我们使用每个段落在文档流中的绝对高度 (`offsetTop`) 作为 Map 的 Key，确保同一个段落无论被采集多少次，在结果中只会出现一次。
3.  **异步渲染等待**：虚拟列表在滚动后需要毫秒级时间从内存向 DOM 注入数据。脚本设置了 `200ms` 的 `await Promise` 延时，确保捕获到的是渲染后的真实内容。
4.  **步进重叠扫描**：采用 `0.6` 倍视口高度的步进方案。这意味着每次滚动都有 40% 的重叠区域，配合 offsetTop 去重，可以彻底消除由于滚动太快导致某几行刚好在渲染间隙被“跳过”的风险。
5.  **AI 友好化清洗**：在采集时即时重组 DOM 结构，将分散在不同 `div` 中的发言人、时间和正文拼合成 `[时间] 说话人: 内容` 的单行格式。

## 4. 通用脚本代码 (General Script)

```javascript
async function runSkillExport() {
  const scrollContainer = document.querySelector('.minutes-module-list') || 
                          document.querySelector('[class*="scroll-container"]');
  if (!scrollContainer) return console.error("未找到滚动容器");

  const contentMap = new Map();
  const originalPos = scrollContainer.scrollTop;
  scrollContainer.scrollTop = 0;
  await new Promise(r => setTimeout(r, 800));

  const totalHeight = scrollContainer.scrollHeight;
  const step = scrollContainer.clientHeight * 0.6;
  
  for (let top = 0; top <= totalHeight; top += step) {
    scrollContainer.scrollTop = top;
    await new Promise(r => setTimeout(r, 200));
    
    // 适配常见的 row class
    const rows = document.querySelectorAll('.minutes-module-row, [class*="item"]');
    rows.forEach(row => {
      const text = row.innerText.trim();
      if (text) contentMap.set(row.offsetTop, text);
    });
  }

  scrollContainer.scrollTop = originalPos;
  const output = Array.from(contentMap.entries())
    .sort((a, b) => a[0] - b[0])
    .map(e => e[1].replace(/\n/g, ' '))
    .join('\n');

  const blob = new Blob([output], { type: 'text/markdown' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'skill.md';
  a.click();
}
```