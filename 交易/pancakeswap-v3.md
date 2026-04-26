# PancakeSwap V3 买卖交易实现

## 合约和函数

PancakeSwap V3 BSC 主网地址：

```go
var (
	V3Router = "0x1b81D678ffb9C0263b24A97847620C99d213eB14"
	Quoter   = "0xB048Bbc1Ee6b733FFfCFb9e9CeF7375518e25997"
	Factory  = "0x0BFbCF9fa4f9C56B0F40a671Ad40E0805A091865"
)
```

买入和卖出都使用：

```solidity
exactInputSingle(ExactInputSingleParams params) payable returns (uint256 amountOut)
quoteExactInput(bytes path, uint256 amountIn) view returns (uint256 amountOut)
```

加仓的另一种写法使用：

```solidity
exactOutputSingle(ExactOutputSingleParams params) payable returns (uint256 amountIn)
quoteExactOutput(bytes path, uint256 amountOut) view returns (uint256 amountIn)
```

V3 path 编码规则：

```text
tokenIn(20 bytes) + fee(3 bytes) + tokenOut(20 bytes)
```

## 买入步骤

1. 连接 BSC RPC，加载私钥，得到 `from`。
2. 解析 `amountIn`，单位 wei，作为投入的 BNB。
3. 设置 `tokenIn=WBNB`，`tokenOut=req.TokenOut`。
4. 解析 V3 池费率 `fee`，例如 `500` 表示 0.05%。
5. 编码 path：`WBNB + fee + tokenOut`。
6. 调用 Quoter `quoteExactInput(path, amountIn)`，得到预计输出 `estimatedOut`。
7. 按滑点计算 `amountOutMinimum = estimatedOut * (1 - slippage)`。
8. 组装 `ExactInputSingleParams`：
   `tokenIn, tokenOut, fee, recipient, deadline, amountIn, amountOutMinimum, sqrtPriceLimitX96=0`。
9. ABI 打包 `exactInputSingle(params)`。
10. 构造交易：`to=V3Router`，`value=amountIn`，`data=exactInputSingle(...)`。
11. 签名发送，等待回执。

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

const pancakeV3RouterABI = `[
  {
    "inputs":[{"components":[
      {"internalType":"address","name":"tokenIn","type":"address"},
      {"internalType":"address","name":"tokenOut","type":"address"},
      {"internalType":"uint24","name":"fee","type":"uint24"},
      {"internalType":"address","name":"recipient","type":"address"},
      {"internalType":"uint256","name":"deadline","type":"uint256"},
      {"internalType":"uint256","name":"amountIn","type":"uint256"},
      {"internalType":"uint256","name":"amountOutMinimum","type":"uint256"},
      {"internalType":"uint160","name":"sqrtPriceLimitX96","type":"uint160"}
    ],"internalType":"struct IV3SwapRouter.ExactInputSingleParams","name":"params","type":"tuple"}],
    "name":"exactInputSingle",
    "outputs":[{"internalType":"uint256","name":"amountOut","type":"uint256"}],
    "stateMutability":"payable",
    "type":"function"
  }
]`

const pancakeV3QuoterABI = `[
  {"inputs":[{"internalType":"bytes","name":"path","type":"bytes"},{"internalType":"uint256","name":"amountIn","type":"uint256"}],"name":"quoteExactInput","outputs":[{"internalType":"uint256","name":"amountOut","type":"uint256"}],"stateMutability":"view","type":"function"}
]`

func EncodeV3Path(tokenIn, tokenOut common.Address, fee uint32) []byte {
	path := append(tokenIn.Bytes(), big.NewInt(int64(fee)).FillBytes(make([]byte, 3))...)
	path = append(path, tokenOut.Bytes()...)
	return path
}

func QuoteV3ExactInput(ctx context.Context, client *ethclient.Client, quoter common.Address, path []byte, amountIn *big.Int) (*big.Int, error) {
	qABI, err := abi.JSON(strings.NewReader(pancakeV3QuoterABI))
	if err != nil {
		return nil, err
	}
	data, err := qABI.Pack("quoteExactInput", path, amountIn)
	if err != nil {
		return nil, err
	}
	res, err := client.CallContract(ctx, ethereum.CallMsg{To: &quoter, Data: data}, nil)
	if err != nil {
		return nil, err
	}
	var amountOut *big.Int
	if err := qABI.UnpackIntoInterface(&amountOut, "quoteExactInput", res); err != nil {
		return nil, err
	}
	return amountOut, nil
}

