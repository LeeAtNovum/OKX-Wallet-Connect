<template>
  <div class="container">
    <h1>OKX Wallet Connect</h1>
    
    <button v-if="!walletConnected" @click="connectWallet" class="connect-button">连接钱包</button>
    <button v-else @click="disconnectWallet" class="disconnect-button">断开连接</button>
    
    <div v-if="walletConnected" class="wallet-info">
      <h2>钱包信息</h2>
      <p><strong>地址:</strong> {{ walletAddress }}</p>
      <p><strong>余额:</strong> {{ walletBalance }} {{ networkCurrency }}</p>
      <p><strong>当前网络:</strong> {{ networkName }}</p>
    </div>
    
    <div v-if="showNetworkWarning" class="network-warning">
      <h2>网络提示</h2>
      <p>请切换到 BNB Smart Chain 网络</p>
      <button @click="switchToBNBChain" class="switch-network-button">切换到 BNB Smart Chain</button>
    </div>
    
    <div v-if="showInstallGuide" class="install-guide">
      <h2>未检测到 OKX Wallet</h2>
      <p>请先安装 OKX Wallet 浏览器扩展</p>
      <a href="https://www.okx.com/web3" target="_blank" class="install-link">
        点击这里安装 OKX Wallet
      </a>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, toRaw } from 'vue';
import { ethers } from 'ethers';

// 响应式状态
const walletConnected = ref(false);
const walletAddress = ref('');
const walletBalance = ref('0');
const showInstallGuide = ref(false);
const provider = ref(null);
const networkId = ref('');
const networkName = ref('');
const networkCurrency = ref('BNB');
const showNetworkWarning = ref(false);

// 生命周期钩子
onMounted(() => {
  // 检查是否已经连接
  checkConnection();
  
  // 添加事件监听器
  if (window.okxwallet) {
    window.okxwallet.on('accountsChanged', handleAccountsChanged);
    window.okxwallet.on('chainChanged', handleChainChanged);
  }
});

onBeforeUnmount(() => {
  // 移除事件监听器
  if (window.okxwallet) {
    window.okxwallet.removeListener('accountsChanged', handleAccountsChanged);
    window.okxwallet.removeListener('chainChanged', handleChainChanged);
  }
});

// 方法
async function checkConnection() {
  if (window.okxwallet) {
    try {
      const accounts = await window.okxwallet.request({ method: 'eth_accounts' });
      if (accounts.length > 0) {
        walletAddress.value = accounts[0];
        walletConnected.value = true;
        provider.value = new ethers.providers.Web3Provider(window.okxwallet);
        await updateWalletInfo();
      }
    } catch (error) {
      console.error('检查连接状态失败:', error);
    }
  }
}

async function connectWallet() {
  // 检查是否安装了OKX Wallet
  if (window.okxwallet) {
    try {
      // 请求用户连接钱包
      const accounts = await window.okxwallet.request({ method: 'eth_requestAccounts' });
      
      if (accounts.length > 0) {
        // 获取钱包地址
        walletAddress.value = accounts[0];
        walletConnected.value = true;
        
        // 创建provider
        provider.value = new ethers.providers.Web3Provider(window.okxwallet);
        
        // 更新钱包信息
        await updateWalletInfo();
        
        // 检查并切换到BNB网络
        await checkAndSwitchNetwork();
      }
    } catch (error) {
      console.error('连接钱包失败:', error);
      alert('连接钱包失败: ' + error.message);
    }
  } else {
    // 如果没有安装OKX Wallet，显示安装指南
    showInstallGuide.value = true;
  }
}

function disconnectWallet() {
  walletConnected.value = false;
  walletAddress.value = '';
  walletBalance.value = '0';
  networkName.value = '';
  provider.value = null;
}

async function updateWalletInfo() {
  provider.value = new ethers.providers.Web3Provider(window.okxwallet);
  if (!provider.value || !walletConnected.value) return;
  
  try {
    // 获取钱包余额
    const balance = await toRaw(provider.value).getBalance(walletAddress.value);
    walletBalance.value = parseFloat(ethers.utils.formatEther(balance)).toFixed(4);
    
    // 获取网络信息
    const network = await toRaw(provider.value).getNetwork();
    console.log('当前网络:', network);
    networkId.value = network.chainId;
    
    // 设置网络名称和货币
    setNetworkInfo(network.chainId);
  } catch (error) {
    console.error('更新钱包信息失败:', error);
  }
}

