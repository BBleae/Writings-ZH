  
发现新的 Raydium 交易对
===================

概述
---

在本文中，我将介绍我在 Raydium（Solana 区块链）市场上检测新的 X/SOL 交易对的方法。由于我在 Solana 方面仍然是初学者，这不会是一个深入的分析，而是一个简单的“操作指南”，作为进一步研究和寻找优化形式的良好开端。需要具备 Golang 的基本知识（代码片段将使用这种语言）和 Solana 生态系统的基本知识。

这篇文章是写给像我一样在 Solana 生态系统中初出茅庐的程序员的，尽管对这个生态系统有基本的了解，但他们在解决以下问题时仍然难以找到任何想法。

要解决的问题
-------

我们想要解决的问题是如何在 Raydium 市场上为新创建的交易对自动生成买单（可能也包括卖单）。为了解决这个问题，我们需要知道两件事：如何检测 Raydium 市场上的新交易对以及如何自动生成买单。让我们从后者开始。

Raydium 交换
-----------

每次我们在[Raydium](https://raydium.io/swap/)应用程序中买卖代币时，都会生成相应的交易并发送到区块链。包含买单的示例交易[看起来像这样](https://solscan.io/tx/2xDzRw7QamtB8CMJJxi6SFQ4FSE88kELVTgwzRyWQB5zc158yUZY4SLNspjAbw5AM6pVEFCVraxa1cwPPtgew3py)。这里最重要的是由`Raydium Liquidity Pool V4`程序（地址： `675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8` ）执行的指令#7。请注意，该语句包含多个我们需要以某种方式获取的账户作为输入参数。已知的是：

*
    程序： `675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8` (Raydium 程序)
*
    输入账户 #1: `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` (代币程序)
*
    输入账户 #3: `5Q544fKrFoe6tsEbD7S8EmxGTJYAKtTVhAW5Q5pge4j1` (Raydium 权限)
*
    输入账户 #8: `srmqPvymJeFKQ4zGQed1GFppgkRHL9kaELCbyksJtPX` (OpenBook/Serum 程序)

输入账户 #16、#17、#18 和参数 #19、#20 是用户提供的参数，因此：

*
    输入账户 #16: 将从中收集 SOL 的代币账户（在购买的情况下）
*
    InputAccount #17: 购买的代币将进入的 TokenAccount；当我们第一次购买代币时，必须首先创建这样的账户；在这个特定的交易中，声明#6 负责此事
*
    输入账户 #18：我们的钱包地址
*
    AmountIn #19：我们想要花费在购买代币上的 lamports 数量；1'000'000'000 lamports = 1 SOL
*
    MinAmountOut #20：我们期望出现在我们账户中的最少代币数量是多少；对于这笔交易，0 相当于将滑点参数设置为 100%

指令数据是指令的输入数据，我们对给定的交易进行如下解码：

[00:01] - 在`Swap`指令的情况下，第一个字节总是取值 9 (09)  
  
[01:09] - 接下来的八个字节是小端序的`AmountIn #19`参数（00ca9a3b00000000 等于 1'000'000'000）  
  
[09:11] - 接下来的八个字节是小端序的`MinAmountOut #20`参数（00000000000000000 是 0）

鉴于上述情况，这意味着必须以某种方式从区块链中获取 `InputAccount #2, #4, #5, #6, #7, #9, #10, #11, #12, #13, #14, #15` 参数。我们将通过监听区块链中的相关交易来实现这一点。

额外链接到负责在应用程序中生成交易的 Raydium 代码：[https://github.com/raydium-io/raydium-sdk/blob/master/src/liquidity/liquidity.ts#L1026](https://github.com/raydium-io/raydium-sdk/blob/master/src/liquidity/liquidity.ts#L1026)

值得在读完整篇文章后看一看。

新池流量
-----

现在我们知道了发送买单所需的内容，是时候看看在 Raydium 上创建新交易对的示例流程了。

1.
    创建新代币

这一点从检测新对的角度来看并不重要，但与代币本身相关的额外（元）数据可能是有用的，例如，用于确定代币是由人类还是脚本生成的，并在决定是否生成买单时提供有用的信息。在区块链上创建代币的示例交易[看起来像这样](https://solscan.io/tx/5FsUJx29VdKi59UmEa783ehAnRF5gMzFP1hvk5icQhxmAA8xsG6qE3rB5gd7kwkJiWaMTvLGSnmM72aaZNXJ9NHg)。最重要的是 Metaplex 程序的指令#5，但我不会在本文中解释。

2.
    血清市场的创建 (OpenBook)

这是必须检测到的第一个交易，以正确识别 Raydium 市场中的新交易对。你可以通过搜索 Google 来了解 Serum 是什么（一个好的开始：[https://docs.projectserum.com/introduction/overview](https://docs.projectserum.com/introduction/overview)）。简而言之，它是运行在 Solana 生态系统中的订单簿。我找不到任何提供最新市场信息的公共来源，所以我开始自己分析负责创建新市场的交易。

[这就是](https://solscan.io/tx/2PDK8UsWbutpSuJ7r8czxyaZFgrj5mbtEXxAh2n8tD2ZNHG11XQXVWAgH9xrVRMwNRte2snkmct8iqn9tvzZrTQg)创建 Serum 新市场的示例交易的样子。这里最重要的指令是第 6 条。这是由地址为 `srmqPvymJeFKQ4zGQed1GFppgkRHL9kaELCbyksJtPX` 的 OpenBook（Serum）程序执行的指令。如果你已经检查了此指令的输入参数，你可能已经注意到一些参数名称与 Raydium 的`Swap`指令的参数名称重叠。确实如此，`InitMarket`指令直接为我们提供了一些执行`Swap`指令所需的参数。这些是：

*
    InputAccount #1: 相当于 `Swap` 指令中的 InputAccount #9 (#Market -> #SerumMarket)
*
    InputAccount #3: 相当于 `Swap` 指令中的 InputAccount #12 (#EventQueue -> #SerumEventQueue)
*
    InputAccount #4：相当于`Swap`指令中的 InputAccount #10 (#Bids -> #SerumBids)
*
    InputAccount #5: 相当于来自`Swap`指令的 InputAccount #11 (#Asks -> #SerumAsks)
*
    InputAccount #6: 相当于来自`Swap`指令的 InputAccount #13 (#BaseVault -> #SerumCoinVaultAccount)
*
    InputAccount #7：相当于`Swap`指令中的 InputAccount #14 (#QuoteVault -> #SerumPcVaultAccount)

我不完全理解命名法，但默认情况下（基于对各种交易的观察），Base/Coin 对应于代币，而 Quote/Pc 对应于货币（SOL、USDC 等）。然而，这并不总是如此，交易对可以反过来创建，这对于`Swap`指令的正确性很重要。

`InitMarket` 还间接为我们提供了所有剩余的参数：

*
    VaultSignerNonce #14：用于从`Swap`指令中计算 InputAccount #15 的地址 (#SerumVaultSigner)
*
    InputAccount #1 (#Market)：用于计算 InputAccount #2、#4、#5、#6、#7 (#AmmId、#AmmOpenOrders、#AmmTargetOrders、#PoolCoinTokenAccount、#PoolPcTokenAccount) 的地址

指令数据是指令的输入数据，我们对给定的交易进行解码如下：

[00:05] - 如果我理解正确的话，前五个字节与市场/指令版本有关 (via: [https://github.com/project-serum/serum-dex/blob/master/dex/src/instruction.rs#L327](https://github.com/project-serum/serum-dex/blob/master/dex/src/instruction.rs#L327) ) (0000000000)  
  
[05:0D] - 接下来的八个字节是`BaseLotSize #10`参数，按小端序排列（00e1f50500000000 - 对我们来说不重要）  
  
[0D:15] - 接下来的八个字节是`QuoteLotSize #11`参数，以小端序排列（10270000000000000 - 对我们来说不重要）  
  
[15:17] - 接下来的两个字节是 `FeeRateBps #12` 参数，按小端序排列（9600 - 对我们来说不重要）  
  
[17:1F] - 接下来的八个字节是`VaultSignerNonce #14`参数，以小端序排列 (02000000000000000 - **需要**)  
  
[1F:27] - 接下来的八个字节是`QuoteDustThreshold #13`参数，以小端序排列（f401000000000000 - 对我们来说不重要）

有了`VaultSignerNonce`，我们可以根据此代码计算`SerumVaultSigner`的地址：[https://github.com/raydium-io/raydium-sdk/blob/master/src/utils/createMarket.ts#L90](https://github.com/raydium-io/raydium-sdk/blob/master/src/utils/createMarket.ts#L90)

拥有`SerumMarket`我们可以使用此代码计算 `AmmId, AmmOpenOrders, AmmTargetOrders, PoolCoinTokenAccount, PoolPcTokenAccount` 的地址：[https://github.com/raydium-io/raydium-sdk/blob/master/src/liquidity/liquidity.ts#](https://github.com/raydium-io/raydium-sdk/blob/master/src/liquidity/liquidity.ts#) L416-L559

别害怕，我会教你如何用 Go 完成所有事情。

3.
    添加 Raydium 的流动性池到 Serum 市场

这一步是不需要的，因为我们已经拥有生成任何有效`Swap`语句所需的所有信息。然而，添加流动性交易也值得关注，因为它们提供了在做出购买订单决策时有用的额外信息：池的大小（池中的代币和 SOL 数量）、市场开放时间以及 LP 代币将被交付的地址。

[这就是](https://solscan.io/tx/4CgGAnctaNtCWzsdVpVq3Vb43RnRcEGvvo7nTqA6PtHskSMpZxZxzuiTSbg6DcagB8jbLAFGdP8GUxHAhnBDf4VL) 在 Raydium 上增加流动性的示例交易的样子。这里最重要的指令是第 5 条。事实上，我们不再需要从这个声明中提取任何 InputAccount，有用的信息隐藏在声明的输入数据中。

指令数据是指令的输入数据，我们对给定交易进行解码如下：

[00:01] - 在`IDO Purchase`指令的情况下，第一个字节总是取值 1 (01)  
  
[01:02] - 下一个字节是随机数 (fe - 254)  
  
[02:0A] - 接下来的八个字节是市场的开盘时间；`数量 #13` 作为小端序的 Unix 时间戳 (52e995650000000000 - 1'704'323'410 - 2024 年 1 月 3 日星期三 23:10:10 GMT+0000)  
  
[0A:12] - 接下来的八个字节是池中`pc`令牌的数量，以小端序表示 (0060891c05600300 - 950'000'000'000'000)  
  
[12:2A] - 接下来的八个字节是池中`coin`代币的数量，采用小端序（00e40b5402000000 - 10'000'000'000）

我之前提到过，`pc` 通常是货币 (SOL)，而 `coin` 是我们想要的代币，但正如你在这个交易的例子中看到的那样，情况正好相反，所以总是值得检查池中地址和代币值的正确性。

指令数据输入布局可以在这里找到：[https://github.com/raydium-io/raydium-sdk/blob/master/src/liquidity/liquidity.ts#L1596](https://github.com/raydium-io/raydium-sdk/blob/master/src/liquidity/liquidity.ts#L1596)

如果我们查看负责创建指令的代码：[https://github.com/raydium-io/raydium-sdk/blob/master/src/liquidity/liquidity.ts#L1598](https://github.com/raydium-io/raydium-sdk/blob/master/src/liquidity/liquidity.ts#L1598)，我们会看到用于生成交易的账户比在 Solscan 上显示的更多。通过在 Explorer 中打开相同的交易，这一点显而易见：[https://explorer.solana.com/tx/4CgGAnctaNtCWzsdVpVq3Vb43RnRcEGvvo7nTqA6PtHskSMpZxZxzuiTSbg6DcagB8jbLAFGdP8GUxHAhnBDf4VL](https://explorer.solana.com/tx/4CgGAnctaNtCWzsdVpVq3Vb43RnRcEGvvo7nTqA6PtHskSMpZxZxzuiTSbg6DcagB8jbLAFGdP8GUxHAhnBDf4VL)

两个有用的地址是`账户 #18`（添加流动性的钱包地址）和`账户 #21`（LP 代币的目标地址）。

代码示例
-----

在我们开始之前，一些有用的链接：

*
    Solana RPC 主网节点配置：[https://docs.solana.com/cluster/rpc-endpoints](https://docs.solana.com/cluster/rpc-endpoints)
*
    RPC 节点 http API: [https://docs.solana.com/api/http](https://docs.solana.com/api/http)
*
    RPC 节点 websocket API: [https://docs.solana.com/api/websocket](https://docs.solana.com/api/websocket)
*
    Go 包用于 Solana：[https://github.com/gagliardetto/solana-go](https://github.com/gagliardetto/solana-go)

请注意，尽管代码看起来是线性的，但逐个复制每个片段不会创建正确的文件。每个片段是一个独立的逻辑部分，展示了逐步检测新对的过程。

请注意，使用单个`mainnet-beta` RPC 节点可能由于速率限制而不足。

1.
    我们将首先加载我们的钱包，我们将用它来签署交易：

    package main

    import (
        "os"

        bin "github.com/gagliardetto/binary"
        "github.com/gagliardetto/solana-go"
        "github.com/gagliardetto/solana-go/rpc"
        "github.com/gagliardetto/solana-go/rpc/ws"
    )

    func main() {
        data, err := os.ReadFile("c:/path/to/private/key/in/base.58")
        if err != nil {
            panic(err)
        }

        wallet := solana.MustPrivateKeyFromBase58(string(data))
        ...

2.
    我们将创建一个新的 websocket 和 rpc 连接到节点，以监听与 OpenBook 相关的交易日志：

        ...
        wsClient, err := ws.Connect(ctx, rpc.MainNetBeta_WS)
        if err != nil {
            panic(err)
        }
    
        rpcClient, err := rpc.Connect(ctx, rpc.MainNetBeta_RPC) // Will be used later.
        if err != nil {
            panic(err)
        }
    
        defer wsClient.Close()
    
        var OpenBookDex solana.PublicKey = solana.MustPublicKeyFromBase58("srmqPvymJeFKQ4zGQed1GFppgkRHL9kaELCbyksJtPX")
        subID, err := wsClient.LogsSubscribeMentions(OpenBookDex, rpc.CommitmentProcessed) // CommitmentProcessed will give us information about the transaction as soon as possible, but there is a possibility that it will not be approved in the blockchain, so we have to check it ourselves.
        if err != nil {
            panic(err)
        }
    
        defer subID.Unsubscribe()
        ...

3.
    然后我们将开始监听（注意 Recv 函数是阻塞的，并且有很多事务，因此值得将日志的接收和处理分成单独的 goroutine）：

        ...
        for {
            log, err := subID.Recv()
            if err != nil {
                // Handle error.
            }
    
            // We don't want to process logs of failed transactions.
            if log.Value.Logs == nil || log.Value.Err != nil {
                continue 
            }
    
            foundCandidate := false
    
            // We parse the logs looking for potential InitMarket instructions.
            // The pattern below may not be 100% correct and may change in the future, but for a simple example it will work.
            for i := range log.Value.Logs {
                curLog := log.Value.Logs[i]
                if !strings.Contains(curLog, "Program 11111111111111111111111111111111 success") {
                    continue // Search further.
                }
    
                if i+1 >= len(log.Value.Logs) {
                    break // No more logs.
                }
    
                nextLog := log.Value.Logs[i+1]
                if !strings.Contains(nextLog, "Program srmqPvymJeFKQ4zGQed1GFppgkRHL9kaELCbyksJtPX invoke [1]") {
                    continue // Search further.
                }
    
                // This could be InitMarket instruction.
                foundCandidate = true
                break
            }
    
            if !foundCandidate {
                continue
            }
            ...

4.
    一旦我们根据解析的日志找到了一个可能的 InitMarket 交易候选者，我们需要检索具有完整详细信息的交易：

            ...
            Max_Transaction_Version := 1 // Needed to be able to retrieve non-legacy transactions.
            ctx, cancel := context.WithTimeout(ctx, 15*time.Second)
            rpcTx, err := rpcClient.GetTransaction(rctx, log.Value.Signature, &rpc.GetTransactionOpts{
                MaxSupportedTransactionVersion: &Max_Transaction_Version,
                Commitment:                     rpc.CommitmentConfirmed, // GetTransaction doesn't work with CommitmentProcessed. This is also blocking function so worth moving to separate goroutine. 
            })
            cancel()
    
            if err != nil {
                // Handle error: retry/abort etc.
                continue
            }
    
            // Check if transactions failed. Don't know if its needed as we checked that earlier on when getting logs, but doesn't cost much.
            if rpcTx.Meta.Err != nil {
                continue // Transaction failed, nothing to do with it anymore.
            }
    
            tx, err := rpcTx.Transaction.GetTransaction()
            if err != nil {
                continue // Error getting solana.Transaction object from rpc object.
            }
            ...

5.
    此时，我们已经拥有了包含所有细节的交易。然而，它仍然是一个候选交易。我们需要分析它，以确定这是否是我们感兴趣的交易：

            ...
            type market struct {
                // Serum
                Market solana.PublicKey
                EventQueue solana.PublicKey
                Bids solana.PublicKey
                Asks solana.PublicKey
                BaseVault solana.PublicKey
                QuoteVault solana.PublicKey
                BaseMint solana.PublicKey
                QuoteMint solana.PublicKey
                VaultSigner solana.PublicKey
    
                // Raydium
                AmmID solana.PublicKey
                AmmPoolCoinTokenAccount solana.PublicKey
                AmmPoolPcTokenAccount solana.PublicKey
                AmmPoolTokenMint solana.PublicKey
                AmmTargetOrders solana.PublicKey
                AmmOpenOrders solana.PublicKey
            }
    
            var m market
    
            for _, instr := range tx.Message.Instructions {
                program, err := tx.Message.Program(instr.ProgramIDIndex)
                if err != nil {
                    continue // Program account index out of range.
                }
    
                if program.String() != OpenBookDex.String() {
                    continue // Instruction not called by OpenBook, check next.
                }
    
                if len(instr.Accounts) < 10 {
                    continue // Not enough accounts for InitMarket instruction.
                }
    
                if len(instr.Data) < 37 {
                    continue // Not enough input data for InitMarket instruction.
                }
    
                const BaseMintIndex = 7
                const QuoteMintIndex = 8
    
                // Just some helper function
                safeIndex := func(idx uint16) solana.PublicKey {
                    if idx >= uint16(len(tx.Message.AccountKeys)) {
                        return solana.PublicKey{}
                    }
                    return tx.Message.AccountKeys[idx]
                }
    
                // In this example we search for pairs X/SOL only.
                if safeIndex(instr.Accounts[QuoteMintIndex]) != solana.WrappedSol && safeIndex(instr.Accounts[BaseMintIndex]) != solana.WrappedSol {
                    fmt.Printf("Found serum market, but not with SOL currency")
                    break
                }
    
                // At this point, its probably InitMarket instruction.
                m.Market = safeIndex(instr.Accounts[0])
                m.EventQueue = safeIndex(instr.Accounts[2])
                m.Bids = safeIndex(instr.Accounts[3])
                m.Asks = safeIndex(instr.Accounts[4])
                m.BaseVault = safeIndex(instr.Accounts[5])
                m.QuoteVault = safeIndex(instr.Accounts[6])
                m.BaseMint = safeIndex(instr.Accounts[7])
                m.QuoteMint = safeIndex(instr.Accounts[8])
    
                caller := tx.Message.AccountKeys[0] // Additional info. Use if needed.
    
                // Get VaultSignerNonce from instruction data and calculate address for SerumVaultSigner.
                vaultSignerNonce := instr.Data[23:31] 
                vaultSigner, err := solana.CreateProgramAddress(
                    [][]byte{
                        m.Market.Bytes()[:],
                        vaultSignerNonce,
                    }, OpenBookDex)
    
                if err != nil {
                    fmt.Printf("Error while creating vault signer account: %v", err)
                    break
                }
    
                m.VaultSigner = vaultSigner
            }
    
            // Check if all accounts are filled with values. Im leaving that to the reader.
            if !m.valid() {
                fmt.Printf("Not a InitMarket transaction")
                continue
            }
            ...

6.
    在这一点上，我们可以确定这是一个 InitMarket 交易。现在是时候计算缺失的地址了：

            ...
            // Helper function.
            associatedAccount := func(market solana.PublicKey, seed []byte) (solana.PublicKey, error) {
                Raydium_Liquidity_Program_V4 := solana.MustPublicKeyFromBase58("675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8")
                acc, _, err := solana.FindProgramAddress([][]byte{
                    Raydium_Liquidity_Program_V4.Bytes(),
                    market.Bytes(),
                    seed,
                }, Raydium_Liquidity_Program_V4)
                return acc, err
            }
    
            ammID, err := associatedAccount(m.Market, []byte("amm_associated_seed"))
            if err != nil {
                // Handle error
            }
            m.AmmID = ammID
    
            ammPoolCoinTokenAccount, err := associatedAccount(m.Market, []byte("coin_vault_associated_seed"))
            if err != nil {
                // Handle error
            }
            m.AmmPoolCoinTokenAccount = ammPoolCoinTokenAccount
    
            ammPoolPcTokenAccount, err := associatedAccount(m.Market, []byte("pc_vault_associated_seed"))
            if err != nil {
                // Handle error
            }
            m.AmmPoolPcTokenAccount = ammPoolPcTokenAccount
    
            ammPoolTokenMint, err := associatedAccount(m.Market, []byte("lp_mint_associated_seed"))
            if err != nil {
                // Handle error
            }
            m.AmmPoolTokenMint = ammPoolTokenMint
    
            ammTargetOrders, err := associatedAccount(m.Market, []byte("target_associated_seed"))
            if err != nil {
                // Handle error
            }
            m.AmmTargetOrders = ammTargetOrders
    
            ammOpenOrders, err := associatedAccount(m.Market, []byte("open_order_associated_seed"))
            if err != nil {
                // Handle error
            }
            m.AmmOpenOrders = ammOpenOrders
            ...

7.
    此时我们已经拥有生成交换指令所需的所有地址。从带有`IDO Purchase`指令的交易中提取额外信息与上述类似，因此我将跳过它，因为没有必要。我将直接进入在 Raydium 上生成买单的阶段。我将在这里展示的代码基于仓库 `https://github.com/gopartyparrot/goparrot-twap` 中的代码：

            ...
    
            // This must implement solana.Instruction interface.
            type RaySwapInstruction struct {
                bin.BaseVariant
                InAmount                uint64
                MinimumOutAmount        uint64
                solana.AccountMetaSlice `bin:"-" borsh_skip:"true"`
            }
    
            func NewRaydiumSwapInstruction(
                // Parameters:
                inAmount uint64,
                minimumOutAmount uint64,
                // Accounts:
                tokenProgram solana.PublicKey,
                ammId solana.PublicKey,
                ammAuthority solana.PublicKey,
                ammOpenOrders solana.PublicKey,
                ammTargetOrders solana.PublicKey,
                poolCoinTokenAccount solana.PublicKey,
                poolPcTokenAccount solana.PublicKey,
                serumProgramId solana.PublicKey,
                serumMarket solana.PublicKey,
                serumBids solana.PublicKey,
                serumAsks solana.PublicKey,
                serumEventQueue solana.PublicKey,
                serumCoinVaultAccount solana.PublicKey,
                serumPcVaultAccount solana.PublicKey,
                serumVaultSigner solana.PublicKey,
                userSourceTokenAccount solana.PublicKey,
                userDestTokenAccount solana.PublicKey,
                userOwner solana.PublicKey,
            ) *RaySwapInstruction {
    
                inst := RaySwapInstruction{
                    InAmount:         inAmount,
                    MinimumOutAmount: minimumOutAmount,
                    AccountMetaSlice: make(solana.AccountMetaSlice, 18),
                }
                inst.BaseVariant = bin.BaseVariant{
                    Impl: inst,
                }
    
                inst.AccountMetaSlice[0] = solana.Meta(tokenProgram)
                inst.AccountMetaSlice[1] = solana.Meta(ammId).WRITE()
                inst.AccountMetaSlice[2] = solana.Meta(ammAuthority)
                inst.AccountMetaSlice[3] = solana.Meta(ammOpenOrders).WRITE()
                inst.AccountMetaSlice[4] = solana.Meta(ammTargetOrders).WRITE()
                inst.AccountMetaSlice[5] = solana.Meta(poolCoinTokenAccount).WRITE()
                inst.AccountMetaSlice[6] = solana.Meta(poolPcTokenAccount).WRITE()
                inst.AccountMetaSlice[7] = solana.Meta(serumProgramId)
                inst.AccountMetaSlice[8] = solana.Meta(serumMarket).WRITE()
                inst.AccountMetaSlice[9] = solana.Meta(serumBids).WRITE()
                inst.AccountMetaSlice[10] = solana.Meta(serumAsks).WRITE()
                inst.AccountMetaSlice[11] = solana.Meta(serumEventQueue).WRITE()
                inst.AccountMetaSlice[12] = solana.Meta(serumCoinVaultAccount).WRITE()
                inst.AccountMetaSlice[13] = solana.Meta(serumPcVaultAccount).WRITE()
                inst.AccountMetaSlice[14] = solana.Meta(serumVaultSigner)
                inst.AccountMetaSlice[15] = solana.Meta(userSourceTokenAccount).WRITE()
                inst.AccountMetaSlice[16] = solana.Meta(userDestTokenAccount).WRITE()
                inst.AccountMetaSlice[17] = solana.Meta(userOwner).SIGNER().WRITE()
    
                return &inst
            }
    
            func (inst *RaySwapInstruction) ProgramID() solana.PublicKey {
                return Raydium_Liquidity_Program_V4 // Defined in one of previous snippets.
            }
    
            func (inst *RaySwapInstruction) Accounts() (out []*solana.AccountMeta) {
                return inst.Impl.(solana.AccountsGettable).GetAccounts()
            }
    
            func (inst *RaySwapInstruction) Data() ([]byte, error) {
                buf := new(bytes.Buffer)
                if err := bin.NewBorshEncoder(buf).Encode(inst); err != nil {
                    return nil, fmt.Errorf("unable to encode instruction: %w", err)
                }
                return buf.Bytes(), nil
            }
    
            func (inst *RaySwapInstruction) MarshalWithEncoder(encoder *bin.Encoder) (err error) {
                // Swap instruction is number 9
                err = encoder.WriteUint8(9)
                if err != nil {
                    return err
                }
                err = encoder.WriteUint64(inst.InAmount, binary.LittleEndian)
                if err != nil {
                    return err
                }
                err = encoder.WriteUint64(inst.MinimumOutAmount, binary.LittleEndian)
                if err != nil {
                    return err
                }
                return nil
            }
            ...

8.
    尽管有交换指令，我们还需要生成前面的交易，这些交易将创建一个临时 SOL 账户和一个给定代币的新关联代币账户（如果我们是第一次购买代币，否则不需要创建 ATA）：

            ...
            tokenAddress := market.BaseMint
            instrs = []solana.Instruction{}
            signers = []solana.PrivateKey{wallet} // Wallet loaded in the first snippet.
    
            // Create temporary account for the SOL that will be used as a source for the swap.
            tempSpenderWallet := solana.NewWallet()
    
            // Create ATA account for the token and buyer (assume it is first buy for the token).
            ataAccount, _, err := solana.FindAssociatedTokenAddress(wallet.PublicKey(), tokenAddress)
            if err != nil {
                return nil, nil, err
            }
    
            // Calculate how much lamports we need to create temporary account.
            // rentCost is the value returned by `getMinimumBalanceForRentExemption` rpc call.
            // amountInLamports is user defined value (how much SOL in lamports to spend for buying)
            accountLamports := rentCost + amountInLamports
    
            // Create ATA account for the token and buyer account.
            createAtaInst, err := associatedtokenaccount.NewCreateInstruction(
                wallet.PublicKey(), wallet.PublicKey(), tokenAddress,
            ).ValidateAndBuild()
    
            if err != nil {
                return nil, nil, err
            }
    
            // Create temporary account for the SOL to be swapped
            createAccountInst, err := system.NewCreateAccountInstruction(
                accountLamports,
                165, // TokenAccount size is 165 bytes always.
                solana.TokenProgramID,
                wallet.PublicKey(),
                tempSpenderWallet.PublicKey(),
            ).ValidateAndBuild()
    
            if err != nil {
                return nil, nil, err
            }
    
            // Initialize the temporary account
            initAccountInst, err := token.NewInitializeAccountInstruction(
                tempSpenderWallet.PublicKey(),
                solana.WrappedSol,
                wallet.PublicKey(),
                solana.SysVarRentPubkey,
            ).ValidateAndBuild()
    
            if err != nil {
                return nil, nil, err
            }
    
            // Create the swap instruction
            swapInst := raydium.NewRaydiumSwapInstruction(
                amountInLamports,
                minimumOutTokens,                       // This is defined by user. 0 means the swap will use 100% slippage (used often in bots).
                solana.TokenProgramID,
                market.AmmID,                           // AmmId
                Raydium_Authority_Program_V4,           // AmmAuthority
                market.AmmOpenOrders,                   // AmmOpenOrders
                market.AmmTargetOrders,                 // AmmTargetOrders
                market.AmmPoolCoinTokenAccount,         // PoolCoinTokenAccount
                market.AmmPoolPcTokenAccount,           // PoolPcTokenAccount
                serum.OpenBookDex,                      // SerumProgramId - OpenBook
                market.Market,                          // SerumMarket
                market.Bids,                            // SerumBids
                market.Asks,                            // SerumAsks
                market.EventQueue,                      // SerumEventQueue
                market.BaseVault,                       // SerumCoinVaultAccount
                market.QuoteVault,                      // SerumPcVaultAccount
                market.VaultSigner,                     // SerumVaultSigner
                tempSpenderWallet.PublicKey(),          // UserSourceTokenAccount
                ataAccount,                             // UserDestTokenAccount
                wallet.PublicKey(),                     // UserOwner (signer)
            )
    
            // Close temporary account
            closeAccountInst, err := token.NewCloseAccountInstruction(
                tempSpenderWallet.PublicKey(),
                wallet.PublicKey(),
                wallet.PublicKey(),
                []solana.PublicKey{},
            ).ValidateAndBuild()
    
            if err != nil {
                return nil, nil, err
            }
    
            // Set instructions and signers.
            instrs = append(instrs, createAtaInst, createAccountInst, initAccountInst, swapInst, closeAccountInst) // Preserve order.
            signers = append(signers, tempSpenderWallet.PrivateKey)
            ...

9.
    最后一步是签署并发送交易：

            ...
            // Create transaction from instructions.
            tx, err := solana.NewTransaction(
                instrs,
                latestblockhash,    // Latest Blockhash can be retrieved with `getRecentBlockhash` rpc call.
                solana.TransactionPayer(signers[0].PublicKey()),
            )
    
            if err != nil {
                return nil, err
            }
    
            // Sign transaction.
            _, err = tx.Sign(
                func(key solana.PublicKey) *solana.PrivateKey {
                    for _, payer := range signers {
                        if payer.PublicKey().Equals(key) {
                            return &payer
                        }
                    }
                    return nil
                },
            )
    
            if err != nil {
                return nil, err
            }
    
            MaxRetries := 5
            ctx, cancel := context.WithTimeout(ctx, 15*time.Second)
            sig, err := rpcClient.SendTransactionWithOpts(
                ctx,
                tx,
                rpc.TransactionOpts{ // Use proper options for your case.
                    SkipPreflight: true,
                    MaxRetries:    &MaxRetries,
                },
            )
            cancel()
    
            if err != nil {
                // Handle error
            }
    
            // Track (or not) transaction status to know it was properly stored in blockchain.
            
            // Repeat process for all logs got from websocket client.

优化可能性
--------

这里介绍的方法不是最优的，但它能完成任务。优化整个过程的方法之一是使用您自己的或专用的 Solana 节点，这样您将直接访问交易，并且流程将从以下方面改变：

订阅交易日志 > 接收交易日志 > 解析日志并选择候选者 > 向 RPC 节点发送请求获取完整交易 > 解析交易数据

 到：

订阅完整交易 > 接收交易 > 解析交易数据

更新日志
-----

*
    [17.01.2024] 文件初始版本。

@media (prefers-color-scheme: dark) { body { color: #fff !important; background-color: #272727 !important; } }
