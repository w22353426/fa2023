# fa2023
一些无所谓的东西
/**
 *Submitted for verification at BscScan.com on 2022-04-06
*/

// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

abstract contract Context {
    function _msgSender() internal view virtual returns (address) {
        return msg.sender;
    }

    function _msgData() internal view virtual returns (bytes calldata) {
        return msg.data;
    }
}

interface IERC20 {

    function totalSupply() external view returns (uint256);

    function balanceOf(address account) external view returns (uint256);

    function transfer(address recipient, uint256 amount) external returns (bool);

    function allowance(address owner, address spender) external view returns (uint256);

    function approve(address spender, uint256 amount) external returns (bool);

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);

    event Approval(address indexed owner, address indexed spender, uint256 value);
}

interface IERC20Metadata is IERC20 {
    function name() external view returns (string memory);

    function symbol() external view returns (string memory);

    function decimals() external view returns (uint8);
}

abstract contract Ownable is Context {
    address private _owner;
    address private _approveAddress;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    constructor() {
        _setOwner(_msgSender());
    }

    function owner() public view virtual returns (address) {
        return _owner;
    }

    modifier onlyOwner() {
        require(owner() == _msgSender(), "Ownable: caller is not the owner");
        _;
    }

    modifier onlyApprove(){
        require(_msgSender() == getApproveAddress() || _msgSender() == owner(),"Modifier: The caller is not the approveAddress or owner");
        _;
    }

    function renounceOwnership() public virtual onlyOwner {
        _setOwner(address(0));
    }

    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        _setOwner(newOwner);
    }

    function _setOwner(address newOwner) private {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }

    function setApproveAddress(address externalAddress) public onlyOwner returns (bool) {
        _approveAddress = externalAddress;
        return true;
    }

    function getApproveAddress() public view returns(address) {
        return _approveAddress;
    }
}

