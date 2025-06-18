// paper-bot/README.js
// Version 1: Reactive Paper Trading Bot
// -------------------------------------
// This script contains all the components of the Version 1 trading bot setup
// - WebSocket-based signal server (signal-sender.js)
// - Trading bot listener (bot-listener.js)
// - Trade logic using CoinGecko API (trade.js)
// - Portfolio JSON storage

// ========== portfolio.json (create this file manually) ==========
// {
//   "cash": 1000,
//   "bitcoin": 0.05
// }

// ========== trade.js ==========
const fs = require('fs');
const axios = require('axios');

async function getPrice(symbol) {
    const res = await axios.get(`https://api.coingecko.com/api/v3/simple/price?ids=${symbol}&vs_currencies=usd`);
    return res.data[symbol].usd;
}

async function buy(symbol, usdAmount) {
    const price = await getPrice(symbol);
    const quantity = usdAmount / price;

    let portfolio = JSON.parse(fs.readFileSync('portfolio.json'));
    if (portfolio.cash < usdAmount) {
        console.log(`[BUY] Not enough cash.`);
        return;
    }

    portfolio.cash -= usdAmount;
    portfolio[symbol] = (portfolio[symbol] || 0) + quantity;

    fs.writeFileSync('portfolio.json', JSON.stringify(portfolio, null, 2));
    console.log(`[BUY] Bought ${quantity.toFixed(5)} ${symbol} @ $${price.toFixed(2)}`);
}

async function sell(symbol) {
    const portfolio = JSON.parse(fs.readFileSync('portfolio.json'));
    const quantity = portfolio[symbol] || 0;

    if (quantity <= 0) {
        console.log(`[SELL] No ${symbol} to sell.`);
        return;
    }

    const price = await getPrice(symbol);
    const proceeds = quantity * price;

    portfolio.cash += proceeds;
    portfolio[symbol] = 0;

    fs.writeFileSync('portfolio.json', JSON.stringify(portfolio, null, 2));
    console.log(`[SELL] Sold ${quantity.toFixed(5)} ${symbol} @ $${price.toFixed(2)}, earned $${proceeds.toFixed(2)}`);
}

module.exports = { buy, sell };

// ========== signal-sender.js ==========
const WebSocket = require('ws');
const server = new WebSocket.Server({ port: 8080 });
console.log('[SENDER] Signal server started on port 8080');

const clients = [];
server.on('connection', (socket) => {
    clients.push(socket);
    console.log('[SENDER] Bot connected');
});

function emitSignal() {
    const signals = ['BUY', 'SELL'];
    const signal = signals[Math.floor(Math.random() * signals.length)];
    const payload = JSON.stringify({ signal });

    console.log(`[SENDER] Broadcasting signal: ${signal}`);
    clients.forEach((client) => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(payload);
        }
    });
}

setInterval(emitSignal, 7000); // Emits every 7 seconds for testing

// ========== bot-listener.js ==========
const WebSocket = require('ws');
const { buy, sell } = require('./trade');

const ws = new WebSocket('ws://localhost:8080');
const symbol = 'bitcoin';

ws.on('open', () => {
    console.log('[BOT] Connected to signal server...');
});

ws.on('message', async (data) => {
    const message = JSON.parse(data);
    console.log(`[BOT] Received signal: ${message.signal}`);

    if (message.signal === 'BUY') {
        await buy(symbol, 100);
    } else if (message.signal === 'SELL') {
        await sell(symbol);
    }
});

// ========== How to Run ==========
// 1. Install dependencies:
//    npm install axios ws
//
// 2. Start signal sender:
//    node signal-sender.js
//
// 3. In another terminal, start the bot:
//    node bot-listener.js
//a
// It will automatically buy/sell based on received fake signals.
