[![Join the chat at https://gitter.im/poanetwork/poa-bridge](https://badges.gitter.im/poanetwork/poa-bridge.svg)](https://gitter.im/poanetwork/poa-bridge?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://travis-ci.org/poanetwork/poa-parity-bridge-contracts.svg?branch=master)](https://travis-ci.org/poanetwork/poa-parity-bridge-contracts)

# POA Bridge Smart Contracts
These contracts provide the core functionality for the POA bridge. They implement the logic to relay assests between two EVM-based blockchain networks. The contracts collect bridge validator's signatures to approve and facilitate relay operations. 

The POA bridge smart contracts are intended to work with [the bridge process implemented on NodeJS](https://github.com/poanetwork/token-bridge).
Please refer to the bridge process documentation to configure and deploy the bridge.

## Bridge Overview

The POA Bridge allows users to transfer assets between two chains in the Ethereum ecosystem. It is composed of several elements which are located in different POA Network repositories:

**Bridge Elements**
1. Solidity smart contracts, contained in this repository.
2. [Token Bridge](https://github.com/poanetwork/token-bridge). A NodeJS oracle responsible for listening to events and sending transactions to authorize asset transfers.
3. [Bridge UI Application](https://github.com/poanetwork/bridge-ui). A DApp interface to transfer tokens and coins between chains.
4. [Bridge Monitor](https://github.com/poanetwork/bridge-monitor). A tool for checking balances and unprocessed events in bridged networks.
5. [Bridge Deployment Playbooks](https://github.com/poanetwork/deployment-bridge). Manages configuration instructions for remote deployments.

## Bridge Smart Contracts Summary

### Operations

Currently, the contracts support two types of relay operations:
* Tokenize the native coin in one blockchain network (Home) into an ERC20 token in another network (Foreign).
* Swap a token presented by an existing ERC20 contract in a Foreign network into an ERC20 token in the Home network, where one pair of bridge contracts corresponds to one pair of ERC20 tokens.


### Components

The POA bridge contracts consist of several components:
* The [**Home Bridge**](https://github.com/poanetwork/poa-bridge-contracts/blob/master/contracts/upgradeable_contracts/BasicHomeBridge.sol) smart contract. This is currently deployed in POA.Network.
  * `executeAffirmation(address recipient, uint256 value, bytes32 transactionHash) external onlyValidator`
    * validator affirm(認證) `transactionHash` 的交易發生過
    * 有足夠的validator affirm，就執行`onExecuteAffirmation`，給`recipient` `value`的錢
  * `submitSignature(bytes signature, bytes message) external onlyValidator`
    * validator在接收`UserRequestForSignature`的event後，開始簽名
    * validator提交對`message`的簽名，`signature`
    * 合約會記錄`signature`，有多少人簽了`message`
    * 當夠多人簽的時候，`emit CollectedSignatures(msg.sender, hashMsg, reqSigs)` 
      * `hashMsg`為訊息的hash 
      * `reqSigs`為至少所需的簽名數
      * 等待validator接收
  * `setMessagesSigned(bytes32 _hash, bool _status) internal`
    * 記錄某個人是否有簽某一項訊息
    * `_hash`表示某人和某訊息
    * `_status`即有沒有簽
  * `setAffirmationsSigned(bytes32 _withdrawal, bool _status) internal`
    * 記錄某個人是否有認證某一項訊息
    * `_withdrawal`表示某人和某訊息
    * `_status`即有沒有認證
  * `markAsProcessed(uint256 _v) internal pure returns(uint256)`
    * 把數量最左邊的bit設為1，表示有足夠多的人認證或簽名
  * `onExecuteAffirmation(address, uint256) internal returns(bool)`
    * 把錢給接收者
* The [**Foreign Bridge**](https://github.com/poanetwork/poa-bridge-contracts/blob/master/contracts/upgradeable_contracts/BasicForeignBridge.sol) smart contract. This is deployed in the Ethereum Mainnet.
  * `executeSignatures(uint8[] vs, bytes32[] rs, bytes32[] ss, bytes message) external`
    * 接收`CollectedSignatures`之後(也就是說，有足夠的簽名)，任何人都可以觸發relay
    * `Message.hasEnoughValidSignatures(message, vs, rs, ss, validatorContract())`確認有足夠多的簽名
    * ?validator contract 跟 home chain的相同
    * `(recipient, amount, txHash, contractAddress) = Message.parseMessage(message)`
    * `require(contractAddress == address(this))`確定發起者想relay到這個bridge
    * `require(!relayedMessages(txHash))`確定該交易還沒有被relay過
    * `require(onExecuteMessage(recipient, amount))` relay
* Depending on the type of relay operations the following components are also used:
  * in `NATIVE-TO-ERC` mode: the ERC20 token (in fact, the ERC677 extension is used) is deployed on the Foreign network;
  * in `ERC-TO-ERC` mode: the ERC20 token (in fact, the ERC677 extension is used) is deployed on the Home network;
* [Basic Bridge](https://github.com/poanetwork/poa-bridge-contracts/blob/master/contracts/upgradeable_contracts/BasicBridge.sol): 在鏈上記載bridge規則 
  * owner可以決定一些參數，如: gas price，單筆交易的最大最小金額，一日交易的量等
  * `claimTokens(address _token, address _to) public onlyOwner`
    * owner可以把合約的以太或ERC20 `_token`給`_to`
* The [**Validators**](https://github.com/poanetwork/poa-bridge-contracts/blob/master/contracts/upgradeable_contracts/BridgeValidators.sol) smart contract is deployed in both the POA.Network and the Ethereum Mainnet.
  * 在鏈上記載validator的規則
  * owner可以任意決定validators(加入、移除)，以及validator認證的最少數量
  * 判斷某地址是否為validator
* [HomeBridgeNativeToErc](https://github.com/poanetwork/poa-bridge-contracts/blob/master/contracts/upgradeable_contracts/native_to_erc20/HomeBridgeNativeToErc.sol)
  * `contract HomeBridgeNativeToErc is EternalStorage, BasicBridge, BasicHomeBridge`
    * 繼承HomeBridge, 在POA的網路上
  * `initialize (
        address _validatorContract,
        uint256 _dailyLimit,
        uint256 _maxPerTx,
        uint256 _minPerTx,
        uint256 _homeGasPrice,
        uint256 _requiredBlockConfirmations
    ) public
      returns(bool)`
    * 設定bridge的validator跟重要的參數
    * `_validatorContract`: 決定bridge的validators
  * `function () public payable`
    * fallback function
    * `setTotalSpentPerDay(getCurrentDay(), totalSpentPerDay(getCurrentDay()).add(msg.value))`
      * 把錢放到bridge裡，當天可以花的錢增加
    * `emit UserRequestForSignature(msg.sender, msg.value)`
      * 發event等validators收
  * `onExecuteAffirmation(address _recipient, uint256 _value)`
    * 給 `_recipient` `_value` 的錢
* ForeignBridgeNativeToErc
  * `contract ForeignBridgeNativeToErc is ERC677Receiver, BasicBridge, BasicForeignBridge, ERC677Bridge`
    * 繼承ForeignBridge, 在乙太坊的網路上
    * `initialize(
        address _validatorContract,
        address _erc677token,
        uint256 _dailyLimit,
        uint256 _maxPerTx,
        uint256 _minPerTx,
        uint256 _foreignGasPrice,
        uint256 _requiredBlockConfirmations
    ) public returns(bool)`
      * 設定bridge的validator跟重要的參數
      * `_validatorContract`: 決定bridge的validators
  * `claimTokensFromErc677(address _token, address _to) external onlyOwner`
  * `onExecuteMessage(address _recipient, uint256 _amount) internal returns(bool)`
  * `fireEventOnTokenTransfer(address _from, uint256 _value) internal`
    * `emit UserRequestForAffirmation(_from, _value)`
### Bridge Roles and Responsibilities

Responsibilities and roles of the bridge:
- **Administrator** role (representation of a multisig contract):
  - add/remove validators
  - set daily limits on both bridges
  - set maximum per transaction limit on both bridges
  - set minimum per transaction limit on both bridges
  - upgrade contracts in case of vulnerability
  - set minimum required signatures from validators in order to relay a user's transaction
- **Validator** role:
  - provide 100% uptime to relay transactions
  - listen for `UserRequestForSignature` events on Home Bridge and sign an approval to relay assets on Foreign network
  - listen for `CollectedSignatures` events on Home Bridge. As soon as enough signatures are collected, transfer all collected signatures to the Foreign Bridge contract.
  - listen for `UserRequestForAffirmation` or `Transfer` (depending on the bridge mode) events on the Foreign Bridge and send approval to Home Bridge to relay assets from Foreign Network to Home
- **User** role:
  - sends assets to Bridge contracts:
    - in `NATIVE-TO-ERC` mode: send native coins to the Home Bridge to receive ERC20 tokens from the Foreign Bridge, send ERC20 tokens to the Foreign Bridge to unlock native coins from the Home Bridge;
    - in `ERC-TO-ERC` mode: transfer ERC20 tokens to the Foreign Bridge to mint ERC20 tokens on the Home Network, transfer ERC20 tokens to the Home Bridge to unlock ERC20 tokens on Foreign networks. 

## Usage

### Install Dependencies
```bash
npm install
```
### Deploy
Please the [README.md](deploy/README.md) in the `deploy` folder for instructions and .env file configuration

### Test
```bash
npm test
```

### Flatten
```bash
npm run flatten
```

## Contributing

See the [CONTRIBUTING](CONTRIBUTING.md) document for contribution, testing and pull request protocol.

## License

[![License: GPL v3.0](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

This project is licensed under the GNU General Public License v3.0. See the [LICENSE](LICENSE) file for details.



