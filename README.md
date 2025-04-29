# note

这里是清枫华的线上记事本
生人勿进

香港
vmess://eyJ2IjoyLCJwcyI6IjIzM2JveS10Y3AtMTg4LjExNi4yMi44OSIsImFkZCI6IjE4OC4xMTYuMjIuODkiLCJwb3J0IjoiMjQzNDAiLCJpZCI6IjY4MGIxMjlhLWMzMTAtNGIxMy05NGU5LTZkYWIzNmY4NGRmZCIsImFpZCI6IjAiLCJuZXQiOiJ0Y3AiLCJ0eXBlIjoibm9uZSIsInBhdGgiOiIifQ==

美国线路
vmess://eyJ2IjoyLCJwcyI6IjIzM2JveS10Y3AtMTA3LjE3My4zNC45NSIsImFkZCI6IjEwNy4xNzMuMzQuOTUiLCJwb3J0IjoiNDY1NDUiLCJpZCI6IjliMDdmODYyLTJkYWYtNGQ5Ni1iZjE5LTdlOGQyODkyMmQzOCIsImFpZCI6IjAiLCJuZXQiOiJ0Y3AiLCJ0eXBlIjoibm9uZSIsInBhdGgiOiIifQ==

备用美国线路
vmess://eyJ2IjoyLCJwcyI6IjIzM2JveS10Y3AtMzguMTgwLjIwNC4xNTYiLCJhZGQiOiIzOC4xODAuMjA0LjE1NiIsInBvcnQiOiIxMTM5OSIsImlkIjoiMjgxMTY0YjItMWY0OS00OWVjLTk4MjEtMjFjZjJmMjRiNjYwIiwiYWlkIjoiMCIsIm5ldCI6InRjcCIsInR5cGUiOiJub25lIiwicGF0aCI6IiJ9

订阅链接：
https://hongxingdl.love/hxyunvip?token=8342de6b4cf2c0047172c346cb9dd795

wss://chainstream.api.syndica.io/api-key/yFPkeHZwYqx1PuyS326yVYhpQGgeqJaxq5Dtu9pzLVXNN1pF8BrDSp7bPycHAUKEAPfs3PNzLU7NF4TBQt32LWdAxZKLc67fx

require('dotenv').config();
const { Connection, PublicKey, Keypair, Transaction, VersionedTransaction } = require('@solana/web3.js');
const { Token, TOKEN_PROGRAM_ID } = require('@solana/spl-token');
const axios = require('axios');
const { default: BigNumber } = require('bignumber.js');

// ################## 配置部分 ##################
const CONFIG = {
  RPC_URL: process.env.RPC_URL || 'https://api.mainnet-beta.solana.com',
  WALLET_SECRET: JSON.parse(process.env.WALLET_SECRET), // 使用Uint8Array格式的私钥
  CHECK_INTERVAL: 5000, // 钱包检查间隔
  SLIPPAGE: 1, // 滑点百分比
  JUPITER_API: 'https://quote-api.jup.ag/v6',
  MIN_SOL_BALANCE: 0.01 // 最小SOL余额要求
};

// ################## 初始化部分 ##################
const connection = new Connection(CONFIG.RPC_URL);
const walletKeypair = Keypair.fromSecretKey(new Uint8Array(CONFIG.WALLET_SECRET));
const walletPublicKey = walletKeypair.publicKey.toString();

// ################## 核心功能 ##################
class AutoSeller {
  constructor() {
    this.processedTokens = new Set();
    this.init();
  }

  async init() {
    console.log(`🚀 启动SOL自动卖出机器人`);
    console.log(`👛 监控钱包地址: ${walletPublicKey}`);
    console.log(`🔄 检查间隔: ${CONFIG.CHECK_INTERVAL/1000}秒`);
    this.startMonitoring();
  }

  async startMonitoring() {
    setInterval(async () => {
      try {
        await this.checkWalletBalance();
        const tokens = await this.getWalletTokens();
        await this.processNewTokens(tokens);
      } catch (error) {
        console.error('监控循环出错:', error.message);
      }
    }, CONFIG.CHECK_INTERVAL);
  }

  async checkWalletBalance() {
    const balance = await connection.getBalance(walletKeypair.publicKey);
    if (balance < CONFIG.MIN_SOL_BALANCE * 1e9) {
      throw new Error(`SOL余额不足，需要至少${CONFIG.MIN_SOL_BALANCE} SOL`);
    }
  }

  async getWalletTokens() {
    try {
      const { value } = await connection.getParsedTokenAccountsByOwner(
        walletKeypair.publicKey,
        { programId: TOKEN_PROGRAM_ID }
      );
      
      return value
        .filter(t => parseInt(t.account.data.parsed.info.tokenAmount.amount) > 0)
        .map(t => ({
          mint: t.account.data.parsed.info.mint,
          amount: t.account.data.parsed.info.tokenAmount.amount,
          decimals: t.account.data.parsed.info.tokenAmount.decimals
        }));
    } catch (error) {
      console.error('获取钱包代币失败:', error);
      return [];
    }
  }

  async processNewTokens(tokens) {
    for (const token of tokens) {
      if (!this.processedTokens.has(token.mint)) {
        console.log(`🆕 检测到新代币: ${token.mint}`);
        this.processedTokens.add(token.mint);
        setTimeout(() => this.sellToken(token), 5000);
      }
    }
  }

  async sellToken(token) {
    try {
      console.log(`⚡ 开始卖出 ${token.mint}`);
      
      // 步骤1: 获取市场价格
      const quote = await this.getJupiterQuote(token);
      if (!quote) throw new Error('获取报价失败');
      
      // 步骤2: 构建交易
      const { swapTransaction } = await axios.post(`${CONFIG.JUPITER_API}/swap`, {
        quoteResponse: quote,
        userPublicKey: walletPublicKey,
        wrapAndUnwrapSol: true,
        dynamicComputeUnitLimit: true,
      });
      
      // 步骤3: 发送交易
      const swapTxBuf = Buffer.from(swapTransaction, 'base64');
      const transaction = VersionedTransaction.deserialize(swapTxBuf);
      transaction.sign([walletKeypair]);
      
      const signature = await connection.sendTransaction(transaction);
      console.log(`📤 交易已发送: https://solscan.io/tx/${signature}`);
      
      // 步骤4: 确认交易
      await connection.confirmTransaction(signature);
      console.log('✅ 交易确认成功');
    } catch (error) {
      console.error(`卖出代币失败: ${error.message}`);
    }
  }

  async getJupiterQuote(token) {
    try {
      const amount = new BigNumber(token.amount)
        .dividedBy(10 ** token.decimals)
        .toFixed(9);
      
      const { data } = await axios.get(`${CONFIG.JUPITER_API}/quote`, {
        params: {
          inputMint: token.mint,
          outputMint: 'So11111111111111111111111111111111111111112', // SOL
          amount: amount,
          slippageBps: CONFIG.SLIPPAGE * 100,
          feeBps: 0,
          onlyDirectRoutes: false
        }
      });
      
      return data;
    } catch (error) {
      console.error('获取Jupiter报价失败:', error.response?.data || error.message);
      return null;
    }
  }
}

// ################## 启动机器人 ##################
new AutoSeller();

// ################## 错误处理 ##################
process.on('unhandledRejection', (reason, promise) => {
  console.error('未处理的Promise拒绝:', reason);
});

process.on('uncaughtException', (error) => {
  console.error('未捕获的异常:', error);
  process.exit(1);
});
