# Dex_001_sori

~~~
//Dex.sol:Dex, swap() line88-89

swap_amount = pool_b - ((pool_a * pool_b) / (pool_a + a_token_amount));

swap_amount = swap_amount * fee / 1000;

~~~

**Vector: The liquidity can be drained.

**Severity Level: Critical(drain issue)**

### Description

In the swap function, the swap_amount can be equal to pool_b, if the attacker puts an massive a_token_amount to the parameter. Here's a concrete example:

1. The attacker sets a massive amount of amountX, and put it in the parameter
2. The `(pool_a + a_token_amount)` would exceed the `(pool_a * pool_b)`, resulting the `((pool_a * pool_b) / (pool_a + a_token_amount))` to ZERO.
3. The `swap_amount` will be the whole amount of the liquidity, since the `pool_b - ((pool_a * pool_b) / (pool_a + a_token_amount))` is `pool_b - 0`.
4. The attacker then add a minimum balances of the `tokenX` to the contract, which makes the pool exchange rate `small amount of tokenX : amountY`.
5. The attacker swaps the `tokenX` to `tokenY`, retriving all the used `tokenY` balance when robbing the `tokenX` in the pool with additional `tokenY`. 
6. As a result, the attacker gets all the liquidity inside the pool!

### Impact

Users can get a large propotion of the balamce inside the pool, which results robbing more tokens from the pool.

### Recommendation

Consider adding the require check if the amount of `newReserveX` is zero.(both in amountX->amountY and amountY->amountX swap!)

### Patch

Knowned

------------------------------------------------------------------------
# Dex_002_Mia

