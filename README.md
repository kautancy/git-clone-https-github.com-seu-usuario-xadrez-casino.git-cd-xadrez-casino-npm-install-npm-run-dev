xadrez-casino-backend/
├── .env
├── index.js
├── services/
│   └── withdraw.js
├── utils/
│   └── convert.js
├── logs/
│   └── withdrawals.json
├── package.json
PRIVATE_KEY=chave_privada_da_wallet_do_cassino
RPC_URL=https://mainnet.base.org  # ou outro RPC (infura, alchemy etc)
const express = require('express');
const cors = require('cors');
const dotenv = require('dotenv');
const { handleWithdraw } = require('./services/withdraw');

dotenv.config();

const app = express();
app.use(cors());
app.use(express.json());

app.post('/withdraw', async (req, res) => {
  const { address, fichas } = req.body;
  try {
    const result = await handleWithdraw(address, fichas);
    res.status(200).json(result);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Erro ao processar saque.' });
  }
});

const PORT = 3001;
app.listen(PORT, () => {
  console.log(`🚀 Backend do Xadrez Casino rodando em http://localhost:${PORT}`);
});
const { ethers } = require('ethers');
const { convertFichasToETH } = require('../utils/convert');
const fs = require('fs-extra');
const path = require('path');

const logPath = path.join(__dirname, '../logs/withdrawals.json');

async function handleWithdraw(toAddress, fichas) {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

  // valor em ETH equivalente às fichas
  const { ethValue, ethFinal, usdTotal, taxa } = await convertFichasToETH(fichas);

  const tx = await wallet.sendTransaction({
    to: toAddress,
    value: ethers.parseEther(ethFinal.toFixed(8)),
  });

  const log = {
    para: toAddress,
    fichas,
    usdTotal,
    taxa: `${taxa}%`,
    ethEnviado: ethFinal,
    txHash: tx.hash,
    data: new Date().toISOString()
  };

  await fs.ensureFile(logPath);
  const existing = await fs.readJson(logPath).catch(() => []);
  await fs.writeJson(logPath, [...existing, log], { spaces: 2 });

  return { status: 'ok', txHash: tx.hash };
}

module.exports = { handleWithdraw };
const axios = require('axios');

async function convertFichasToETH(fichas) {
  const valorPorFichaUSD = 0.005;
  const usdTotal = fichas * valorPorFichaUSD;
  const taxa = 6;
  const usdLiquido = usdTotal * ((100 - taxa) / 100);

  // preço atual do ETH em USD
  const r = await axios.get('https://api.coinbase.com/v2/prices/ETH-USD/spot');
  const ethPrice = parseFloat(r.data.data.amount);

  const ethValue = usdTotal / ethPrice;
  const ethFinal = usdLiquido / ethPrice;

  return { ethValue, ethFinal, usdTotal, taxa };
}

module.exports = { convertFichasToETH };
final response = await http.post(
  Uri.parse('http://IP_DO_SERVIDOR:3001/withdraw'),
  headers: {'Content-Type': 'application/json'},
  body: json.encode({
    'address': '0xCarteiraDoJogador',
    'fichas': 500,
  }),
);
