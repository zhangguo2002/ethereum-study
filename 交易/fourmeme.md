# FourMeme 买卖交易实现

## 合约和函数

FourMeme Router：

```go
const FourMemeRouter = "0x5c952063c7fc8610FFDB798152D69F0B9550762b"
```

买入函数：

```solidity
buyTokenAMAP(address token, address to, uint256 funds, uint256 minAmount) payable
```

卖出函数：

```solidity
sellToken(address token, uint256 amount, uint256 minFunds)
```

卖出估价辅助合约：

```go
const FourMemeHelper3 = "0xF251F83e40a78868FcfA3FA4599Dad6494E46034"
```

估价函数：

```solidity
trySell(address token, uint256 amount)
returns (address tokenManager, address quote, uint256 funds, uint256 fee)
```

## 买入步骤

1. 连接 BSC RPC，加载 BSC 私钥，得到 `from` 地址。
2. 解析 `amountIn`，单位是 wei，也就是买入时投入的 BNB 数量。
3. 解析目标 token 地址 `tokenOut`。
4. 解析 FourMeme Router ABI。
5. 计算 `minAmount`。项目当前用 `amountIn * (1 - slippage)` 作为最小 token 输出，这只是项目里的写法；真实情况下更推荐用 FourMeme 报价接口或合约返回值计算 token 最小输出。
6. ABI 打包 `buyTokenAMAP(tokenOut, from, amountIn, minAmount)`。
7. 查询 `nonce`、`gasPrice`、`chainID`，用 `Value=amountIn` 估算 gas。
8. 构造 EVM legacy transaction：`to=FourMemeRouter`，`value=amountIn`，`data=buyTokenAMAP(...)`。
9. EIP155 签名，发送交易，等待 `WaitMined` 回执。

## 买入模板代码

```go
package trade

import (
	"context"
	"crypto/ecdsa"
	"math/big"
	"strconv"
	"strings"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

const fourMemeRouterABI = `[
  {
    "inputs":[
      {"internalType":"address","name":"token","type":"address"},
      {"internalType":"address","name":"to","type":"address"},
      {"internalType":"uint256","name":"funds","type":"uint256"},
      {"internalType":"uint256","name":"minAmount","type":"uint256"}
    ],
    "name":"buyTokenAMAP",
    "outputs":[],
    "stateMutability":"payable",
    "type":"function"
  }
]`

func BuyFourMeme(
	ctx context.Context,
	client *ethclient.Client,
	privateKeyHex string,
	tokenOutHex string,
	amountInWei *big.Int,
	slippagePercent string,
) (string, error) {
	router := common.HexToAddress("0x5c952063c7fc8610FFDB798152D69F0B9550762b")

	privateKey, err := crypto.HexToECDSA(privateKeyHex)
	if err != nil {
		return "", err
	}
	from := crypto.PubkeyToAddress(*privateKey.Public().(*ecdsa.PublicKey))
	tokenOut := common.HexToAddress(tokenOutHex)

	routerABI, err := abi.JSON(strings.NewReader(fourMemeRouterABI))
	if err != nil {
		return "", err
	}

	slip, err := strconv.ParseFloat(slippagePercent, 64)
	if err != nil {
		return "", err
	}
	slipFactor := big.NewInt(10000 - int64(slip*100))
	minAmount := new(big.Int).Mul(amountInWei, slipFactor)
	minAmount.Div(minAmount, big.NewInt(10000))

	data, err := routerABI.Pack("buyTokenAMAP", tokenOut, from, amountInWei, minAmount)
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
		From:  from,
		To:    &router,
		Value: amountInWei,
		Data:  data,
	})
	if err != nil {
		gasLimit = 300000
	}

	chainID, err := client.NetworkID(ctx)
	if err != nil {
		return "", err
	}
	tx := types.NewTransaction(nonce, router, amountInWei, gasLimit, gasPrice, data)
	signed, err := types.SignTx(tx, types.NewEIP155Signer(chainID), privateKey)
	if err != nil {
		return "", err
	}
	if err := client.SendTransaction(ctx, signed); err != nil {
		return "", err
	}

	receipt, err := bind.WaitMined(ctx, client, signed)
	if err != nil {
		return "", err
	}
	if receipt.Status != types.ReceiptStatusSuccessful {
		return signed.Hash().Hex(), ethereum.NotFound
	}
	return signed.Hash().Hex(), nil
}
```

## 卖出步骤

1. 连接 BSC RPC，加载私钥，得到 `from`。
2. 读取目标 token 的 `balanceOf(from)`。
3. 按卖出百分比计算 `amountIn = balance * percent / 100`。
4. 对 FourMeme Router 执行 ERC20 `approve(router, amountIn)`，等待授权交易确认。
5. 调用 Helper3 的 `trySell(token, amountIn)`，取返回的 `funds` 作为预计可得到的 BNB 数量。
6. 根据滑点计算 `minFunds = funds * (1 - slippage)`。
7. ABI 打包 `sellToken(token, amountIn, minFunds)`。
8. 查询 `nonce`、`gasPrice`、`chainID`，估算 gas。
9. 构造交易：`to=FourMemeRouter`，`value=0`，`data=sellToken(...)`。
10. 签名发送，等待回执。