~~~
function addLiquidity(

uint256 amountX,

uint256 amountY,

uint256 minLPReturn

) external override refresh returns (uint256 lpAmount) {

// 유동성 풀의 비율과 일치하는지 확인합니다.

require(balanceX / amountX == balanceY / amountY);

// 유동성 풀의 비율에 맞게 자산을 분배합니다.

// token.transfer 변경분을 계산합니다.

lpAmount = Math.max(

uint256(int256(amountX) + dy),

uint256(int256(amountY) + dx)

);

require(lpAmount > 0 && lpAmount >= minLPReturn, "Invalid LP amount");
~~~

**Vector: Total LP token not tracked when removing the collateral**

**Severity Level: informational (known isssue by the Dev, check the line 7-9)**

### Description

The LP token amount, which is for the collateral investor, is not added when withdraw their collateral. the `lpBalances` of the msg.sender is used in calculating the `rx`, `ry`, but it does not includes the other LP's balances. which means the msg.sender can achieve other's collateral when withdrawal.

### Impact

The remover user can get all the collaterals which is provided in the pool.

### Recommendation

1. Consider adding the `uint256 totaLPlBalance` which can track all the LPtoken amounts.
2. Consider minting a new `IERC20 token` for LPtoken, which can view the totalSupply by a getter function.
### Patch

Knowned

------------------------------------------------------------------------
# Dex_003_Mia

~~~
function addLiquidity(

uint256 amountX,

uint256 amountY,

uint256 minLPReturn

) external override refresh returns (uint256 lpAmount) {

// 유동성 풀의 비율과 일치하는지 확인합니다.

require(balanceX / amountX == balanceY / amountY);

// 유동성 풀의 비율에 맞게 자산을 분배합니다.

// token.transfer 변경분을 계산합니다.

lpAmount = Math.max(

uint256(int256(amountX) + dy),

uint256(int256(amountY) + dx)

);

require(lpAmount > 0 && lpAmount >= minLPReturn, "Invalid LP amount");
~~~

**Vector: Hardcoded balanceRate might cause to DoS due to the high traffic or frontrunning**

**Severity Level: Critical**

### Description

The require check in the line-2 will revert when the user's add liquidity balance rate is not the same as the pool token's balance.

### Impact

If the pool has high traffic, and if there are additional swap TX in the mempool which modifies the token swap rate, the user's TX would be rejected.Furthermore, if malicious node intends to make a DoS attack, he or she could simply do it by swaping some tokens(X or Y) in order to block the user's TX. 

### Recommendation

Consider adding the additional logic to modify the user's swap token amount in the executing time, here's an example:
```
uint256 _optimalAmountY = (_amountX * reserveY) / reserveX;

uint256 _optimalAmountX = (_amountY * reserveX) / reserveY;

if (_amountY > _optimalAmountY) {

_amountY = _optimalAmountY;

} else {

_amountX = _optimalAmountX;

}
```
### Patch
()

------------------------------------------------------------------------
# Dex_004_Entropy

~~~
uint256 reserveX = tokenX.balanceOf(address(this));

uint256 reserveY = tokenY.balanceOf(address(this));

...

uint a = (amountX * LPtotalSupply) / reserveX;

uint b = (amountY * LPtotalSupply) / reserveY;
~~~

**Vector: (DoS)Malicious attacker can block user's addliquidity by sandwitch attack**

**Severity Level: Medium (The attacker can block others multiple time, but each attack requires gas fee)**

### Description

The `reserveX` and `reserveY` got it's balances from the `address(this)`, which refers the balance of the contract at that moment. This means that someone can transfer his or her tokens to the contract, which can cause DoS attack. Here's an example.

1. the attacker provides liquidity to the pool.
2. the vector(Entropy) adds liquidity to the pool.
3. the attacker sees the Tx in the pool, and frontruns a transfer to the pool.
4. Entropy tries to add liquidity to the pool, but the LPtoken amount rounds to ZERO, which results `revert` or snatched all the tokens to the Attacker (if Entropy's `minLPReturn` parameter is zero)




### Impact

The remover user can get all the collaterals which is provided in the pool.

### Recommendation

1. Consider adding the `uint256 totaLPlBalance` which can track all the LPtoken amounts.
2. Consider minting a new `IERC20 token` for LPtoken, which can view the totalSupply by a getter function.
### Patch

Knowned



------------------------------------------------------------------------
# Dex_005_Entropy

~~~
function swap(

uint256 amountX,

uint256 amountY,

uint256 minAmountOut

) external returns (uint256 amountOut) {
...

newReserveY = reserveY + amountY;

newReserveX = (reserveX * reserveY) / newReserveY;

amountOut = reserveX - newReserveX;
~~~

**Vector: The attacker can drain all the liquidity in the pool using a large amount of `amountY` or `amountX`.

**Severity Level: Critical(drain issue. especially critical to the newly created pools that has small amount of liquidities)**

### Description

The amount that the users want to swap can be decided by putting that amount in the `amountX` or `amountY` parameter, if used properly. But the Attacker can Drain all the liquidity of the tokens what we want to receive, and then drain the other side of the tokens in the pool by transfering the token. here's the attack scenario.

1. The attacker sets a massive amount of amountX, and put it in the parameter
2. The `newReserveY` would exceed the `reserveX*reserveY`, resulting the `newReserveX` to ZERO.
3. The `amountOut` will be the whole amount of the liquidity, since the `reserveX - newReserveX` is `reservewX(pool amount) - 0`.
4. The attacker then add a minimum balances of the `tokenX` to the contract, which makes the pool exchange rate `small amount of tokenX : amountY`.
5. The attacker swaps the `tokenX` to `tokenY`, retriving all the used `tokenY` balance when robbing the `tokenX` in the pool with additional `tokenY`. 
6. As a result, the attacker gets all the liquidity inside the pool!
### Impact

The remover user can get all the collaterals which is provided in the pool.

### Recommendation

Consider adding the require check if the amount of `newReserveX` is zero.(both in amountX->amountY and amountY->amountX swap!)
### Patch

Knowned




------------------------------------------------------------------------
# Dex_006_Hakid29

~~~


function swap(uint amountX, uint amountY, uint minOutput) external returns (uint output) {


...


uint newPoolY = poolY + amountY;

uint newPoolX = (poolX * poolY) / newPoolY;

output = poolX - newPoolX;


~~~

**Vector: The attacker can drain all the liquidity in the pool using a large amount of `amountY` or `amountX`.

**Severity Level: Critical(drain issue. especially critical to the newly created pools that has small amount of liquidities)**

### Description

The amount that the users want to swap can be decided by putting that amount in the `amountX` or `amountY` parameter, if used properly. But the Attacker can Drain all the liquidity of the tokens what we want to receive, and then drain the other side of the tokens in the pool by transfering the token. here's the attack scenario.

1. The attacker sets a massive amount of amountX, and put it in the parameter
2. The `newPoolY` would exceed the `reserveX*reserveY`, resulting the `newPoolX` to ZERO.
3. The `output` will be the whole amount of the liquidity, since the `poolX - newPoolX` is `poolX - 0`.
4. The attacker then add a minimum balances of the `tokenX` to the contract, which makes the pool exchange rate `small amount of tokenX : amountY`.
5. The attacker swaps the `tokenX` to `tokenY`, retriving all the used `tokenY` balance when robbing the `tokenX` in the pool with additional `tokenY`. 
6. As a result, the attacker gets all the liquidity inside the pool!
### Impact

The remover user can get all the collaterals which is provided in the pool.

### Recommendation

Consider adding the require check if the amount of `newReserveX` is zero.(both in amountX->amountY and amountY->amountX swap!)
### Patch

Knowned


------------------------------------------------------------------------
# Dex_007_Hakid29

~~~


uint poolX = tokenX.balanceOf(address(this));

uint poolY = tokenY.balanceOf(address(this));

  

uint lpTokenX = (amountX * totalSupply()) / poolX;

uint lpTokenY = (amountY * totalSupply()) / poolY;

  

lpTokens = lpTokenX < lpTokenY ? lpTokenX : lpTokenY;

}

~~~

**Vector: The attacker can rob the user's addliquidity tokens by frontrunning. or blocks the user's liquidity adding(DoS)

**Severity Level: Critical**

### Description

When the pool was first created, a malicious user add some liquidity. Then he can watch whether someone adds a liquidity to the pool, and frontruns to send the token pairs to the vault(jist sending it by transfer, not by addliquidity function.). As a result, the victim's addliquidity would fail, or not receiving any LPtokens because the denominator poolX or poolY qould exceed the numerator, resulting to mint ZERO LP's.

### Impact

The attacker could do ether DoS or robbing all the tokens without any risks.(if the attacker itself does not get frontrunned)

### Recommendation

Consider minting a **large** amount of `IERC20 LPtoken` when creating a vault, which can prevent user from attacker. (it should be large enough than the frontrunned amount!!)


++
it is similar to ERC4626 vault inflation attack, so reading the https://docs.openzeppelin.com/contracts/4.x/erc4626 is highly recommended.
Also, the Uniswap had the same problem, and the recommendation above is inspired from the UniswapV2. Check the initial LP token minting part inside the vault.

### Patch

Knowned



------------------------------------------------------------------------
# Dex_008_Icarus

~~~


uint256 private constant MINIMUM_LIQUIDITY = 1000;


~~~

**Vector: Unused variable

**Severity Level: Informational(gas adoption)**

### Description

Use this as an initial LPtoken amount, or deprecate it.

### Impact

additional gas fee

### Recommendation

same as the description

### Patch

Knowned


------------------------------------------------------------------------
# Dex_009_dokPark

~~~


if (liquidityBefore == 0 || xBefore == 0 || yBefore == 0) {

liquidityAfter = Math.sqrt(xAfter * yAfter);

} else {

uint256 liquidityX = (liquidityBefore * xAfter) / xBefore;

uint256 liquidityY = (liquidityBefore * yAfter) / yBefore;

  

liquidityAfter = (liquidityX < liquidityY)

? liquidityX

: liquidityY;

}

  

uint256 lpAmount = liquidityAfter - liquidityBefore;


~~~

**Vector: The liquidity rates of the swap can be modified when adding the liquidity

**Severity Level: High**

### Description

The `lpAmount` is calculated by the liquidityAfter and the liquidityBefore, which is deriveds from the liquidityAfter calculation logic. but when it calculates, the logic doesn't check if the added token balance rate fits the current token balance rates.

### Impact

1. the new LP provider can lose their money, since they got less LPtokens than the exceeded amount of the tokens. For example, if a user provides 1000 tokensX and 1500 tokenY to the pool which exchange rates are 1:1, the user would receive the LPtoken calculated by the tokenX balance which has lower amount than tokenY(1000<1500), resulting the loss of the exceeding amounts of tokenY. This occurs because of this logic.

~~~
liquidityAfter = (liquidityX < liquidityY)

? liquidityX

: liquidityY;

}
~~~

### Recommendation

set the check of the balance rate if it is proper, or modify the actual transfer amount from user to the pool contract.
Do **NOT** add a reverting logic if the rate is not true, because the DoS will occur due to the swap frontruinning or the high swap transaction amount of that pool. We highly recommend using the check logic like this:

~~~
uint256 _optimalAmountY = (_amountX * reserveY) / reserveX;

uint256 _optimalAmountX = (_amountY * reserveX) / reserveY;

  

if (_amountY > _optimalAmountY) {

_amountY = _optimalAmountY;

} else {

_amountX = _optimalAmountX;

}
~~~

In the code, you can see the modify logic that ensures the user to add the liquidity in a correct rate. It would be a good choice using this logic before actually transfering the the tokens to the contract.
### Patch

Knowned

----------------------------------------------------------------------------------------------


# Dex_010_Castle

~~~
//Dex.sol:swap() line 69-75

function swap(uint amountX, uint amountY, uint minOutput) public returns (uint outputAmount) {

//토큰 둘 중 하나는 반드시 0

require(amountX == 0 || amountY == 0);

if(amountX == 0){

outputAmount = swap_Y(amountY, minOutput);

}

if(amountY == 0){

outputAmount = swap_X(amountX, minOutput);

}

}


~~~

**Vector: Not reverting of the zero amount of swap

**Severity Level: Low**

### Description

There are no revert checks if the amountX or amountY has Non-zero value, which does not revert when the swap events does not occur.

### Impact

Users can lost their additional gas fee because their are no revert signs.

Plus, ina a long term, if additional Dapp wants to use this contract as an swap api, then the second Dapp will be critical to the exploit, which accurs indirect reputatuon loss to this Defi(Castle). 

### Recommendation

Consider adding the revert check before the actual swap internal logic. Here's an example.

~~~
if(amountX == 0 && amountY == 0){revert("No swap balance!");}
~~~

The check logic which is written by if is more gas efficient than the revert check, so we recommend to add this logic right after or before this logic.

~~~
require(amountX == 0 || amountY == 0);
~~~

### Patch

Knowned

------------------------------------------------------------------------
# Dex_011_jachellin

~~~


function removeLiquidity(

uint _lpReturn,

uint _minAmountX,

uint _minAmountY

) public returns (uint _tx, uint _ty) {

uint poolAmountX = tokenX.balanceOf(address(this));

uint poolAmountY = tokenY.balanceOf(address(this));

_tx = (poolAmountX * _lpReturn) / totalLP;

_ty = (poolAmountY * _lpReturn) / totalLP;

require(

_tx >= _minAmountX && _ty >= _minAmountY,

"Insufficient minimum amount"

);

require(

_lpReturn <= poolAmountX + poolAmountY,

"Insufficient LP return"

);

  

tokenX.transfer(msg.sender, _tx);

tokenY.transfer(msg.sender, _ty);

totalLP -= _lpReturn;

  

return (_tx, _ty);

}

~~~

**Vector: LPtoken does not move to the Pool contract when redeeming the liquidity.

**Severity Level: High-medium**

### Description

The LPtoken on the contract still remains after the redeem has finished. It doesn't results to a drain(the compiler compiles solidity 0.8.13, so there are over/underflow checks in `totalLP -= _lpReturn;` which prevents from draining.) but the token which is used as a LP is useless.

### Impact

additional gas fee in deploying the contract, and dummy tokens that decreases user experiences.

### Recommendation

Consider removing the ERC20 LPtokens or add a transferfrom logic to the contract.

### Patch

Knowned

------------------------------------------------------------------------

# Dex_012_kaymin

~~~
//Dex.sol:addliquidity line 27-33

reserve_x = token_x.balanceOf(address(this));

  

if (total_supply == 0) {

liquidity = sqrt(amount_x * amount_y);

} else {

liquidity = min(amount_x * total_supply / reserve_x, amount_y * total_supply / reserve_y);

}


~~~

**Vector: The tokenY added to the pool not by addliquidity function are locked and cannot be rescued

**Severity Level: medium**

### Description

The total `tokenY` balances that the pool has got are not considered when adding or removing the pool, which can cause the loss of the additional profit to the LP. 

### Impact

same as the description

### Recommendation

Consider check the balanceOf tokenY too, when doing addliquidity or removeliquidity.(the vector is too massive and it is deeply involved to the architectural part of the contract, we are afraid to give you the recommendetions.)

### Patch

Knowned

------------------------------------------------------------------------

# Dex_013_gloomydumber

~~~

function mint(address _to) external returns (uint256 liquidity) {

(uint112 _reserveX, uint112 _reserveY) = getReserves();

uint256 balanceX = IERC20(tokenX).balanceOf(address(this));

uint256 balanceY = IERC20(tokenY).balanceOf(address(this));

uint256 amountX = balanceX - _reserveX;

uint256 amountY = balanceY - _reserveY;

uint256 totalSupply = totalSupply();

  

if (totalSupply == 0) {

liquidity = Math.sqrt(amountX * amountY);

} else {

liquidity = Math.min(amountX * totalSupply / reserveX, amountY * totalSupply / reserveY);

}

  

require(liquidity > 0, "INSUFFICIENT_LIQUIDITY_MINTED");

  

_mint(_to, liquidity);

update(balanceX, balanceY);

}

~~~

**Vector: The liquidityTokens can be minted multiple times, which allows attackers pump their balances in the pool.

**Severity Level: Critical(drain issue)**

### Description

The mint function is external, and it  gives the caller the liquidityTokens by calculating the reserveX and Y. The attacker can gain the additional LP by simply calling this mint function right after the addliquidity function, in order to modify the reserveX and the reserveY balance.

### Impact

Users can get a large propotion of the balamce inside the pool, which results robbing more tokens from the pool.

### Recommendation

Consider use mint function as a internal, or just use  -mint in the ERC20 contract.

### Patch

Knowned

------------------------------------------------------------------------



# Lending_001_Entropy

~~~

uint borrowedPeriod = block.number -

userBalances[msg.sender].borrwedBlock;

userBalances[msg.sender].borrowedAsset =

(userBalances[msg.sender].borrowedAsset *

pow(INTEREST_RATE, borrowedPeriod)) /

DECIMAL; // 이자 계산

uint256 maxBorrowable = (collateralValue * 50) /

(100 * assetPrice) -

userBalances[msg.sender].borrowedAsset; // LTV = 50%

  

require(maxBorrowable >= _amount, "Insufficient collateral");

  

userBalances[msg.sender].borrwedBlock = block.number; // update the borrowedBlock

userBalances[msg.sender].borrowedAsset += _amount;

totalBorrowedUSDC += _amount;
~~~

**Vector: Users can borrow without not counting the rates.

**Severity Level: medium**

### Description

If the users. borrow twice in one TX, the block.number count will be synchronized and thus does not check the actual fee rates of the borrow. Here's the senario:

Hakid29, who had borrowed on this lending protocol before, wants to make an additional borrow, but he wants to borrow more.

1. So he first borrow a very small amount of USDC, which makes the `userBalances[msg.sender].borrwedBlock` to the currenent `block.number`.
2. The attacker then call another borrrow call inside the same TX.
3. In the second borrow, the `pow(INTEREST_RATE, borrowedPeriod))` would be 1, which results the `userBalances[msg.sender].borrowedAsset *pow(INTEREST_RATE, borrowedPeriod)) /DECIMAL;` to **Zero**. (if borrowedAsset does not exceed 1e27, the DECIMAL) 
4. In this way, the attacker can borrow more collateral than what he or she should borrow!

### Impact

The attackers can borrow the collateral in a way that exceed the actual maxborrow rates.

### Recommendation

This solution is deeply involved in your architecture, so we are afraid to give you the exact recommendations.

But there are two ways of modifing this structure.(this is only a rough comment!)

1. Follow the initial borrow block.timestamp value when calculating the rate fees, not synchronizing it.
2. Silo the borrow receipt and treat them as a different. even when the user repays or the receipt goes to a liquidiation auctions.

### Patch

Knowned

------------------------------------------------------------------------

# Lending_002_dokPark


~~~
//DreamAcademyLending.sol:DreamAcademyLending deposit(), line 51

require(msg.value >= amount, "Invalid amount");

~~~

**Vector: Users will loss their ethers if they missmatch the amount and the msg.value.

**Severity Level: Low**

### Description

If the users send 2 ether to the contract, with 1 ether amount parameter, the user's 1 ether would be vanished to the contract.

### Impact

Same on the description.

### Recommendation

We recommend you to modify the require check like this:

~~~
require(msg.value == amount, "Invalid amount");
~~~

In this way, the TX would be reverted rather than losing the additional amount of ethers.
### Patch

Knowned

------------------------------------------------------------------------

# Lending_003_Castle


~~~

function distributeInterest(uint totalInterest) internal {

// 예치된 USDC에 대한 이자를 분배

for (uint i=0; i<users.length; i++) {

address user = users[i];

uint share = (depositUSDC[user] * totalInterest) / usdctotal;

deptInterest[user] += share;

}

}

}

~~~

**Vector: Users can drain interest by making sybil accounts.

**Severity Level: High(interest drain)**

### Description

Users account are always pushed to the arrays when they deposit their accounts, while the totalcollarteral they had deposited is calculated my an mapping. In this way, users can deposit multiple times and get more interest when the distributeInterest() function is called. this is because the for logic evokes `uint share = (depositUSDC[user] * totalInterest) / usdctotal;` with the same user several times, and thus increases the `deptInterest[user] += share;` more.

### Impact

The malicious user can drain all the interest by making a sybil deposit.

### Recommendation

In a short term:
Try adding a sort function in the array, which sorts all the elements in the array without duplication. Here's an example:

~~~
//BTF Dapp, from the upsideAcademy.

function uniquifyAddresses(address[] memory addrs) external pure returns (address[] memory) {

if (addrs.length == 0) {

return new address[](0);

}

  

address[] memory uniqueAddrs = new address[](addrs.length);

uint256 uniqueCount = 0;

  

for (uint256 i = 0; i < addrs.length; i++) {

bool isDuplicate = false;

for (uint256 j = 0; j < uniqueCount; j++) {

if (addrs[i] == uniqueAddrs[j]) {

isDuplicate = true;

break;

}

}

if (!isDuplicate) {

uniqueAddrs[uniqueCount] = addrs[i];

uniqueCount++;

}

}

  

// 결과 배열의 크기를 조정

assembly {

mstore(uniqueAddrs, uniqueCount)

}

  

return uniqueAddrs;

}
~~~

In a long term:
Try modifying the users list into a struct, or mapping.

### Patch

Knowned

------------------------------------------------------------------------



# Lending_004_jachellin


~~~

// UpsideAcademyLending.sol:UpsideAcademyLending deposit() line 56-57, 65-66

if (accounts[msg.sender].depositedETH == 0) {

accountAddr.push(msg.sender);

}


...


// UpsideAcademyLending.sol:UpsideAcademyLending distributrInterest()

function distributeInterest () public {

uint interestAccrued = calTotalInterest();

  
uint interestDistributed;

for (uint i=0;i<accountAddr.length;i++) {

address distributed_account = accountAddr[i];

interestDistributed += accounts[distributed_account].interest;

}

  
uint interest_left = interestAccrued - interestDistributed;


uint total_deposited_usdc;

for (uint i=0;i<accountAddr.length;i++) {

address deposited_account = accountAddr[i];

total_deposited_usdc += accounts[deposited_account].depositedUSDC;

}

  

for (uint i=0;i<accountAddr.length;i++) {

address interest_account = accountAddr[i];

if (accounts[interest_account].depositedUSDC > 1) {

accounts[interest_account].interest += interest_left*(accounts[interest_account].depositedUSDC*1e18/total_deposited_usdc)/1e18;

  

}

}

  

}

~~~

**Vector: Users can drain interest by making sybil accounts.

**Severity Level: High(interest drain)**

### Description

The users can duplicates their accounts when getting distributed, since the `deposit()` function pushes the user's address multiple times if the users only deposited USDC or ethers.

### Impact

The users can get more interests when the distributeInterest functions are called.

### Recommendation

We recommend you to modify the depodit function, in order to first check if there are already accounts in the array, and pushes if the contract is not in there:


### Patch

Knowned

------------------------------------------------------------------------



# Lending_005_nulln0rm


~~~

for(i = 0; i < length; i++) {

user = user_info[users[i]];

timeElapsed = (block.number - user.deposit_update_time) / 7200;

if (timeElapsed == 0)

continue;

  

calc = _pow(total_borrowed, block.number / 7200 , true) - _pow(total_borrowed, user.deposit_update_time / 7200, true);

calc = calc * user.deposit_usdc / total_deposited_USDC;

user.last_accrued_interest += calc;

  

user.deposit_update_time = block.number;

user_info[users[i]] = user;

}

~~~

**Vector: The attackers can occur OOG(out of gas) DoS by simply depositing many account with a small amount of tokens or ethers.(like wei!)

**Severity Level: Critical(DoS)**

### Description

The users can deposit multiple times by different accounts, which can occur the OOG when calculating the `_depositUpdate()` function. this occurs because of the `for` check.

### Impact

The `_depositUpdate()` function is in the `liquidiate, borrow, withdrawal, repay, and also in the deposit` function, so the whole Dapp would be attacked.

### Recommendation

We recommend you to modify to erase the check function, and add or div the amount when it deposits, or withdraws. This would make the deploy gas more high, but it is safe from the OOG DoS.


### Patch

Knowned

# Lending_006_Sori

~~~

function initializeLendingProtocol(address a1) external payable{

// ???

usdc.transferFrom(msg.sender, address(this), msg.value);

}

~~~

**Vector: unused function.

**Severity Level: Informational

### Description

the logic is not necessary in Dapp.

### Impact

additional gas fee in deployment.

### Recommendation

We recommend to deprecate it, or integrate it with a constructor with initializable logic.


### Patch

Knowned

# Lending_007_kenny

~~~

function initializeLendingProtocol(address a1) external payable{

// ???

usdc.transferFrom(msg.sender, address(this), msg.value);

}

~~~

**Vector: The attackers can occur OOG(out of gas) DoS by simply depositing many account with a small amount of tokens or ethers.(like wei!)

**Severity Level: Critical(DoS)**

### Description

The users can deposit multiple times by different accounts, which can occur the OOG when calculating the `_depositUpdate()` function. this occurs because of the `for` check.

### Impact

The `_depositUpdate()` function is in the `liquidiate, borrow, withdrawal, repay, and also in the deposit` function, so the whole Dapp would be attacked.

### Recommendation

We recommend you to modify to erase the check function, and add or div the amount when it deposits, or withdraws. This would make the deploy gas more high, but it is safe from the OOG DoS.


### Patch

Knowned
