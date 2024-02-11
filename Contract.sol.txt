pragma solidity ^0.8.0;

interface IUniswapV2Factory {
    function createPair(address tokenA, address tokenB) external returns (address pair);
}

interface IUniswapV2Router02 {
    function WETH() external pure returns (address);
    function addLiquidityETH(
        address token,
        uint amountTokenDesired,
        uint amountTokenMin,
        uint amountETHMin,
        address to,
        uint deadline
    ) external payable returns (uint amountToken, uint amountETH, uint liquidity);
}

interface IERC20 {
    function transfer(address to, uint256 value) external returns (bool);
    function approve(address spender, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract BeeMeme {
    string public constant name = "BeeMeme";
    string public constant symbol = "BMC";
    uint8 public constant decimals = 18;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    uint256 public rewardFee = 2; // 2% holder reward
    uint256 public liquidityFee = 3; // 3% liquidity fee

    address public uniswapV2Router;
    address public uniswapV2Pair;
    address public immutable WETH;

    mapping(address => uint256) public lastTransactionTime;
    uint256 public constant maxTransactionAmount = 1000 * 10**18; // Example limit: 1000 BMC

    bool public paused = false;
    address public owner;

    modifier notPaused() {
        require(!paused || msg.sender == owner, "Contract is paused");
        _;
    }

    modifier cooldown() {
        require(block.timestamp > lastTransactionTime[msg.sender] + 5 minutes, "Cooldown period has not elapsed");
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only contract owner can perform this action");
        _;
    }

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event RewardPaid(address indexed holder, uint256 amount);

    constructor(uint256 initialSupply, address _uniswapV2Router) {
        totalSupply = initialSupply * 10 ** uint256(decimals);
        balanceOf[msg.sender] = totalSupply;
        owner = msg.sender;
        uniswapV2Router = _uniswapV2Router;
        WETH = IUniswapV2Router02(_uniswapV2Router).WETH();
        uniswapV2Pair = IUniswapV2Factory(IUniswapV2Router02(_uniswapV2Router).factory())
            .createPair(address(this), WETH);
    }

    function _transfer(address _from, address _to, uint256 _value) internal {
        require(_from != address(0), "Invalid sender address");
        require(_to != address(0), "Invalid recipient address");
        require(_value > 0, "Invalid amount");
        require(_value <= balanceOf[_from], "Insufficient balance");

        uint256 transferAmount = _value;
        uint256 rewardAmount = transferAmount * rewardFee / 100;
        uint256 liquidityAmount = transferAmount * liquidityFee / 100;

        balanceOf[_from] -= transferAmount;
        balanceOf[_to] += transferAmount - rewardAmount; // Transfer minus reward

        // Reward to holder
        balanceOf[address(this)] += rewardAmount;
        emit RewardPaid(address(this), rewardAmount);

        emit Transfer(_from, _to, transferAmount - rewardAmount);

        if (_to == uniswapV2Pair) {
            // Add liquidity if transferring to Uniswap pair
            _addLiquidity(_from, liquidityAmount);
        }

        lastTransactionTime[_from] = block.timestamp;
        lastTransactionTime[_to] = block.timestamp;
    }

    function _addLiquidity(address _from, uint256 _amount) internal {
        // approve token transfer to cover all possible scenarios
        _approve(_from, uniswapV2Router, _amount);

        // add the liquidity
        IUniswapV2Router02(uniswapV2Router).addLiquidityETH{value: _amount}(
            address(this),
            _amount,
            0, // slippage is unavoidable
            0, // slippage is unavoidable
            address(this),
            block.timestamp
        );
    }

    function transfer(address _to, uint256 _value) public notPaused cooldown returns (bool success) {
        require(_value <= maxTransactionAmount, "Exceeds maximum transaction amount");
        _transfer(msg.sender, _to, _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) public notPaused cooldown returns (bool success) {
        require(_value <= maxTransactionAmount, "Exceeds maximum transaction amount");
        require(_value <= allowance[_from][msg.sender], "Allowance exceeded");
        _approve(_from, msg.sender, allowance[_from][msg.sender] - _value);
        _transfer(_from, _to, _value);
        return true;
    }

    function _approve(address _owner, address _spender, uint256 _amount) internal {
        require(_owner != address(0), "Invalid owner address");
        require(_spender != address(0), "Invalid spender address");

        allowance[_owner][_spender] = _amount;
        emit Approval(_owner, _spender, _amount);
    }

    function approve(address _spender, uint256 _amount) public notPaused cooldown returns (bool success) {
        _approve(msg.sender, _spender, _amount);
        return true;
    }

    function pause() public onlyOwner {
        paused = true;
    }

    function unpause() public onlyOwner {
        paused = false;
    }

    function setFees(uint256 _rewardFee, uint256 _liquidityFee) external onlyOwner {
        require(_rewardFee + _liquidityFee < 100, "Total fees cannot exceed 100%");
        rewardFee = _rewardFee;
        liquidityFee = _liquidityFee;
    }

    function emergencyWithdraw(address _token, uint256 _amount) public onlyOwner {
        require(_token != address(this), "Cannot withdraw contract token");
        IERC20(_token).transfer(owner, _amount);
    }
}