## 卖出模板代码

```go
const erc20ABI = `[
  {"constant":true,"inputs":[{"name":"owner","type":"address"}],"name":"balanceOf","outputs":[{"name":"","type":"uint256"}],"type":"function"},
  {"constant":false,"inputs":[{"name":"spender","type":"address"},{"name":"value","type":"uint256"}],"name":"approve","outputs":[{"name":"","type":"bool"}],"type":"function"}
]`

const fourMemeSellABI = `[
  {
    "inputs":[
      {"internalType":"address","name":"token","type":"address"},
      {"internalType":"uint256","name":"amount","type":"uint256"},
      {"internalType":"uint256","name":"minFunds","type":"uint256"}
    ],
    "name":"sellToken",
    "outputs":[],
    "stateMutability":"nonpayable",
    "type":"function"
  }
]`

const fourMemeTrySellABI = `[
  {
    "inputs":[
      {"internalType":"address","name":"token","type":"address"},
      {"internalType":"uint256","name":"amount","type":"uint256"}
    ],
    "name":"trySell",
    "outputs":[
      {"internalType":"address","name":"tokenManager","type":"address"},
      {"internalType":"address","name":"quote","type":"address"},
      {"internalType":"uint256","name":"funds","type":"uint256"},
      {"internalType":"uint256","name":"fee","type":"uint256"}
    ],
    "stateMutability":"view",
    "type":"function"
  }
]`

func SellFourMeme(
	ctx context.Context,
	client *ethclient.Client,
	privateKeyHex string,
	tokenHex string,
	percent float64,
	slippagePercent string,
) (string, error) {
	router := common.HexToAddress("0x5c952063c7fc8610FFDB798152D69F0B9550762b")
	helper := common.HexToAddress("0xF251F83e40a78868FcfA3FA4599Dad6494E46034")

	privateKey, err := crypto.HexToECDSA(privateKeyHex)
	if err != nil {
		return "", err
	}
	from := crypto.PubkeyToAddress(*privateKey.Public().(*ecdsa.PublicKey))
	tokenAddr := common.HexToAddress(tokenHex)

	tokenABI, err := abi.JSON(strings.NewReader(erc20ABI))
	if err != nil {
		return "", err
	}
	token := bind.NewBoundContract(tokenAddr, tokenABI, client, client, client)

	var balanceOut []interface{}
	if err := token.Call(&bind.CallOpts{Context: ctx, From: from}, &balanceOut, "balanceOf", from); err != nil {
		return "", err
	}
	balance := balanceOut[0].(*big.Int)
	amountIn := new(big.Int)
	new(big.Float).Mul(new(big.Float).SetInt(balance), big.NewFloat(percent/100)).Int(amountIn)

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
	auth.Value = big.NewInt(0)
	auth.GasPrice = gasPrice
	auth.GasLimit = 100000

	approveTx, err := token.Transact(auth, "approve", router, amountIn)
	if err != nil {
		return "", err
	}
	if _, err := bind.WaitMined(ctx, client, approveTx); err != nil {
		return "", err
	}

	helperABI, err := abi.JSON(strings.NewReader(fourMemeTrySellABI))
	if err != nil {
		return "", err
	}
	helperContract := bind.NewBoundContract(helper, helperABI, client, client, client)
	var quoteOut []interface{}
	if err := helperContract.Call(&bind.CallOpts{Context: ctx, From: from}, &quoteOut, "trySell", tokenAddr, amountIn); err != nil {
		return "", err
	}
	funds := quoteOut[2].(*big.Int)

	slip, err := strconv.ParseFloat(slippagePercent, 64)
	if err != nil {
		return "", err
	}
	minFundsFloat := new(big.Float).Mul(new(big.Float).SetInt(funds), big.NewFloat(1-slip/100))
	minFunds, _ := minFundsFloat.Int(nil)

	sellABI, err := abi.JSON(strings.NewReader(fourMemeSellABI))
	if err != nil {
		return "", err
	}
	data, err := sellABI.Pack("sellToken", tokenAddr, amountIn, minFunds)
	if err != nil {
		return "", err
	}

	nonce, err := client.PendingNonceAt(ctx, from)
	if err != nil {
		return "", err
	}
	gasLimit, err := client.EstimateGas(ctx, ethereum.CallMsg{From: from, To: &router, Data: data})
	if err != nil {
		gasLimit = 300000
	}

	tx := types.NewTransaction(nonce, router, big.NewInt(0), gasLimit, gasPrice, data)
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

