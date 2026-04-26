# PancakeSwap V2 买卖交易实现

## 合约和函数

Router 地址：

```go
var PancakeV2Routers = map[string]string{
	"56": "0x10ED43C718714eb63d5aA57B78B54704E256024E", // BSC mainnet
	"97": "0xD99D1c33F9fC3444f8101754aBC46c52416550D1", // BSC testnet
}
```

WBNB 地址：

```go
const WBNB = "0xBB4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c"
```

买入使用：

```solidity
getAmountsOut(uint256 amountIn, address[] path)
swapExactETHForTokens(uint256 amountOutMin, address[] path, address to, uint256 deadline) payable
```

卖出使用：

```solidity
approve(address spender, uint256 value)
getAmountsOut(uint256 amountIn, address[] path)
swapExactTokensForETH(uint256 amountIn, uint256 amountOutMin, address[] path, address to, uint256 deadline)
```

## 买入步骤

1. 根据 `chainIndex` 获取 PancakeSwap V2 Router。
2. 连接 BSC RPC，加载私钥，得到 `from`。
3. 解析 `amountIn`，单位 wei，作为买入投入的 BNB。
4. 构造交易路径：`[WBNB, tokenOut]`。
5. 调用 `getAmountsOut(amountIn, path)`，得到预计输出 `estimatedOut`。
6. 按滑点计算 `amountOutMin = estimatedOut * (1 - slippage)`。
7. ABI 打包 `swapExactETHForTokens(amountOutMin, path, from, deadline)`。
8. 查询 `nonce`、`gasPrice`、`chainID`，用 `Value=amountIn` 估算 gas。
9. 构造交易：`to=router`，`value=amountIn`，`data=swapExactETHForTokens(...)`。
10. 签名发送，等待回执。

## 买入模板代码

```go
package trade

import (
	"context"
	"crypto/ecdsa"
	"math/big"
	"strconv"
	"strings"
	"time"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

const pancakeV2RouterABI = `[
  {"inputs":[{"type":"uint256","name":"amountIn"},{"type":"address[]","name":"path"}],"name":"getAmountsOut","outputs":[{"type":"uint256[]","name":"amounts"}],"stateMutability":"view","type":"function"},
  {"inputs":[{"type":"uint256","name":"amountOutMin"},{"type":"address[]","name":"path"},{"type":"address","name":"to"},{"type":"uint256","name":"deadline"}],"name":"swapExactETHForTokens","outputs":[{"type":"uint256[]","name":"amounts"}],"stateMutability":"payable","type":"function"}
]`

func BuyPancakeV2(
	ctx context.Context,
	client *ethclient.Client,
	privateKeyHex string,
	routerHex string,
	tokenOutHex string,
	amountInWei *big.Int,
	slippagePercent string,
) (string, error) {
	router := common.HexToAddress(routerHex)
	wbnb := common.HexToAddress("0xBB4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c")
	tokenOut := common.HexToAddress(tokenOutHex)
	path := []common.Address{wbnb, tokenOut}

	privateKey, err := crypto.HexToECDSA(privateKeyHex)
	if err != nil {
		return "", err
	}
	from := crypto.PubkeyToAddress(*privateKey.Public().(*ecdsa.PublicKey))

	routerABI, err := abi.JSON(strings.NewReader(pancakeV2RouterABI))
	if err != nil {
		return "", err
	}

	getData, err := routerABI.Pack("getAmountsOut", amountInWei, path)
	if err != nil {
		return "", err
	}
	res, err := client.CallContract(ctx, ethereum.CallMsg{To: &router, Data: getData}, nil)
	if err != nil {
		return "", err
	}
	var amountsOut []*big.Int
	if err := routerABI.UnpackIntoInterface(&amountsOut, "getAmountsOut", res); err != nil {
		return "", err
	}
	estimatedOut := amountsOut[len(amountsOut)-1]

	slip, err := strconv.ParseFloat(slippagePercent, 64)
	if err != nil {
		return "", err
	}
	minOutFloat := new(big.Float).Mul(new(big.Float).SetInt(estimatedOut), big.NewFloat(1-slip/100))
	amountOutMin, _ := minOutFloat.Int(nil)

	deadline := big.NewInt(time.Now().Add(10 * time.Minute).Unix())
	swapData, err := routerABI.Pack("swapExactETHForTokens", amountOutMin, path, from, deadline)
	if err != nil {
		return "", err
	}

	nonce, err := client.PendingNonceAt(ctx, from)
	if err != nil {
		return "", err
	}
	gasPrice, err := client.SuggestGasPrice(ctx)
	if err != nil {
		return "", err
	}
	gasLimit, err := client.EstimateGas(ctx, ethereum.CallMsg{
		From: from, To: &router, Value: amountInWei, Data: swapData,
	})
	if err != nil {
		gasLimit = 300000
	}
	chainID, err := client.NetworkID(ctx)
	if err != nil {
		return "", err
	}

	tx := types.NewTransaction(nonce, router, amountInWei, gasLimit, gasPrice, swapData)
	signed, err := types.SignTx(tx, types.NewEIP155Signer(chainID), privateKey)
	if err != nil {
		return "", err
	}
	if err := client.SendTransaction(ctx, signed); err != nil {
		return "", err
	}
	_, err = bind.WaitMined(ctx, client, signed)
	return signed.Hash().Hex(), err
}
```

## 卖出步骤

