<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dify 工作流助手</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; background-color: #f4f6f8; margin: 0; display: flex; justify-content: center; padding: 20px; }
        .chat-container { width: 100%; max-width: 800px; background: white; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); display: flex; flex-direction: column; height: 90vh; }
        .chat-header { padding: 20px; border-bottom: 1px solid #eee; text-align: center; font-weight: bold; font-size: 1.2em; background: #fff; border-radius: 12px 12px 0 0; }
        .chat-history { flex: 1; padding: 20px; overflow-y: auto; background-color: #fafafa; }
        .message { margin-bottom: 20px; max-width: 85%; line-height: 1.6; }
        .message.user { margin-left: auto; background: #007bff; color: white; padding: 12px 16px; border-radius: 16px 16px 0 16px; }
        .message.ai { margin-right: auto; background: white; border: 1px solid #e0e0e0; padding: 12px 16px; border-radius: 16px 16px 16px 0; color: #333; white-space: pre-wrap; }
        .input-area { padding: 20px; background: white; border-top: 1px solid #eee; border-radius: 0 0 12px 12px; }
        .file-group { margin-bottom: 10px; display: flex; align-items: center; gap: 10px; }
        .file-group label { font-size: 0.9em; font-weight: bold; color: #555; width: 120px; }
        .btn-submit { width: 100%; padding: 12px; background: #007bff; color: white; border: none; border-radius: 8px; font-size: 1em; cursor: pointer; transition: 0.3s; margin-top: 10px; }
        .btn-submit:disabled { background: #ccc; cursor: not-allowed; }
        .loading { font-style: italic; color: #888; }
    </style>
</head>
<body>

<div class="chat-container">
    <div class="chat-header">数据分析与处理 Agent</div>
    
    <div class="chat-history" id="chatHistory">
        <div class="message ai">你好！我是处理助手。请在下方上传 Cabot 文件和 Supplier 文件，我会为你生成 Final Report。</div>
    </div>

    <div class="input-area">
        <div class="file-group">
            <label>Cabot File:</label>
            <input type="file" id="cabotFile" accept=".txt,.pdf,.csv,.xlsx,.docx">
        </div>
        <div class="file-group">
            <label>Supplier File:</label>
            <input type="file" id="supplierFile" accept=".txt,.pdf,.csv,.xlsx,.docx">
        </div>
        <button id="submitBtn" class="btn-submit" onclick="handleProcess()">开始处理并生成报告</button>
    </div>
</div>

<script>
    // ⚠️ 替换成你的真实 API Key 
    const DIFY_API_KEY = 'app-JsRkEsTc0LYgpm1LLw2hyWhQ'; 
    // ⚠️ 如果是自己部署的 Dify，请改成你的域名，例如 http://你的IP/v1
    const API_BASE_URL = 'https://api.dify.ai/v1'; 

    const chatHistory = document.getElementById('chatHistory');
    const submitBtn = document.getElementById('submitBtn');

    // 添加消息到聊天面板
    function addMessage(content, sender) {
        const msgDiv = document.createElement('div');
        msgDiv.className = `message ${sender}`;
        msgDiv.textContent = content;
        chatHistory.appendChild(msgDiv);
        chatHistory.scrollTop = chatHistory.scrollHeight;
        return msgDiv;
    }

    // 步骤1：上传单个文件到 Dify
    async function uploadFileToDify(file) {
        const formData = new FormData();
        formData.append('file', file);
        formData.append('user', 'web-user'); // 标识用户身份

        const response = await fetch(`${API_BASE_URL}/files/upload`, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${DIFY_API_KEY}`
            },
            body: formData
        });

        if (!response.ok) {
            const err = await response.json();
            throw new Error(`文件上传失败: ${err.message || '未知错误'}`);
        }
        const data = await response.json();
        return data.id; // 返回文件ID
    }

    // 步骤2：执行工作流
    async function runWorkflow(cabotFileId, supplierFileId) {
        const payload = {
            inputs: {
                "cabot_file": {
                    "transfer_method": "local_file",
                    "upload_file_id": cabotFileId,
                    "type": "document" // 根据你实际接受的文件类型，可以是 document 或 image
                },
                "supplier_file": {
                    "transfer_method": "local_file",
                    "upload_file_id": supplierFileId,
                    "type": "document"
                }
            },
            response_mode: "blocking", // 阻塞模式，等待工作流全部执行完毕返回结果
            user: "web-user"
        };

        const response = await fetch(`${API_BASE_URL}/workflows/run`, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${DIFY_API_KEY}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(payload)
        });

        if (!response.ok) {
            const err = await response.json();
            throw new Error(`工作流执行失败: ${err.message || '未知错误'}`);
        }
        
        const data = await response.json();
        // 解析最终返回的数据 (你图片里的输出节点叫 final_report)
        return data.data.outputs.final_report; 
    }

    // 主处理逻辑
    async function handleProcess() {
        const cabotInput = document.getElementById('cabotFile');
        const supplierInput = document.getElementById('supplierFile');

        if (!cabotInput.files[0] || !supplierInput.files[0]) {
            alert("请必须同时上传 Cabot 文件和 Supplier 文件！");
            return;
        }

        const cabotFile = cabotInput.files[0];
        const supplierFile = supplierInput.files[0];

        // 界面更新状态
        submitBtn.disabled = true;
        submitBtn.innerText = '处理中，请稍候...';
        addMessage(`[已上传文件] \n1. ${cabotFile.name}\n2. ${supplierFile.name}`, 'user');
        const loadingMsg = addMessage('正在分析文件内容并生成报告，请稍候...', 'ai');
        loadingMsg.classList.add('loading');

        try {
            // 1. 上传文件获取 ID
            console.log("正在上传 Cabot 文件...");
            const cabotFileId = await uploadFileToDify(cabotFile);
            console.log("正在上传 Supplier 文件...");
            const supplierFileId = await uploadFileToDify(supplierFile);

            // 2. 调用工作流
            console.log("执行工作流...");
            const finalReport = await runWorkflow(cabotFileId, supplierFileId);

            // 3. 显示结果
            chatHistory.removeChild(loadingMsg);
            addMessage(finalReport, 'ai');

        } catch (error) {
            console.error(error);
            chatHistory.removeChild(loadingMsg);
            addMessage(`❌ 发生错误：${error.message}`, 'ai');
        } finally {
            // 恢复按钮状态
            submitBtn.disabled = false;
            submitBtn.innerText = '开始处理并生成报告';
            
            // 清空文件选择框（可选）
            cabotInput.value = '';
            supplierInput.value = '';
        }
    }
</script>
</body>
</html>
