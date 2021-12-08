## zksync contracts

contracts中的合约是直接部署到L1上的，目前zksync已经有了自己的合约语言 znic 使用zkevm作为其虚拟机。

`contracts/contracts/Governance.sol`

addToken 将token 添加至网络中

```solidity
function addToken(address _token)external{
		requireGovernor(msg.sender);  // 检查给定的地址是不是ggovernor
        require(tokenIds[_token] == 0, "1e"); // token exists
        require(totalTokens < MAX_AMOUNT_OF_REGISTERED_TOKENS, "1f"); // no free identifiers for tokens

        totalTokens++;
        uint16 newTokenId = totalTokens; // it is not `totalTokens - 1` because tokenId = 0 is reserved for eth

        tokenAddresses[newTokenId] = _token;
        tokenIds[_token] = newTokenId;
        emit NewToken(_token, newTokenId);	

}
```



