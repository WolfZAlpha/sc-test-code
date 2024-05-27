# sc-test-code

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract ProsperaToken is ERC20, ERC20Burnable, ERC20Pausable, Ownable, ReentrancyGuard {
    using SafeMath for uint256;

    uint256 private constant _totalSupply = 1000000000 * 10**18; // Updated total supply
    IERC20 public usdcToken;
    address public taxWallet;

    uint256 public constant burnRate = 3; // 3% burn rate
    uint256 public constant taxRate = 6; // 6% tax rate

    mapping(address => bool) private _blacklist;

    struct Stake {
        uint256 amount;
        uint256 startTime;
        uint256 lockDuration;
    }

    mapping(address => Stake) private _stakes;
    mapping(address => uint256) private _stakeRewards;

    event Staked(address indexed user, uint256 amount, uint256 total);
    event Unstaked(address indexed user, uint256 amount, uint256 total);
    event RevenueShared(address indexed user, uint256 amount);
    event TokensLocked(address indexed user, uint256 amount, uint256 lockDuration);
    event TokensPurchased(address indexed buyer, uint256 amount, uint256 price);
    event BlacklistUpdated(address indexed user, bool value);

    constructor(IERC20 _usdcToken, address _taxWallet) ERC20("Prospera", "PROS") {
        _mint(msg.sender, _totalSupply);
        usdcToken = _usdcToken;
        taxWallet = _taxWallet;
    }

    modifier notBlacklisted(address account) {
        require(!_blacklist[account], "Blacklisted address");
        _;
    }

    function _transfer(address sender, address recipient, uint256 amount) internal virtual override notBlacklisted(sender) notBlacklisted(recipient) {
        require(sender != address(0), "Transfer from the zero address");
        require(recipient != address(0), "Transfer to the zero address");

        uint256 burnAmount = amount.mul(burnRate).div(100);
        uint256 taxAmount = amount.mul(taxRate).div(100);
        uint256 transferAmount = amount.sub(burnAmount).sub(taxAmount);

        // Burn the tokens
        _burn(sender, burnAmount);

        // Transfer the tax to the tax wallet
        _safeTransferETH(taxWallet, taxAmount);

        // Transfer the remaining tokens to the recipient
        super._transfer(sender, recipient, transferAmount);
    }

    function mint(address to, uint256 amount) external onlyOwner {
        require(to != address(0), "Mint to the zero address");
        _mint(to, amount);
    }

    function addToBlacklist(address account) external onlyOwner {
        require(account != address(0), "Blacklist the zero address");
        _blacklist[account] = true;
        emit BlacklistUpdated(account, true);
    }

    function removeFromBlacklist(address account) external onlyOwner {
        require(account != address(0), "Remove from blacklist the zero address");
        _blacklist[account] = false;
        emit BlacklistUpdated(account, false);
    }

    function stake(uint256 amount) external nonReentrant whenNotPaused notBlacklisted(msg.sender) {
        require(amount > 0, "Cannot stake 0 tokens");
        _burn(msg.sender, amount);
        _stakes[msg.sender].amount = _stakes[msg.sender].amount.add(amount);
        _stakes[msg.sender].startTime = block.timestamp;
        emit Staked(msg.sender, amount, _stakes[msg.sender].amount);
    }

    function unstake(uint256 amount) external nonReentrant whenNotPaused notBlacklisted(msg.sender) {
        require(amount > 0, "Cannot unstake 0 tokens");
        require(_stakes[msg.sender].amount >= amount, "Insufficient staked amount");
        require(block.timestamp >= _stakes[msg.sender].startTime.add(_stakes[msg.sender].lockDuration), "Tokens are still locked");

        uint256 reward = calculateReward(msg.sender, amount);
        _mint(msg.sender, amount.add(reward));
        _stakes[msg.sender].amount = _stakes[msg.sender].amount.sub(amount);
        emit Unstaked(msg.sender, amount, _stakes[msg.sender].amount);
    }

    function calculateReward(address staker, uint256 amount) public view returns (uint256) {
        uint256 stakingDuration = block.timestamp.sub(_stakes[staker].startTime);
        uint256 rewardRate = getRewardRate(amount, _stakes[staker].lockDuration);
        return amount.mul(rewardRate).mul(stakingDuration).div(1 days).div(10000); // assuming rewardRate is in basis points
    }

    function getRewardRate(uint256 amount, uint256 lockDuration) public pure returns (uint256) {
        uint256 baseRate;
        if (amount >= 500000 * 10**18) {
            baseRate = 25; // 2.5% per day
        } else if (amount >= 100000 * 10**18) {
            baseRate = 19; // 1.9% per day
        } else if (amount >= 50000 * 10**18) {
            baseRate = 16; // 1.6% per day
        } else if (amount >= 20000 * 10**18) {
            baseRate = 13; // 1.3% per day
        } else if (amount >= 5000 * 10**18) {
            baseRate = 8;  // 0.8% per day
        } else {
            baseRate = 3;  // 0.3% per day
        }

        // Increase rate based on lock duration
        if (lockDuration >= 365 days) {
            return baseRate.mul(2); // 2x for 1 year lock
        } else if (lockDuration >= 180 days) {
            return baseRate.mul(15).div(10); // 1.5x for 6 months lock
        } else {
            return baseRate;
        }
    }

    function lockTokens(uint256 amount, uint256 lockDuration) external nonReentrant whenNotPaused notBlacklisted(msg.sender) {
        require(amount > 0, "Cannot lock 0 tokens");
        require(lockDuration >= 30 days, "Lock duration must be at least 30 days");
        _burn(msg.sender, amount);
        _stakes[msg.sender].amount = _stakes[msg.sender].amount.add(amount);
        _stakes[msg.sender].startTime = block.timestamp;
        _stakes[msg.sender].lockDuration = lockDuration;
        emit TokensLocked(msg.sender, amount, lockDuration);
    }

    function distributeRevenue(uint256 amount) external onlyOwner nonReentrant whenNotPaused {
        require(usdcToken.balanceOf(address(this)) >= amount, "Insufficient USDC balance");
        uint256 totalStaked = getTotalStaked();
        require(totalStaked > 0, "No tokens staked");

        for (uint256 i = 0; i < totalStakers.length; i++) {
            address staker = totalStakers[i];
            uint256 share = _stakes[staker].amount.mul(amount).div(totalStaked);
            usdcToken.transfer(staker, share);
            emit RevenueShared(staker, share);
        }
    }

    function getTotalStaked() public view returns (uint256) {
        uint256 total = 0;
        for (uint256 i = 0; i < totalStakers.length; i++) {
            total = total.add(_stakes[totalStakers[i]].amount);
        }
        return total;
    }

    function burn(uint256 amount) public override onlyOwner {
        super.burn(amount);
    }

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    function totalStaked(address staker) external view returns (uint256) {
        return _stakes[staker].amount;
    }

    // ICO functionality
    enum IcoTier { Tier1, Tier2, Tier3 }
    uint256 public constant icoSupply = 153750000 * 10**18;
    uint256 public constant tier1Tokens = 40000000 * 10**18;
    uint256 public constant tier2Tokens = 50000000 * 10**18;
    uint256 public constant tier3Tokens = 63750000 * 10**18;
    uint256 public constant tier1Price = 0.02 ether;
    uint256 public constant tier2Price = 0.08 ether;
    uint256 public constant tier3Price = 0.16 ether;
    uint256 public tier1Sold;
    uint256 public tier2Sold;
    uint256 public tier3Sold;
    bool public icoActive = true;
    IcoTier public currentTier = IcoTier.Tier1;

    function buyTokens(uint256 tokenAmount) external payable nonReentrant whenNotPaused notBlacklisted(msg.sender) {
        require(icoActive, "ICO is not active");
        uint256 cost;

        if (currentTier == IcoTier.Tier1) {
            require(tier1Sold.add(tokenAmount) <= tier1Tokens, "Not enough Tier 1 tokens");
            cost = tokenAmount.mul(tier1Price).div(10**18);
            tier1Sold = tier1Sold.add(tokenAmount);

            if (tier1Sold >= tier1Tokens) {
                currentTier = IcoTier.Tier2;
            }
        } else if (currentTier == IcoTier.Tier2) {
            require(tier2Sold.add(tokenAmount) <= tier2Tokens, "Not enough Tier 2 tokens");
            cost = tokenAmount.mul(tier2Price).div(10**18);
            tier2Sold = tier2Sold.add(tokenAmount);

            if (tier2Sold >= tier2Tokens) {
                currentTier = IcoTier.Tier3;
            }
        } else if (currentTier == IcoTier.Tier3) {
            require(tier3Sold.add(tokenAmount) <= tier3Tokens, "Not enough Tier 3 tokens");
            cost = tokenAmount.mul(tier3Price).div(10**18);
            tier3Sold = tier3Sold.add(tokenAmount);

            if (tier3Sold >= tier3Tokens) {
                icoActive = false;
            }
        } else {
            revert("Invalid ICO tier");
        }

        require(msg.value == cost, "Incorrect ETH amount sent");

        _mint(msg.sender, tokenAmount);
        emit TokensPurchased(msg.sender, tokenAmount, cost);
    }

    function endIco() external onlyOwner {
        icoActive = false;
    }

    receive() external payable {}

    fallback() external payable {}

    function _isHolder(address account) internal view returns (bool) {
        for (uint256 i = 0; i < holders.length; i++) {
            if (holders[i] == account) {
                return true;
            }
        }
        return false;
    }

    function _safeTransferETH(address to, uint256 amount) internal {
        (bool success, ) = to.call{value: amount}("");
        require(success, "ETH transfer failed");
    }
}
