```dataviewjs
// ================= 配置区域 =================
const source = '"/"';  // 你的项目文件夹
const dateField = "date";      // 日期属性名

// 📌 这里填你刚才存放 js 和 css 的路径 (相对于库根目录)
const localJsPath = "Scripts/vis-timeline-graph2d.min.js";
const localCssPath = "Scripts/vis-timeline-graph2d.min.css";

// ================= 核心代码 =================

const container = dv.el("div", "⏳ 正在加载本地组件...", { attr: { style: "height: 400px; border: 1px solid var(--background-modifier-border);" } });

// 获取本地文件的真实系统路径 (app://...)
const getLocalPath = (path) => {
    return app.vault.adapter.getResourcePath(path);
};

// 动态加载函数
const loadScript = (url) => {
    return new Promise((resolve, reject) => {
        if (window.vis) { resolve(); return; } 
        const script = document.createElement('script');
        script.src = url;
        script.onload = resolve;
        script.onerror = () => reject(new Error(`无法加载JS: ${url}`));
        document.head.appendChild(script);
    });
};

const loadCSS = (url) => {
    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = url;
    document.head.appendChild(link);
};

try {
    // 1. 获取 Obsidian 内部生成的资源链接
    const realJsUrl = getLocalPath(localJsPath);
    const realCssUrl = getLocalPath(localCssPath);

    // 2. 加载本地资源
    await Promise.all([
        loadScript(realJsUrl),
        loadCSS(realCssUrl)
    ]);

    // 3. 准备数据
    const pages = dv.pages(source).where(p => p[dateField]);
    
    if (pages.length === 0) {
        container.innerText = "❌ 未找到数据，请检查文件夹内是否有含日期的笔记。";
    } else {
        const items = new vis.DataSet(
            pages.map(p => {
                const d = new Date(p[dateField]);
                if (isNaN(d.getTime())) return null;

                return {
                    id: p.file.path,
                    content: `<span style="font-size:12px">${p.file.name}</span>`, 
                    start: d,
                    title: "点击打开: " + p.file.name,
                    style: "border-color: var(--interactive-accent); background-color: var(--background-primary); color: var(--text-normal); border-radius: 4px;"
                };
            }).filter(i => i !== null).values
        );

        container.innerText = ""; 

        const options = {
            height: '100%',
            start: new Date(new Date().getTime() - 1000 * 60 * 60 * 24 * 7), 
            end: new Date(new Date().getTime() + 1000 * 60 * 60 * 24 * 7),
            zoomKey: "ctrlKey", 
            horizontalScroll: true,
            moveable: true
        };

        const timeline = new vis.Timeline(container, items, options);

        timeline.on('select', function (properties) {
            if (properties.items.length > 0) {
                app.workspace.openLinkText(properties.items[0], "", false);
            }
        });
    }

} catch (error) {
    container.innerHTML = `
        <div style="color: red;">⚠️ 加载失败</div>
        <div>${error.message}</div>
        <br>
        <div style="color: gray; font-size: 0.8em;">
        请检查：<br>
        1. Scripts 文件夹里是否有 vis-timeline-graph2d.min.js 文件？<br>
        2. 代码顶部的 localJsPath 路径写对了吗？
        </div>
    `;
}