function setNetworkInfo(chainId) {
  switch (chainId) {
    case 56:
      networkName.value = 'BNB Smart Chain';
      networkCurrency.value = 'BNB';
      showNetworkWarning.value = false;
      break;
    case 1:
      networkName.value = 'Ethereum Mainnet';
      networkCurrency.value = 'ETH';
      showNetworkWarning.value = true;
      break;
    case 137:
      networkName.value = 'Polygon';
      networkCurrency.value = 'MATIC';
      showNetworkWarning.value = true;
      break;
    default:
      networkName.value = `Chain ID: ${chainId}`;
      networkCurrency.value = 'ETH';
      showNetworkWarning.value = true;
  }
}

async function checkAndSwitchNetwork() {
  if (!provider.value) return;
  
  try {
    const network = await toRaw(provider.value).getNetwork();
    if (network.chainId !== 56) {
      await switchToBNBChain();
    }
  } catch (error) {
    console.error('检查网络失败:', error);
  }
}

async function switchToBNBChain() {
  if (!window.okxwallet) return;
  
  try {
    // 尝试切换到BNB Smart Chain
    await window.okxwallet.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: '0x38' }], // 56 in hex
    });
  } catch (switchError) {
    // 如果网络不存在，则添加网络
    if (switchError.code === 4902) {
      try {
        await window.okxwallet.request({
          method: 'wallet_addEthereumChain',
          params: [
            {
              chainId: '0x38', // 56 in hex
              chainName: 'BNB Smart Chain',
              nativeCurrency: {
                name: 'BNB',
                symbol: 'BNB',
                decimals: 18,
              },
              rpcUrls: ['https://bsc-dataseed.binance.org/'],
              blockExplorerUrls: ['https://bscscan.com/'],
            },
          ],
        });
      } catch (addError) {
        console.error('添加网络失败:', addError);
      }
    } else {
      console.error('切换网络失败:', switchError);
    }
  }
}

function handleAccountsChanged(accounts) {
  if (accounts.length === 0) {
    // 用户断开了钱包
    disconnectWallet();
  } else if (accounts[0] !== walletAddress.value) {
    // 用户切换了账户
    walletAddress.value = accounts[0];
    updateWalletInfo();
  }
}

function handleChainChanged(chainId) {
  // 用户切换了网络，需要刷新页面状态
  const parsedChainId = parseInt(chainId, 16);
  setNetworkInfo(parsedChainId);
  updateWalletInfo();
}
</script>

<style>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  color: #333;
  text-align: center;
  margin-bottom: 30px;
}

.connect-button, .disconnect-button, .switch-network-button {
  display: block;
  width: 200px;
  margin: 20px auto;
  padding: 12px 20px;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.connect-button {
  background-color: #2c3e50;
}

.connect-button:hover {
  background-color: #1a2530;
}

.disconnect-button {
  background-color: #e74c3c;
}

.disconnect-button:hover {
  background-color: #c0392b;
}

.switch-network-button {
  background-color: #3498db;
}

.switch-network-button:hover {
  background-color: #2980b9;
}

.wallet-info {
  margin-top: 30px;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background-color: #f9f9f9;
}

.wallet-info h2 {
  margin-top: 0;
  color: #2c3e50;
}

.network-warning {
  margin-top: 30px;
  padding: 20px;
  border: 1px solid #3498db;
  border-radius: 8px;
  background-color: #ebf5fb;
  text-align: center;
}

.network-warning h2 {
  margin-top: 0;
  color: #2980b9;
}

.install-guide {
  margin-top: 30px;
  padding: 20px;
  border: 1px solid #f0ad4e;
  border-radius: 8px;
  background-color: #fcf8e3;
  text-align: center;
}

.install-guide h2 {
  margin-top: 0;
  color: #8a6d3b;
}

.install-link {
  display: inline-block;
  margin-top: 15px;
  padding: 10px 20px;
  background-color: #f0ad4e;
  color: white;
  text-decoration: none;
  border-radius: 4px;
  font-weight: bold;
  transition: background-color 0.3s;
}

.install-link:hover {
  background-color: #ec971f;
}
</style>