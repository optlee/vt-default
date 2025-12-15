```dataviewjs
// ================= 配置区域 =================

// 1. 你的笔记来源
const source = '"/"'; 

// 2. 日期属性名称
const dateField = "date"; 

// 3. 单元格内容渲染函数
const renderContent = (page) => {
    const priority = page.priority ? page.priority : "⚪";
    // 截取标题，防止太长撑爆格子
    const title = page.title;
    //const title = page.title.length > 8 ? page.title.substring(0, 8) + ".." : page.title;
    const status = page.status;
    const charge = page.charge;
    
    // 【修改点】：使用 <br> 换行，并分别设置样式
    // 第一行大一点显示 Emoji，第二行小一点显示标题
    return `<div style="font-size: 0.8em; text-align:center;">${title}</div>
            <div style="font-size: 0.8em; line-height:1.1;">${priority}</div>
            <div style="font-size: 0.8em; line-height:1.1;">${charge}</div>
            `;
};

// ================= 核心逻辑 (请勿修改) =================

// 创建一个容器 div，用于挂载我们的日历
// 这一步很重要，我们需要一个固定的“画板”来反复擦写
const container = dv.el("div", "");

// 获取所有相关笔记 (只查询一次，提高性能)
const allPages = dv.pages(source).where(p => p[dateField]);

// 定义当前的年份和月份 (初始为今天)
let currentYear = new Date().getFullYear();
let currentMonth = new Date().getMonth();

// --- 渲染主函数 ---
function renderCalendar(year, month) {
    // 1. 清空容器，准备重绘
    container.innerHTML = "";

    // 2. 日期计算逻辑
    const firstDay = new Date(year, month, 1);
    const lastDay = new Date(year, month + 1, 0);
    const numDays = lastDay.getDate();
    const firstDayOfWeek = firstDay.getDay(); // 0 is Sunday

    // 3. 筛选当月笔记
    const currentNotes = allPages.filter(p => {
        const pDate = new Date(p[dateField]);
        return pDate.getFullYear() === year && pDate.getMonth() === month;
    });

    // 4. 创建顶部导航栏 (Flex布局)
    const header = document.createElement("div");
    header.style.display = "flex";
    header.style.justifyContent = "space-between";
    header.style.alignItems = "center";
    header.style.marginBottom = "10px";
    header.style.fontWeight = "bold";
    header.style.fontSize = "1.2em";

    // 上一个月按钮
    const prevBtn = document.createElement("button");
    prevBtn.innerText = "◀ 上一月";
    prevBtn.className = "mod-cta"; // 使用 Obsidian 原生按钮样式
    prevBtn.onclick = () => {
        if (month === 0) { renderCalendar(year - 1, 11); } 
        else { renderCalendar(year, month - 1); }
    };

    // 标题文本
    const title = document.createElement("span");
    title.innerText = `${year}年 ${month + 1}月`;

    // 下一个月按钮
    const nextBtn = document.createElement("button");
    nextBtn.innerText = "下一月 ▶";
    nextBtn.className = "mod-cta";
    nextBtn.onclick = () => {
        if (month === 11) { renderCalendar(year + 1, 0); } 
        else { renderCalendar(year, month + 1); }
    };

    // 组装头部
    header.appendChild(prevBtn);
    header.appendChild(title);
    header.appendChild(nextBtn);
    container.appendChild(header);

    // 5. 构建日历表格 HTML string
    let html = `<table style="width:100%; table-layout: fixed; border-collapse: collapse;">`;
    html += `<thead><tr>`;
    ["日", "一", "二", "三", "四", "五", "六"].forEach(d => {
        html += `<th style="width:14%; border:1px solid var(--background-modifier-border); padding:5px; text-align:center; opacity:0.7">${d}</th>`;
    });
    html += `</tr></thead><tbody><tr>`;

    // 填充月初空白
    for (let i = 0; i < firstDayOfWeek; i++) {
        html += `<td style="border:1px solid var(--background-modifier-border); background: var(--background-secondary);"></td>`;
    }

    // 填充日期
    let currentDay = 1;
    let weekDay = firstDayOfWeek;

    while (currentDay <= numDays) {
        if (weekDay > 6) {
            weekDay = 0;
            html += `</tr><tr>`;
        }

        // 查找当天的笔记
        const todayNotes = currentNotes.filter(p => {
            const pDate = new Date(p[dateField]);
            return pDate.getDate() === currentDay;
        });

        // 生成卡片内容
        let cellContent = "";
        todayNotes.forEach(p => {
            cellContent += `<div style="margin-bottom: 3px; font-size:12px; line-height: 1.2; background: var(--interactive-accent); color: var(--text-on-accent); padding: 3px 5px; border-radius: 4px; overflow: hidden; white-space: nowrap; text-overflow: ellipsis;">
                ${renderContent(p)}
            </div>`;
        });

        // 今天的日期高亮
        const isToday = new Date().toDateString() === new Date(year, month, currentDay).toDateString();
        const dayStyle = isToday ? "color: var(--interactive-accent); font-weight:900; font-size:1.1em;" : "color:var(--text-muted)";

        html += `<td style="vertical-align: top; height: 100px; border:1px solid var(--background-modifier-border); padding: 4px;">
            <div style="text-align:right; margin-bottom:5px; ${dayStyle}">${currentDay}</div>
            ${cellContent}
        </td>`;

        currentDay++;
        weekDay++;
    }

    // 补全月末空白
    while (weekDay <= 6) {
        html += `<td style="border:1px solid var(--background-modifier-border); background: var(--background-secondary);"></td>`;
        weekDay++;
    }

    html += `</tr></tbody></table>`;

    // 将表格 HTML 插入到容器
    const tableDiv = document.createElement("div");
    tableDiv.innerHTML = html;
    container.appendChild(tableDiv);
}

// === 启动第一次渲染 ===
renderCalendar(currentYear, currentMonth);