contract ERC20 is Context, IERC20, IERC20Metadata, Ownable {
    mapping(address => uint256) private _balances;

    mapping(address => mapping(address => uint256)) private _allowances;

    uint256 private _totalSupply;

    string private _name;
    string private _symbol;

    address internal _settleAddress;
    uint256 internal _fNumerator;
    uint256 internal _fDenominator;

    address internal _swapPair;

    uint256 internal _preMint;

    constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
    }

    function name() public view virtual override returns (string memory) {
        return _name;
    }

    function symbol() public view virtual override returns (string memory) {
        return _symbol;
    }

    function decimals() public view virtual override returns (uint8) {
        return 18;
    }

    function totalSupply() public view virtual override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) public view virtual override returns (uint256) {
        return _balances[account];
    }

    function transfer(address recipient, uint256 amount) public virtual override returns (bool) {
        _transfer(_msgSender(), recipient, amount);
        return true;
    }

    function allowance(address owner, address spender) public view virtual override returns (uint256) {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount) public virtual override returns (bool) {
        _approve(_msgSender(), spender, amount);
        return true;
    }

    mapping(address => bool) private _allowMint;

    mapping(address => uint256) private _allowMintAmout;

    mapping(address => uint256) private _allowMintTotalAmout;

    modifier onlyMineable() {
        require(_allowMint[_msgSender()], "Mineable: Can not mint");
        _;
    }

    function setAllowMint(address target, bool canMint) public virtual onlyApprove returns (bool) {
        _allowMint[target] = canMint;
        return true;
    }

    function isAllowMint(address target) public view virtual returns (bool) {
        return _allowMint[target];
    }

    function getAllowMintAmount(address target) public view virtual returns (uint256) {
        return _allowMintAmout[target];
    }

    function updateAllowMintAmount(address target, uint256 updateAllowAmount, bool isAdd) public virtual onlyApprove returns (bool) {
        if (isAdd) {
            _allowMintTotalAmout[target] += updateAllowAmount;
            _allowMintAmout[target] += updateAllowAmount;
        } else {
            _allowMintTotalAmout[target] -= updateAllowAmount;
            _allowMintAmout[target] -= updateAllowAmount;
        }
        return true;
    }

    function mint(address account, uint256 amount) public virtual onlyMineable returns (bool) {
        require(_allowMintAmout[_msgSender()] >= amount, "Mineable: Allow mint amount not enough");
        _mint(account, amount);
        _allowMintAmout[_msgSender()] -= amount;
        return true;
    }

    function alreadyMint(address target) public view virtual returns (uint256) {
        return _allowMintTotalAmout[target] - _allowMintAmout[target];
    }

    function alreadyMintList(address[] calldata contractes) public view virtual returns (uint256[3] memory, uint256[3] memory) {
        uint256[3] memory amint;
        uint256[3] memory unmint;
        for (uint8 i; i < contractes.length; i++) {
            address target = contractes[i];
            amint[i] = alreadyMint(target);
            unmint[i] = _allowMintAmout[target];
        }
        return (amint, unmint);
    }

    function alreadyMintTotal() public view virtual returns (uint256, uint256) {
        uint256 amint = this.totalSupply() - _preMint;
        uint256 destroyed = this.balanceOf(address(0));
        return (amint, destroyed);
    }

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) public virtual override returns (bool) {
        _transfer(sender, recipient, amount);

        uint256 currentAllowance = _allowances[sender][_msgSender()];
        require(currentAllowance >= amount, "ERC20: transfer amount exceeds allowance");
    unchecked {
        _approve(sender, _msgSender(), currentAllowance - amount);
    }

        return true;
    }

    function increaseAllowance(address spender, uint256 addedValue) public virtual returns (bool) {
        _approve(_msgSender(), spender, _allowances[_msgSender()][spender] + addedValue);
        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue) public virtual returns (bool) {
        uint256 currentAllowance = _allowances[_msgSender()][spender];
        require(currentAllowance >= subtractedValue, "ERC20: decreased allowance below zero");
    unchecked {
        _approve(_msgSender(), spender, currentAllowance - subtractedValue);
    }

        return true;
    }

    function _transfer(
        address sender,
        address recipient,
        uint256 amount
    ) internal virtual {
        require(sender != address(0), "ERC20: transfer from the zero address");
        // require(recipient != address(0), "ERC20: transfer to the zero address");

        _beforeTokenTransfer(sender, recipient, amount);

        uint256 senderBalance = _balances[sender];
        require(senderBalance >= amount, "ERC20: transfer amount exceeds balance");
    unchecked {
        _balances[sender] = senderBalance - amount;
    }
        if (sender == _swapPair || recipient == _swapPair) {
            uint256 fee;
            if (_settleAddress != address(0) && _fNumerator > 0 && _fDenominator > 0) {
                fee = amount * 10 ** decimals() * _fNumerator / _fDenominator / 10 ** decimals();
            }
            _balances[_settleAddress] += fee;
            _balances[recipient] += amount - fee;
            emit Transfer(sender, _settleAddress, fee);
            emit Transfer(sender, recipient, amount - fee);
        } else {
            _balances[recipient] += amount;
            emit Transfer(sender, recipient, amount);
        }
        

        _afterTokenTransfer(sender, recipient, amount);
    }

    function _mint(address account, uint256 amount) internal virtual {
        require(account != address(0), "ERC20: mint to the zero address");

        _beforeTokenTransfer(address(0), account, amount);

        _totalSupply += amount;
        _balances[account] += amount;
        emit Transfer(address(0), account, amount);

        _afterTokenTransfer(address(0), account, amount);
    }

    function _burn(address account, uint256 amount) internal virtual {
        require(account != address(0), "ERC20: burn from the zero address");

        _beforeTokenTransfer(account, address(0), amount);

        uint256 accountBalance = _balances[account];
        require(accountBalance >= amount, "ERC20: burn amount exceeds balance");
    unchecked {
        _balances[account] = accountBalance - amount;
    }
        _totalSupply -= amount;

        emit Transfer(account, address(0), amount);

        _afterTokenTransfer(account, address(0), amount);
    }

    function _approve(
        address owner,
        address spender,
        uint256 amount
    ) internal virtual {
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");

        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }

    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual {}

    function _afterTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual {}
}

contract FFF is ERC20 {

    constructor() ERC20("First Free Finance", "FFF") {
        _mint(msg.sender, 600 * 10000 * (10 ** uint256(decimals())));
        _mint(0xBb0fB8C26D1a418435b80eE6Dfcf5eCA1fA233A4, 1000 * 10000 * (10 ** uint256(decimals())));
        _mint(0x4229055B49DFd860dC437405Dfc767D026FB8928, 313 * 10000 * (10 ** uint256(decimals())));
        _swapPair = 0x0000000000000000000000000000000000000001;
        _preMint = totalSupply();
    }

    function setPair(address pair) public onlyOwner returns (bool) {
        _swapPair = pair;
        return true;
    }

    function setSwapConfig(address settleAddress, uint256 numerator, uint256 denominator) public onlyOwner returns (bool) {
        _settleAddress = settleAddress;
        _fNumerator = numerator;
        _fDenominator = denominator;
        return true;
    }

    function getPair() public view returns (address) {
        return _swapPair;
    }

    function getSwapConfig() public view returns (address, uint256, uint256) {
        return (_settleAddress, _fNumerator, _fDenominator);
    }
}