func BuyPancakeV3(
	ctx context.Context,
	client *ethclient.Client,
	privateKeyHex string,
	tokenOutHex string,
	amountInWei *big.Int,
	fee uint32,
	slippagePercent string,
) (string, error) {
	router := common.HexToAddress("0x1b81D678ffb9C0263b24A97847620C99d213eB14")
	quoter := common.HexToAddress("0xB048Bbc1Ee6b733FFfCFb9e9CeF7375518e25997")
	wbnb := common.HexToAddress("0xBB4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c")
	tokenOut := common.HexToAddress(tokenOutHex)
	if fee == 0 {
		fee = 500
	}

	privateKey, err := crypto.HexToECDSA(privateKeyHex)
	if err != nil {
		return "", err
	}
	from := crypto.PubkeyToAddress(*privateKey.Public().(*ecdsa.PublicKey))

	path := EncodeV3Path(wbnb, tokenOut, fee)
	estimatedOut, err := QuoteV3ExactInput(ctx, client, quoter, path, amountInWei)
	if err != nil {
		return "", err
	}
	slip, err := strconv.ParseFloat(slippagePercent, 64)
	if err != nil {
		return "", err
	}
	minOutFloat := new(big.Float).Mul(new(big.Float).SetInt(estimatedOut), big.NewFloat(1-slip/100))
	amountOutMinimum, _ := minOutFloat.Int(nil)

	params := struct {
		TokenIn           common.Address
		TokenOut          common.Address
		Fee               *big.Int
		Recipient         common.Address
		Deadline          *big.Int
		AmountIn          *big.Int
		AmountOutMinimum  *big.Int
		SqrtPriceLimitX96 *big.Int
	}{
		TokenIn:           wbnb,
		TokenOut:          tokenOut,
		Fee:               new(big.Int).SetUint64(uint64(fee)),
		Recipient:         from,
		Deadline:          big.NewInt(time.Now().Add(10 * time.Minute).Unix()),
		AmountIn:          amountInWei,
		AmountOutMinimum:  amountOutMinimum,
		SqrtPriceLimitX96: big.NewInt(0),
	}

	routerABI, err := abi.JSON(strings.NewReader(pancakeV3RouterABI))
	if err != nil {
		return "", err
	}
	data, err := routerABI.Pack("exactInputSingle", params)
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
	gasLimit, err := client.EstimateGas(ctx, ethereum.CallMsg{From: from, To: &router, Value: amountInWei, Data: data})
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
	_, err = bind.WaitMined(ctx, client, signed)
	return signed.Hash().Hex(), err
}
```

## 卖出步骤

1. 连接 BSC RPC，加载私钥，得到 `from`。
2. 设置 `tokenIn=req.TokenOut`，`tokenOut=WBNB`。
3. 读取 `tokenIn.balanceOf(from)`。
4. 按卖出百分比计算 `amountIn`。
5. 对 V3 Router 执行 `approve(router, amountIn)`，等待授权确认。
6. 编码 path：`tokenIn + fee + WBNB`。
7. 调用 Quoter `quoteExactInput(path, amountIn)`，得到预计输出 WBNB。
8. 按滑点计算 `amountOutMinimum`。
9. 组装 `ExactInputSingleParams`，打包 `exactInputSingle(params)`。
10. 构造交易：`to=V3Router`，`value=0`，`data=exactInputSingle(...)`。
11. 签名发送，等待回执。

## 卖出模板代码

```go
const erc20V3ABI = `[
  {"constant":true,"inputs":[{"name":"owner","type":"address"}],"name":"balanceOf","outputs":[{"name":"","type":"uint256"}],"type":"function"},
  {"constant":false,"inputs":[{"name":"spender","type":"address"},{"name":"value","type":"uint256"}],"name":"approve","outputs":[{"name":"","type":"bool"}],"type":"function"}
]`

func SellPancakeV3(
	ctx context.Context,
	client *ethclient.Client,
	privateKeyHex string,
	tokenInHex string,
	percent float64,
	fee uint32,
	slippagePercent string,
) (string, error) {
	router := common.HexToAddress("0x1b81D678ffb9C0263b24A97847620C99d213eB14")
	quoter := common.HexToAddress("0xB048Bbc1Ee6b733FFfCFb9e9CeF7375518e25997")
	tokenIn := common.HexToAddress(tokenInHex)
	wbnb := common.HexToAddress("0xBB4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c")
	if fee == 0 {
		fee = 500
	}

	privateKey, err := crypto.HexToECDSA(privateKeyHex)
	if err != nil {
		return "", err
	}
	from := crypto.PubkeyToAddress(*privateKey.Public().(*ecdsa.PublicKey))

	tokenABI, err := abi.JSON(strings.NewReader(erc20V3ABI))
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

	path := EncodeV3Path(tokenIn, wbnb, fee)
	estimatedOut, err := QuoteV3ExactInput(ctx, client, quoter, path, amountIn)
	if err != nil {
		return "", err
	}
	slip, err := strconv.ParseFloat(slippagePercent, 64)
	if err != nil {
		return "", err
	}
	minOutFloat := new(big.Float).Mul(new(big.Float).SetInt(estimatedOut), big.NewFloat(1-slip/100))
	amountOutMinimum, _ := minOutFloat.Int(nil)

	params := struct {
		TokenIn           common.Address
		TokenOut          common.Address
		Fee               *big.Int
		Recipient         common.Address
		Deadline          *big.Int
		AmountIn          *big.Int
		AmountOutMinimum  *big.Int
		SqrtPriceLimitX96 *big.Int
	}{
		TokenIn:           tokenIn,
		TokenOut:          wbnb,
		Fee:               new(big.Int).SetUint64(uint64(fee)),
		Recipient:         from,
		Deadline:          big.NewInt(time.Now().Add(10 * time.Minute).Unix()),
		AmountIn:          amountIn,
		AmountOutMinimum:  amountOutMinimum,
		SqrtPriceLimitX96: big.NewInt(0),
	}

	routerABI, err := abi.JSON(strings.NewReader(pancakeV3RouterABI))
	if err != nil {
		return "", err
	}
	data, err := routerABI.Pack("exactInputSingle", params)
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

