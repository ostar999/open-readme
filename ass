// ==UserScript==
// @name         考题采集助手 v6.0 改进版
// @namespace    ahu-med-pro60
// @version      6.0
// @match        https://waph.ahuyk.com/*
// @grant        GM_getValue
// @grant        GM_setValue
// @grant        GM_download
// ==/UserScript==

(function () {
  "use strict";

  const STORE = "AHU_V60";
  const AUTO = "AHU_AUTO60";

  const TAGS = ["none", "wrong", "key"];
  const TAG_LABELS = { none: "未标记", wrong: "错题", key: "重点" };

  function normalizeQuestion(question) {
    const tag = TAGS.includes(question.tag)
      ? question.tag
      : question.marked
        ? "key"
        : "none";
    return { ...question, tag, marked: tag === "key" };
  }

  function tagLabel(tag) {
    return TAG_LABELS[tag] || TAG_LABELS.none;
  }

  let DB = (GM_getValue(STORE, []) || []).map(normalizeQuestion);
  let auto = GM_getValue(AUTO, false);
  let lastQuestionId = null;
  let currentAnalysis = "";

  const $ = (s) => document.querySelector(s);
  const $$ = (s) => [...document.querySelectorAll(s)];
  const uid = (t) => btoa(unescape(encodeURIComponent(t))).slice(0, 40);
  const normalizeText = (text) => (text || "").replace(/\s+/g, " ").trim();

  function slide() {
    return $(".swiper-slide-active");
  }

  function hasOfficialParse(planBox) {
    return [...planBox.querySelectorAll("div")].some((div) =>
      div.innerText?.includes("官方解析"),
    );
  }

  function findAnalysisContainer(planBox) {
    if (!planBox) return null;
    const titleNode = [...planBox.querySelectorAll("div")].find((div) =>
      div.innerText?.includes("官方解析"),
    );
    if (titleNode?.nextElementSibling) return titleNode.nextElementSibling;
    const candidates = [...planBox.querySelectorAll("div")];
    return (
      candidates.find((div) => div.querySelector("img")) ||
      candidates.find((div) => {
        const text = normalizeText(div.innerText);
        return text && !text.includes("官方解析");
      }) ||
      null
    );
  }

  function extractAnalysisContent(container) {
    if (!container) return "";
    const parts = [];
    const walk = (node) => {
      if (node.nodeType === Node.TEXT_NODE) {
        const text = normalizeText(node.textContent);
        if (text) parts.push(text);
        return;
      }
      if (node.nodeType !== Node.ELEMENT_NODE) return;
      const el = node;
      if (el.tagName === "IMG") {
        const src =
          normalizeText(el.getAttribute("src")) ||
          normalizeText(el.getAttribute("data-src")) ||
          normalizeText(el.getAttribute("data-original"));
        if (src) parts.push(src);
        return;
      }
      Array.from(el.childNodes).forEach(walk);
    };
    walk(container);
    return parts.filter(Boolean).join("\n");
  }

  /* ==== 监听题目变化 ==== */
  new MutationObserver(() => {
    const currentSlide = slide();
    if (!currentSlide) return;

    const questionText = currentSlide
      .querySelector(".que_text_box")
      ?.innerText.trim();
    if (questionText) {
      const questionId = uid(questionText);
      // 如果题目发生变化，重置解析状态
      if (lastQuestionId !== questionId) {
        lastQuestionId = questionId;
        currentAnalysis = "";
        console.log("检测到题目变化:", questionText.substring(0, 30) + "...");
      }
    }
  }).observe(document.body, { childList: true, subtree: true });

  /* ==== 监听解析内容变化 ==== */
  new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
      // 检查是否有新的子节点被添加
      mutation.addedNodes.forEach((node) => {
        if (node.nodeType === Node.ELEMENT_NODE) {
          // 检查是否是解析内容容器
          const planBox = node.classList?.contains("que_plan_box")
            ? node
            : node.querySelector?.(".que_plan_box");

          if (planBox && planBox.style.display !== "none") {
            if (hasOfficialParse(planBox)) {
              const container = findAnalysisContainer(planBox);
              const analysisContent = extractAnalysisContent(container);
              if (analysisContent && analysisContent !== currentAnalysis) {
                currentAnalysis = analysisContent;
                console.log(
                  "检测到新的解析内容:",
                  analysisContent.substring(0, 50) + "...",
                );
              }
            }
          }
        }
      });

      // 检查现有元素的内容变化
      if (mutation.type === "childList" || mutation.type === "subtree") {
        const planBoxes = $$(".que_plan_box");
        planBoxes.forEach((box) => {
          if (box.style.display !== "none") {
            if (hasOfficialParse(box)) {
              const container = findAnalysisContainer(box);
              const analysisContent = extractAnalysisContent(container);
              if (analysisContent && analysisContent !== currentAnalysis) {
                currentAnalysis = analysisContent;
                console.log(
                  "更新解析内容:",
                  analysisContent.substring(0, 50) + "...",
                );
              }
            }
          }
        });
      }
    });
  }).observe(document.body, {
    childList: true,
    subtree: true,
    characterData: true,
  });

  /* ==== 获取当前解析 ==== */
  function getCurrentAnalysis() {
    const currentSlide = slide();
    if (!currentSlide) return "";

    // 直接从当前slide中查找解析内容
    const planBox = currentSlide.querySelector(".que_plan_box");
    if (!planBox || planBox.style.display === "none") return "";

    if (!hasOfficialParse(planBox)) return "";

    const container = findAnalysisContainer(planBox);
    const analysisContent = extractAnalysisContent(container);
    return analysisContent || "";
  }

  /* ==== 读取题目信息 ==== */
  function read() {
    const currentSlide = slide();
    if (!currentSlide) return null;

    const questionElem = currentSlide.querySelector(".que_text_box");
    const questionText = questionElem?.innerText.trim();
    if (!questionText) return null;

    const questionId = uid(questionText);

    // 获取最新解析内容
    const analysis = getCurrentAnalysis();
    if (!analysis) {
      console.log("暂无解析内容，等待用户点击查看解析...");
      return null;
    }

    const chapter = $(".part_name_box")?.innerText.trim() || "";
    const index = $("#swiper_index")?.innerText || "";
    const total = $("#group_num")?.innerText || "";

    // 获取选项
    const options = {};
    currentSlide.querySelectorAll(".que_opt_item").forEach((optionElem) => {
      const optionKey = optionElem
        .querySelector(".que_opt_option")
        ?.innerText.trim();
      const optionValue = optionElem
        .querySelector(".que_opt_con")
        ?.innerText.trim();
      if (optionKey && optionValue) options[optionKey] = optionValue;
    });

    // 获取答案
    let answer = "";
    const answerBox = currentSlide.querySelector(".an_box");
    if (answerBox) {
      const answerMatch = answerBox.innerText.match(/正确答案[:：]\s*(.+)/);
      if (answerMatch) answer = answerMatch[1].trim();
    }

    return {
      id: questionId,
      chapter,
      index,
      total,
      question: questionText,
      options,
      answer,
      analysis,
      tag: "none",
      marked: false,
      time: new Date().toLocaleString(),
    };
  }

  /* ==== 保存数据 ==== */
  function save() {
    GM_setValue(STORE, DB);
    const countElem = $("#collection-count");
    if (countElem) countElem.innerText = DB.length;
    renderSidePanel();
  }

  function setQuestionTag(id, tag) {
    const questionIndex = DB.findIndex((question) => question.id === id);
    if (questionIndex === -1) return;
    const nextTag = TAGS.includes(tag) ? tag : "none";
    DB[questionIndex].tag = nextTag;
    DB[questionIndex].marked = nextTag === "key";
    save();
  }

  /* ==== 手动采集 ==== */
  function collect() {
    const questionData = read();
    if (!questionData) {
      showStatus("请先点击查看解析", "#ff5555");
      return;
    }

    if (DB.some((existing) => existing.id === questionData.id)) {
      showStatus("该题已采集", "lime");
      return;
    }

    DB.unshift(normalizeQuestion(questionData));
    save();
    showStatus("采集成功", "#00e0ff");
    console.log(
      "成功采集题目:",
      questionData.question.substring(0, 30) + "...",
    );
  }

  /* ==== 自动采集 ==== */
  function autoCollect() {
    const questionData = read();
    if (
      !questionData ||
      DB.some((existing) => existing.id === questionData.id)
    ) {
      return;
    }

    DB.unshift(normalizeQuestion(questionData));
    save();
    showStatus("自动采集", "#ffa500");
    console.log(
      "自动采集题目:",
      questionData.question.substring(0, 30) + "...",
    );
  }

  /* ==== 导出CSV ==== */
  function exportCSV() {
    if (!DB.length) {
      alert("暂无采集数据");
      return;
    }

    // 收集所有选项字母
    const optionKeys = new Set();
    DB.forEach((question) => {
      Object.keys(question.options).forEach((key) => optionKeys.add(key));
    });
    const sortedOptions = [...optionKeys].sort();

    // 构建表头
    const headers = [
      "章节",
      "题号",
      "总数",
      "题目",
      ...sortedOptions.map((key) => "选项" + key),
      "答案",
      "官方解析",
      "标记",
    ];

    // 构建数据行
    const rows = DB.map((question) => [
      question.chapter,
      question.index,
      question.total,
      question.question,
      ...sortedOptions.map((key) => question.options[key] || ""),
      question.answer,
      question.analysis,
      question.tag === "none" ? "" : tagLabel(question.tag),
    ]);

    // 生成CSV内容
    const csvContent = [headers, ...rows]
      .map((row) =>
        row.map((cell) => `"${(cell || "").replace(/"/g, '""')}"`).join(","),
      )
      .join("\n");

    // 下载文件
    GM_download({
      url: URL.createObjectURL(
        new Blob(["\uFEFF" + csvContent], { type: "text/csv;charset=utf-8" }),
      ),
      name: `阿虎医考_${new Date().toISOString().slice(0, 10)}.csv`,
    });

    showStatus("导出完成", "#00ff00");
  }

  /* ==== 题库管理界面 ==== */
  function openLibraryManager() {
    const managerDiv = document.createElement("div");
    managerDiv.innerHTML = `
      <style>
        #library-manager {
          position: fixed;
          inset: 0;
          background: rgba(0, 0, 0, 0.8);
          z-index: 999999;
          display: flex;
          align-items: center;
          justify-content: center;
        }
        #manager-box {
          background: #fff;
          width: 90%;
          max-width: 800px;
          height: 85%;
          padding: 20px;
          border-radius: 8px;
          overflow: hidden;
          display: flex;
          flex-direction: column;
        }
        #manager-header {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-bottom: 15px;
          padding-bottom: 10px;
          border-bottom: 1px solid #eee;
        }
        #search-input {
          padding: 8px 12px;
          border: 1px solid #ddd;
          border-radius: 4px;
          width: 200px;
        }
        #clear-btn {
          background: #ff4444;
          color: white;
          border: none;
          padding: 8px 16px;
          border-radius: 4px;
          cursor: pointer;
        }
        #questions-list {
          flex: 1;
          overflow-y: auto;
          padding-right: 10px;
        }
        .question-item {
          border: 1px solid #eee;
          border-radius: 6px;
          padding: 15px;
          margin-bottom: 15px;
          background: #fafafa;
        }
        .question-header {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-bottom: 10px;
        }
        .question-title {
          font-weight: bold;
          color: #333;
          flex: 1;
        }
        .question-actions {
          display: flex;
          gap: 8px;
        }
        .action-btn {
          padding: 4px 12px;
          border: none;
          border-radius: 4px;
          cursor: pointer;
          font-size: 12px;
        }
        .mark-btn {
          background: #ffd700;
          color: #333;
        }
        .delete-btn {
          background: #ff6b6b;
          color: white;
        }
        .chapter-info {
          color: #666;
          font-size: 12px;
          margin-bottom: 10px;
        }
        .options-list {
          margin: 10px 0;
        }
        .option-item {
          margin: 5px 0;
        }
        .answer-line {
          margin: 10px 0;
          font-weight: bold;
          color: #28a745;
        }
        .analysis-details {
          margin-top: 10px;
        }
        .analysis-summary {
          color: #007bff;
          cursor: pointer;
          text-decoration: underline;
        }
        .analysis-content {
          margin-top: 8px;
          padding: 10px;
          background: #f8f9fa;
          border-radius: 4px;
          border-left: 3px solid #007bff;
        }
        .tag-badge {
          display: inline-block;
          margin-right: 6px;
          padding: 2px 6px;
          border-radius: 10px;
          font-size: 12px;
          background: #e9ecef;
          color: #333;
        }
        .tag-badge.tag-wrong {
          background: #ffe3e3;
          color: #c0392b;
        }
        .tag-badge.tag-key {
          background: #fff3cd;
          color: #8a6d3b;
        }
        #close-manager {
          margin-top: 15px;
          padding: 12px;
          background: #6c757d;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
          width: 100%;
        }
      </style>

      <div id="library-manager">
        <div id="manager-box">
          <div id="manager-header">
            <input type="text" id="search-input" placeholder="搜索题目...">
            <button id="clear-btn">清空全部</button>
          </div>
          <div id="questions-list"></div>
          <button id="close-manager">关闭题库</button>
        </div>
      </div>
    `;

    document.body.appendChild(managerDiv);

    function renderQuestions(searchKeyword = "") {
      const listContainer = $("#questions-list");
      const filteredQuestions = DB.filter(
        (question) =>
          !searchKeyword || question.question.includes(searchKeyword),
      );

      listContainer.innerHTML = filteredQuestions
        .map(
          (question, index) => `
        <div class="question-item">
          <div class="question-header">
            <div class="question-title">
              <span class="tag-badge tag-${question.tag}">${question.tag === "none" ? "未标记" : tagLabel(question.tag)}</span>
              [${question.index}/${question.total}] ${question.question}
            </div>
            <div class="question-actions">
              <button class="action-btn tag-btn" data-id="${question.id}" data-tag="wrong">错题</button>
              <button class="action-btn tag-btn" data-id="${question.id}" data-tag="key">重点</button>
              <button class="action-btn tag-btn" data-id="${question.id}" data-tag="none">清除</button>
              <button class="action-btn delete-btn" data-id="${question.id}">删除</button>
            </div>
          </div>
          <div class="chapter-info">${question.chapter}</div>
          <div class="options-list">
            ${Object.entries(question.options)
              .map(
                ([key, value]) =>
                  `<div class="option-item">${key}. ${value}</div>`,
              )
              .join("")}
          </div>
          <div class="answer-line">答案：${question.answer}</div>
          <div class="analysis-details">
            <div class="analysis-summary">官方解析 ▼</div>
            <div class="analysis-content" style="display: none;">${question.analysis}</div>
          </div>
        </div>
      `,
        )
        .join("");

      // 绑定事件
      $$(".tag-btn").forEach((btn) => {
        btn.onclick = () => {
          const questionId = btn.dataset.id;
          const tag = btn.dataset.tag;
          setQuestionTag(questionId, tag);
          renderQuestions($("#search-input").value);
        };
      });

      $$(".delete-btn").forEach((btn) => {
        btn.onclick = () => {
          const questionId = btn.dataset.id;
          const questionIndex = DB.findIndex((item) => item.id === questionId);
          if (questionIndex === -1) return;
          if (confirm("确定删除这道题吗？")) {
            DB.splice(questionIndex, 1);
            save();
            renderQuestions($("#search-input").value);
          }
        };
      });

      $$(".analysis-summary").forEach((summary, index) => {
        summary.onclick = () => {
          const content = summary.nextElementSibling;
          const isVisible = content.style.display !== "none";
          content.style.display = isVisible ? "none" : "block";
          summary.textContent = `官方解析 ${isVisible ? "▼" : "▲"}`;
        };
      });
    }

    renderQuestions();

    $("#search-input").oninput = (e) => renderQuestions(e.target.value);

    $("#clear-btn").onclick = () => {
      if (confirm("确定清空全部题目？此操作不可恢复！")) {
        DB = [];
        save();
        renderQuestions();
      }
    };

    $("#close-manager").onclick = () => managerDiv.remove();
  }

  /* ==== 显示状态 ==== */
  function showStatus(text, color) {
    const statusElem = $("#status-text");
    if (statusElem) {
      statusElem.innerText = text;
      statusElem.style.color = color;
    }
  }

  /* ==== 创建UI界面 ==== */
  function createUI() {
    const uiContainer = document.createElement("div");
    uiContainer.innerHTML = `
      <style>
        #ahu-helper {
          position: fixed;
          right: 20px;
          top: 30%;
          width: 280px;
          background: #2c3e50;
          color: white;
          padding: 15px;
          border-radius: 8px;
          box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
          z-index: 99999;
          font-family: Arial, sans-serif;
        }
        #ahu-helper div {
          margin: 8px 0;
        }
        #ahu-helper button {
          width: 100%;
          margin: 6px 0;
          padding: 10px;
          border: none;
          border-radius: 4px;
          background: #3498db;
          color: white;
          cursor: pointer;
          font-size: 14px;
        }
        #ahu-helper button:hover {
          background: #2980b9;
        }
        #collect-btn {
          background: #27ae60;
        }
        #collect-btn:hover {
          background: #219653;
        }
        #export-btn {
          background: #9b59b6;
        }
        #export-btn:hover {
          background: #8e44ad;
        }
        #manage-btn {
          background: #f39c12;
        }
        #manage-btn:hover {
          background: #d35400;
        }
        #auto-checkbox {
          margin-right: 8px;
        }
        #status-text {
          text-align: center;
          font-weight: bold;
          min-height: 20px;
        }
        #library-side-panel {
          position: fixed;
          left: 20px;
          top: 50%;
          transform: translateY(-50%);
          width: 260px;
          background: #ffffff;
          color: #333;
          padding: 12px;
          border-radius: 8px;
          box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
          z-index: 99998;
          font-family: Arial, sans-serif;
        }
        .panel-title {
          font-weight: bold;
          margin-bottom: 8px;
        }
        .panel-subtitle {
          font-size: 12px;
          color: #666;
          margin: 8px 0 4px;
        }
        .panel-current-text {
          font-size: 13px;
          line-height: 1.4;
          max-height: 54px;
          overflow: hidden;
        }
        .panel-current-status {
          margin-top: 4px;
          font-size: 12px;
          color: #444;
        }
        .panel-actions {
          display: flex;
          gap: 6px;
          margin-top: 6px;
          flex-wrap: wrap;
        }
        .panel-btn {
          border: none;
          border-radius: 4px;
          padding: 4px 8px;
          font-size: 12px;
          cursor: pointer;
          color: #fff;
        }
        .panel-btn.tag-wrong {
          background: #e74c3c;
        }
        .panel-btn.tag-key {
          background: #f1c40f;
          color: #333;
        }
        .panel-btn.tag-none {
          background: #6c757d;
        }
        .panel-btn:disabled {
          opacity: 0.5;
          cursor: not-allowed;
        }
        .panel-list {
          max-height: 300px;
          overflow-y: auto;
        }
        .panel-item {
          border-top: 1px solid #f1f1f1;
          padding-top: 6px;
          margin-top: 6px;
        }
        .panel-item-title {
          font-size: 12px;
          color: #333;
          line-height: 1.3;
          max-height: 32px;
          overflow: hidden;
        }
        .panel-item-tag {
          font-size: 12px;
          margin-top: 2px;
          color: #555;
        }
        .panel-empty {
          font-size: 12px;
          color: #888;
          padding: 6px 0;
        }
      </style>

      <div id="ahu-helper">
        <div>📚 <strong>考题采集助手 v6.0</strong></div>
        <div>章节：<span id="chapter-name"></span></div>
        <div>题号：<span id="question-index"></span></div>
        <div>已采集：<span id="collection-count">${DB.length}</span> 题</div>
        <div id="status-text">待命</div>
        <button id="collect-btn">🎯 采集当前题</button>
        <label>
          <input type="checkbox" id="auto-checkbox" ${auto ? "checked" : ""}>
          🤖 自动采集模式
        </label>
        <button id="manage-btn">📖 题库管理</button>
        <button id="export-btn">💾 导出CSV</button>
      </div>
      <div id="library-side-panel"></div>
    `;

    document.body.appendChild(uiContainer);

    // 绑定事件
    $("#collect-btn").onclick = collect;
    $("#auto-checkbox").onchange = (e) => {
      auto = e.target.checked;
      GM_setValue(AUTO, auto);
      showStatus(
        auto ? "自动模式开启" : "自动模式关闭",
        auto ? "#00ff00" : "#ff9900",
      );
    };
    $("#manage-btn").onclick = openLibraryManager;
    $("#export-btn").onclick = exportCSV;

    // 更新显示信息
    setInterval(() => {
      $("#chapter-name").innerText =
        $(".part_name_box")?.innerText || "未知章节";
      $("#question-index").innerText =
        ($("#swiper_index")?.innerText || "") +
        "/" +
        ($("#group_num")?.innerText || "");
      renderSidePanel();
    }, 500);
  }

  function renderSidePanel() {
    const panel = $("#library-side-panel");
    if (!panel) return;
    const currentSlide = slide();
    const questionText =
      currentSlide?.querySelector(".que_text_box")?.innerText.trim() || "";
    const currentId = questionText ? uid(questionText) : "";
    const currentItem = currentId
      ? DB.find((question) => question.id === currentId)
      : null;
    const latestQuestions = DB.slice(0, 5);
    const currentStatus = currentItem ? tagLabel(currentItem.tag) : "未采集";
    const disabledAttr = currentItem ? "" : "disabled";
    panel.innerHTML = `
      <div class="panel-title">题库快捷标记</div>
      <div class="panel-subtitle">当前题</div>
      <div class="panel-current-text">${questionText || "未检测到题目"}</div>
      <div class="panel-current-status">状态：${currentStatus}</div>
      <div class="panel-actions">
        <button class="panel-btn tag-wrong" data-id="${currentId}" data-tag="wrong" ${disabledAttr}>错题</button>
        <button class="panel-btn tag-key" data-id="${currentId}" data-tag="key" ${disabledAttr}>重点</button>
        <button class="panel-btn tag-none" data-id="${currentId}" data-tag="none" ${disabledAttr}>清除</button>
      </div>
      <div class="panel-subtitle">最近采集</div>
      <div class="panel-list">
        ${
          latestQuestions.length
            ? latestQuestions
                .map(
                  (question) => `
            <div class="panel-item">
              <div class="panel-item-title">[${question.index}/${question.total}] ${question.question}</div>
              <div class="panel-item-tag">标记：${question.tag === "none" ? "无" : tagLabel(question.tag)}</div>
              <div class="panel-actions">
                <button class="panel-btn tag-wrong" data-id="${question.id}" data-tag="wrong">错题</button>
                <button class="panel-btn tag-key" data-id="${question.id}" data-tag="key">重点</button>
                <button class="panel-btn tag-none" data-id="${question.id}" data-tag="none">清除</button>
              </div>
            </div>
          `,
                )
                .join("")
            : `<div class="panel-empty">暂无采集记录</div>`
        }
      </div>
    `;

    panel.querySelectorAll(".panel-btn").forEach((btn) => {
      btn.onclick = () => {
        const questionId = btn.dataset.id;
        const tag = btn.dataset.tag;
        if (!questionId) return;
        setQuestionTag(questionId, tag);
      };
    });
  }

  /* ==== 初始化 ==== */
  createUI();

  // 自动采集监听器
  if (auto) {
    new MutationObserver(() => {
      setTimeout(autoCollect, 300);
    }).observe(document.body, { childList: true, subtree: true });
  }

  console.log("阿虎医考采集助手 v6.0 已启动");
})();