1. 根据 `chainIndex` 获取 PancakeSwap V2 Router。
2. 连接 BSC RPC，加载私钥，得到 `from`。
3. 读取目标 token 的 `balanceOf(from)`。
4. 按卖出百分比计算 `amountIn = balance * percent / 100`。
5. 对 Router 执行 token `approve(router, amountIn)`，等待授权确认。
6. 构造路径：`[tokenIn, WBNB]`。
7. 调用 `getAmountsOut(amountIn, path)`，得到预计输出 BNB。
8. 按滑点计算 `amountOutMin`。
9. ABI 打包 `swapExactTokensForETH(amountIn, amountOutMin, path, from, deadline)`。
10. 构造交易：`to=router`，`value=0`，`data=swapExactTokensForETH(...)`。
11. 签名发送，等待回执。

## 卖出模板代码

```go
const pancakeV2SellABI = `[
  {"name":"getAmountsOut","outputs":[{"type":"uint256[]","name":""}],"inputs":[{"type":"uint256","name":"amountIn"},{"type":"address[]","name":"path"}],"stateMutability":"view","type":"function"},
  {"name":"swapExactTokensForETH","outputs":[{"type":"uint256[]","name":""}],"inputs":[{"type":"uint256","name":"amountIn"},{"type":"uint256","name":"amountOutMin"},{"type":"address[]","name":"path"},{"type":"address","name":"to"},{"type":"uint256","name":"deadline"}],"stateMutability":"nonpayable","type":"function"}
]`

const erc20ForTradeABI = `[
  {"constant":true,"inputs":[{"name":"owner","type":"address"}],"name":"balanceOf","outputs":[{"name":"","type":"uint256"}],"type":"function"},
  {"constant":false,"inputs":[{"name":"spender","type":"address"},{"name":"value","type":"uint256"}],"name":"approve","outputs":[{"name":"","type":"bool"}],"type":"function"}
]`

func SellPancakeV2(
	ctx context.Context,
	client *ethclient.Client,
	privateKeyHex string,
	routerHex string,
	tokenInHex string,
	percent float64,
	slippagePercent string,
) (string, error) {
	router := common.HexToAddress(routerHex)
	tokenIn := common.HexToAddress(tokenInHex)
	wbnb := common.HexToAddress("0xBB4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c")
	path := []common.Address{tokenIn, wbnb}

	privateKey, err := crypto.HexToECDSA(privateKeyHex)
	if err != nil {
		return "", err
	}
	from := crypto.PubkeyToAddress(*privateKey.Public().(*ecdsa.PublicKey))

	tokenABI, err := abi.JSON(strings.NewReader(erc20ForTradeABI))
	if err != nil {
		return "", err
	}
	token := bind.NewBoundContract(tokenIn, tokenABI, client, client, client)

	var balanceOut []interface{}
	if err := token.Call(&bind.CallOpts{Context: ctx, From: from}, &balanceOut, "balanceOf", from); err != nil {
		return "", err
	}
	amountIn := new(big.Int)
	new(big.Float).Mul(new(big.Float).SetInt(balanceOut[0].(*big.Int)), big.NewFloat(percent/100)).Int(amountIn)

	chainID, err := client.NetworkID(ctx)
	if err != nil {
		return "", err
	}
	gasPrice, err := client.SuggestGasPrice(ctx)
	if err != nil {
		return "", err
	}
	auth, err := bind.NewKeyedTransactorWithChainID(privateKey, chainID)
	if err != nil {
		return "", err
	}
	auth.From = from
	auth.GasPrice = gasPrice
	auth.GasLimit = 100000
	auth.Value = big.NewInt(0)

	approveTx, err := token.Transact(auth, "approve", router, amountIn)
	if err != nil {
		return "", err
	}
	if _, err := bind.WaitMined(ctx, client, approveTx); err != nil {
		return "", err
	}

	routerABI, err := abi.JSON(strings.NewReader(pancakeV2SellABI))
	if err != nil {
		return "", err
	}
	getData, err := routerABI.Pack("getAmountsOut", amountIn, path)
	if err != nil {
		return "", err
	}
	res, err := client.CallContract(ctx, ethereum.CallMsg{To: &router, Data: getData}, nil)
	if err != nil {
		return "", err
	}
	var amountsOut []*big.Int
	if err := routerABI.UnpackIntoInterface(&amountsOut, "getAmountsOut", res); err != nil {
		return "", err
	}
	estimatedOut := amountsOut[len(amountsOut)-1]

	slip, err := strconv.ParseFloat(slippagePercent, 64)
	if err != nil {
		return "", err
	}
	minOutFloat := new(big.Float).Mul(new(big.Float).SetInt(estimatedOut), big.NewFloat(1-slip/100))
	amountOutMin, _ := minOutFloat.Int(nil)

	deadline := big.NewInt(time.Now().Add(10 * time.Minute).Unix())
	swapData, err := routerABI.Pack("swapExactTokensForETH", amountIn, amountOutMin, path, from, deadline)
	if err != nil {
		return "", err
	}

	nonce, err := client.PendingNonceAt(ctx, from)
	if err != nil {
		return "", err
	}
	gasLimit, err := client.EstimateGas(ctx, ethereum.CallMsg{From: from, To: &router, Data: swapData})
	if err != nil {
		gasLimit = 300000
	}

	tx := types.NewTransaction(nonce, router, big.NewInt(0), gasLimit, gasPrice, swapData)
	signed, err := types.SignTx(tx, types.NewEIP155Signer(chainID), privateKey)
	if err != nil {
		return "", err
	}
	if err := client.SendTransaction(ctx, signed); err != nil {
		return "", err
	}
	_, err = bind.WaitMined(ctx, client, signed)
	return signed.Hash().Hex(), err
}
```

