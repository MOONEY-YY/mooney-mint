// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Burnable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract MOONEY is ERC20, ERC20Burnable, Ownable {
    error MaxMintCountExceeded();

    address public immutable PAYMENT_TOKEN;
    uint256 public immutable MINT_AMOUNT = 10_000 * 10**18; // 10k tokens
    uint256 public immutable MAX_MINT_COUNT = 5000;
    uint256 public constant PAYMENT_PER_MINT = 5_000_000; // 5 USDC (6 decimals)
    uint256 public mintCount;
    uint256 public constant POOL_SEED_AMOUNT = 50_000_000 * 10**18; // 50M for manual LP

    event Minted(address indexed to, uint256 amount);

    constructor(
        address _paymentToken
    ) ERC20("MOONEY", "MOONEY") Ownable(msg.sender) {
        PAYMENT_TOKEN = _paymentToken;
        // 预留50M给合约 (manual LP)
        _mint(address(this), POOL_SEED_AMOUNT);
    }

    // Public mint: 付费mint
    function mint() external {
        if (mintCount >= MAX_MINT_COUNT) revert MaxMintCountExceeded();

        // 用户approve后, transfer 5 USDC to contract
        IERC20(PAYMENT_TOKEN).transferFrom(msg.sender, address(this), PAYMENT_PER_MINT);

        // 2.5 USDC to owner
        IERC20(PAYMENT_TOKEN).transfer(owner(), PAYMENT_PER_MINT / 2);

        // mint 10k to user
        _mint(msg.sender, MINT_AMOUNT);
        mintCount++;
        emit Minted(msg.sender, MINT_AMOUNT);
    }

    // Withdraw USDC (anytime, owner only)
    function withdrawUSDC(uint256 amount) external onlyOwner {
        IERC20(PAYMENT_TOKEN).transfer(owner(), amount);
    }

    // Manual LP helper: Transfer tokens to owner for Uniswap add
    function withdrawTokensForLP() external onlyOwner {
        _transfer(address(this), owner(), POOL_SEED_AMOUNT);
    }
}
