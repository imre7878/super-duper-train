<!DOCTYPE html>
<html lang="hu">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Crypto Whale Tracker Pro</title>
    <style>
        body {
            font-family: 'Segoe UI', Arial, sans-serif;
            background-color: #0b0e11;
            color: #eaecef;
            margin: 0;
            padding: 40px 20px;
            display: flex;
            justify-content: center;
        }
        .container {
            max-width: 850px;
            width: 100%;
            background: #181a20;
            padding: 35px;
            border-radius: 16px;
            box-shadow: 0 8px 32px rgba(0,0,0,0.5);
            border: 1px solid #2b2f36;
        }
        h1 { color: #f3ba2f; margin-top: 0; font-size: 28px; }
        p { color: #848e9c; }
        .search-box {
            display: flex;
            gap: 12px;
            margin: 30px 0;
        }
        input {
            flex: 1;
            padding: 14px;
            border-radius: 8px;
            border: 1px solid #474d57;
            background: #2b2f36;
            color: white;
            font-size: 16px;
        }
        input:focus { border-color: #f3ba2f; outline: none; }
        button {
            background: #f3ba2f;
            color: #0b0e11;
            border: none;
            padding: 14px 28px;
            border-radius: 8px;
            font-weight: bold;
            cursor: pointer;
            font-size: 16px;
            transition: 0.2s;
        }
        button:hover { background: #ffd256; }
        .loader { display: none; color: #f3ba2f; font-weight: bold; margin: 20px 0; text-align: center; }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
            display: none;
        }
        th, td { padding: 14px; text-align: left; border-bottom: 1px solid #2b2f36; }
        th { color: #848e9c; font-size: 14px; }
        .symbol { font-weight: bold; color: #fff; }
        .badge { background: #2b2f36; padding: 4px 8px; border-radius: 4px; font-size: 12px; color: #848e9c; }
        .price-up { color: #02c076; font-weight: bold; }
    </style>
</head>
<body>

<div class="container">
    <h1>📈 Whale Wallet Pro Tracker</h1>
    <p>Figyeld a legnagyobb BNB hálózati tárcák egyenlegeit és a tokenek élő értékét valós időben.</p>
    
    <div class="search-box">
        <input type="text" id="walletInput" value="0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045" placeholder="Írj be egy BSC tárcacímet (0x...)">
        <button onclick="trackWallet()">Keresés</button>
    </div>

    <div class="loader" id="loader">Adatok lekérése a blokkláncról és árak számítása...</div>

    <table id="resultTable">
        <thead>
            <tr>
                <th>Token</th>
                <th>Szerződés</th>
                <th>Egyenleg</th>
                <th>Dollár Érték ($)</th>
            </tr>
        </thead>
        <tbody id="resultBody"></tbody>
    </table>
</div>

<script>
const ALCHEMY_URL = "https://alchemy.com";

async function trackWallet() {
    const address = document.getElementById('walletInput').value.trim();
    const loader = document.getElementById('loader');
    const table = document.getElementById('resultTable');
    const tbody = document.getElementById('resultBody');

    if(!address) return alert("Kérlek adj meg egy tárcacímet!");

    loader.style.display = "block";
    table.style.display = "none";
    tbody.innerHTML = "";

    try {
        const response = await fetch(ALCHEMY_URL, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                jsonrpc: "2.0",
                method: "alchemy_getTokenBalances",
                params: [address, "erc20", {"maxCount": 20}],
                id: 1
            })
        });
        const data = await response.json();
        const tokens = data.result.tokenBalances;

        let contractsToPrice = [];
        let tempTokensData = [];

        for (let t of tokens) {
            if (t.tokenBalance === "0x0000000000000000000000000000000000000000000000000000000000000000") continue;
            
            const metaRes = await fetch(ALCHEMY_URL, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    jsonrpc: "2.0",
                    method: "alchemy_getTokenMetadata",
                    params: [t.contractAddress],
                    id: 1
                })
            });
            const metaData = await metaRes.json();
            const symbol = metaData.result.symbol || "UNKNOWN";
            const decimals = metaData.result.decimals || 18;
            
            const actualBalance = Number(BigInt(t.tokenBalance)) / Math.pow(10, decimals);

            if(actualBalance > 0) {
                tempTokensData.push({
                    contract: t.contractAddress,
                    symbol: symbol,
                    balance: actualBalance,
                    usdValue: 0
                });
                contractsToPrice.push(t.contractAddress.toLowerCase());
            }
        }

        if(contractsToPrice.length > 0) {
            const geckoRes = await fetch(`https://coingecko.com{contractsToPrice.join(',')}&vs_currencies=usd`);
            const geckoData = await geckoRes.json();

            tempTokensData.forEach(token => {
                const priceInfo = geckoData[token.contract.toLowerCase()];
                if(priceInfo && priceInfo.usd) {
                    token.usdValue = token.balance * priceInfo.usd;
                }
            });
        }

        tempTokensData.sort((a,b) => b.usdValue - a.usdValue);

        tempTokensData.forEach(token => {
            const displayUsd = token.usdValue > 0 ? "$" + token.usdValue.toLocaleString('en-US', {maximumFractionDigits: 2}) : "Nincs ár adat";
            const row = `<tr>
                <td><span class="symbol">${token.symbol}</span></td>
                <td><span class="badge">${token.contract.substring(0,6)}...${token.contract.substring(36)}</span></td>
                <td>${token.balance.toLocaleString(undefined, {maximumFractionDigits: 4})}</td>
                <td class="price-up">${displayUsd}</td>
            </tr>`;
            tbody.innerHTML += row;
        });

        loader.style.display = "none";
        table.style.display = "table";

    } catch (err) {
        console.error(err);
        loader.style.display = "none";
        alert("Hiba történt az adatok betöltésekor.");
    }
}
</script>

</body>
</html>
