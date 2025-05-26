// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IYieldProtocol {
    function deposit(uint amount) external;
    function withdraw(uint amount) external;
    function getAPY() external view returns (uint);
    function balanceOf(address user) external view returns (uint);
}

contract YieldCraft {
    address public owner;
    address public strategist;

    IERC20 public stableToken;

    struct Protocol {
        address protocolAddress;
        uint allocated;
        bool active;
    }

    Protocol[] public protocols;

    mapping(address => uint) public userDeposits;
    uint public totalDeposits;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not authorized");
        _;
    }

    modifier onlyStrategist() {
        require(msg.sender == strategist || msg.sender == owner, "Only strategist can call");
        _;
    }

    constructor(address _token) {
        owner = msg.sender;
        strategist = msg.sender;
        stableToken = IERC20(_token);
    }

    function addProtocol(address _protocol) external onlyOwner {
        protocols.push(Protocol(_protocol, 0, true));
    }

    function deposit(uint amount) external {
        require(amount > 0, "Invalid amount");

        stableToken.transferFrom(msg.sender, address(this), amount);
        userDeposits[msg.sender] += amount;
        totalDeposits += amount;
    }

    function withdraw(uint amount) external {
        require(userDeposits[msg.sender] >= amount, "Insufficient balance");

        // Withdraw from protocols proportionally
        for (uint i = 0; i < protocols.length; i++) {
            if (protocols[i].active && protocols[i].allocated > 0) {
                uint share = (protocols[i].allocated * amount) / totalDeposits;
                IYieldProtocol(protocols[i].protocolAddress).withdraw(share);
                protocols[i].allocated -= share;
            }
        }

        stableToken.transfer(msg.sender, amount);
        userDeposits[msg.sender] -= amount;
        totalDeposits -= amount;
    }

    function rebalance() external onlyStrategist {
        // Step 1: Withdraw all
        for (uint i = 0; i < protocols.length; i++) {
            if (protocols[i].active && protocols[i].allocated > 0) {
                IYieldProtocol(protocols[i].protocolAddress).withdraw(protocols[i].allocated);
                protocols[i].allocated = 0;
            }
        }

        // Step 2: Find best protocol (highest APY)
        uint bestAPY = 0;
        uint bestIndex = 0;

        for (uint i = 0; i < protocols.length; i++) {
            if (protocols[i].active) {
                uint apy = IYieldProtocol(protocols[i].protocolAddress).getAPY();
                if (apy > bestAPY) {
                    bestAPY = apy;
                    bestIndex = i;
                }
            }
        }

        // Step 3: Deposit all into best protocol
        uint balance = stableToken.balanceOf(address(this));
        if (balance > 0) {
            stableToken.approve(protocols[bestIndex].protocolAddress, balance);
            IYieldProtocol(protocols[bestIndex].protocolAddress).deposit(balance);
            protocols[bestIndex].allocated = balance;
        }
    }

    function setStrategist(address _strategist) external onlyOwner {
        strategist = _strategist;
    }

    function getUserDeposit(address user) external view returns (uint) {
        return userDeposits[user];
    }

    function getTotalProtocols() external view returns (uint) {
        return protocols.length;
    }
}

interface IERC20 {
    function transferFrom(address sender, address recipient, uint amount) external returns (bool);
    function transfer(address recipient, uint amount) external returns (bool);
    function approve(address spender, uint amount) external returns (bool);
    function balanceOf(address account) external view returns (uint);
}
