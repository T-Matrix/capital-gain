---
title: NFT Capital Gain Royalty
description: NFT Capital Gain Royalty contract extension to ERC-721
repo link: https://github.com/T-Matrix/capital-gain
author: Liz, Vick-v3, lyxann, 0x50fc, bestysh, T-Matrix
status: Draft
type: Standards Track
category: ERC
created: 2022-11-11
requires: 165, 721
---

## Abstract

We are proposing an alternative standard ERCxxxx (hereinafter referred to as “Capital gain royalty fees''. The standard serves as an extension to ERC2981 royalty implementation. The royalty is based on a percentage of the appreciation of the subject NFT relative to a percentage applied on the sale price in the event that ERC2981 is adopted. This concept is similar to Capital Gain Tax which is widely imposed in many countries. The interface includes functions capGainRoyaltyInfo, fixedRateRoyaltyInfo for different scenarios.

## Motivation

In the ongoing debate of NFT royalties, advocates extoll how beneficial royalties are for an NFT ecosystem’s organic growth but detractors assert that royalties are exploitative and unnecessary. The reason why this debate arises is because the existing royalty structure is expressed as a rate (specified by the creator) of the sale price (hereinafter referred to as “fixed-rate royalty fee) and levied in the event of exchange of hands of the underlying NFT. This results in a misalignment of interests between the creator and the holder.

Cons of fixed-rate royalties:

it increases friction in transactions: the royalties paid by the holder is actually included in the sale price and higher royalty fees result in higher sale prices but less buyers.

the royalties paid to creators are unfair and extractive for holders, especially in a bear market where the NFT sale proceeds upon deduction of royalty fees may be even lower than the original purchase price.

On the flipside, removing royalties would hurt the very artists and creators who first turned to NFTs as a more profitable medium of selling their work. It’s not beneficial for the NFT ecosystem in the long run. Hence, here at [Hunter Club], we believe a hybrid approach, which aligns the interest of both the creator and the holder, is the solution.


## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

The implementer MUST implement capGainRoyaltyInfo  and fixedRateRoyaltyInfo. capGainRoyaltyInfo is royalty fee calculated on purcchase and sale price differnce and fixedRateRoyaltyInfo is same as EIP2981, a fixed percentage of sale price.

The implementer MAY choose to change the percentage value for both `capGainRoyaltyInfo` and `fixedRateRoyaltyInfo`
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

import "@openzeppelin/contracts/utils/introspection/IERC165.sol";

/**
 * @dev Interface for the NFT Increment Royalty Standard.
 */
interface IEIP1998 is IERC165 {

    /**
     * @dev Returns how much difference royalty is owed and to whom, based on a cost price and sale price that 
     * may be denominated in any unit of exchange. 
     * The royalty amount is denominated and should be paid in that same unit of exchange.
     * Note: using fixedRoyaltyInfo instead if costPrice doesn't exist.
     */
    function capGainRoyaltyInfo(uint256 tokenId, uint256 costPrice, uint256 salePrice)
        external
        view
        returns (address receiver, uint256 royaltyAmount);

    /**
     * @dev Returns how much fixed royalty is owed and to whom, based on a sale price that 
     * may be denominated in any unit of exchange. 
     * The royalty amount is denominated and should be paid in that same unit of exchange.
     * Note: The stub royalty is the recent royalty amount. if ignoreStub is false, then the result is :
     * Min(fixedRoyalty, recentRoyaltyAmount). Otherwise, just return fixedRoyalty.
     */
    function fixedRateRoyaltyInfo(uint256 tokenId, uint256 salePrice, bool ignoreStub)
        external
        view
        returns (address receiver, uint256 royaltyAmount);
}

```
Here an exmaple of implemenation in contract:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

import "@openzeppelin/contracts/utils/introspection/ERC165.sol";
import "./IEIP1998.sol";

abstract contract EIP1998 is IEIP1998, ERC165 {

        struct RoyaltyInfo {
        address receiver;        // receiver for tax
        uint8 diffRoyaltyRate;   // Differential tax rate, range in [0-10000]
                uint8 fixedRoyaltyRate;  // fixed tax rate, range in [0-10000]
    }

        // Using this when royalty info is not exist in _tokenRoyaltyInfo for specific tokenId.
        RoyaltyInfo private _defaultRoyaltyInfo;

        // Mapping from token ID to royalty info
        mapping(uint256 => RoyaltyInfo) private _tokenRoyaltyInfo;

        // store the recent royalty value
        uint256 private _stubRoyalty;

        /**
     * @dev See {IERC165-supportsInterface}.
     */
    function supportsInterface(bytes4 interfaceId) public view virtual override(IERC165, ERC165) returns (bool) {
        return interfaceId == type(IEIP1998).interfaceId || super.supportsInterface(interfaceId);
    }

    /**
     * @inheritdoc IEIP1998
     */
    function capGainRoyaltyInfo(
                uint256 _tokenId, 
                uint256 costPrice, 
                uint256 salePrice
        ) external view returns (address, uint256) {
                if (salePrice - costPrice <= 0) {
                        return (_defaultRoyaltyInfo.receiver, 0);
                }

                RoyaltyInfo memory royalty = _tokenRoyaltyInfo[_tokenId];
                if (royalty.receiver == address(0)) {
            royalty = _defaultRoyaltyInfo;
        }

                uint256 royaltyAmount = ((salePrice - costPrice) * royalty.diffRoyaltyRate) / uint256(_feeDenominator());

                return (royalty.receiver, royaltyAmount);
        }

        /**
     * @inheritdoc IEIP1998
     */
    function fixedRateRoyaltyInfo(
                uint256 tokenId, 
                uint256 salePrice,
                bool ignoreStub
        ) external view returns (address, uint256) {
                if (salePrice == 0) {
                        return (_defaultRoyaltyInfo.receiver, 0);
                }

                RoyaltyInfo memory royalty = _tokenRoyaltyInfo[tokenId];
                if (royalty.receiver == address(0)) {
            royalty = _defaultRoyaltyInfo;
        }

                uint256 royaltyAmount = (salePrice * royalty.fixedRoyaltyRate) / uint256(_feeDenominator());
                if (ignoreStub) {
                        return (royalty.receiver, royaltyAmount);
                }

                uint256 result = _minUint256(royaltyAmount, _stubRoyalty);
                return (royalty.receiver, result);
        }

        /**
         * @dev setter & getter for stubRoyaltyAmount
         */
        function _setStubRoyaltyAmount(uint256 stubRoyalty) internal {
                _stubRoyalty = stubRoyalty;
        }

        function _stubRoyaltyAmount() internal view returns (uint256) {
                return _stubRoyalty;
        }

        /**
         * @dev set DefaultRoyaltyInfo.
         */
        function _setDefaultRoyalty(
                address receiver, 
                uint8 diffNumber, 
                uint8 fixedNumber
        ) internal virtual {
        require(diffNumber <= _feeDenominator() && diffNumber >= 0, "EIP1998: diffNumber rate must be rage in 0-10000");
                require(fixedNumber <= _feeDenominator() && fixedNumber >= 0, "EIP1998: fixedNumber rate must be rage in 0-10000");
        require(receiver != address(0), "EIP1998: invalid receiver");

        _defaultRoyaltyInfo = RoyaltyInfo(receiver, diffNumber, fixedNumber);
    }

        /**
         * @dev set RoyaltyInfo for specific tokenId.
         */
        function _setTokenRoyalty(
                uint256 tokenId,
                address receiver, 
                uint8 diffNumber, 
                uint8 fixedNumber
        ) internal virtual {
        require(diffNumber <= _feeDenominator() && diffNumber >= 0, "EIP1998: diffNumber rate must be rage in 0-10000");
                require(fixedNumber <= _feeDenominator() && fixedNumber >= 0, "EIP1998: fixedNumber rate must be rage in 0-10000");
        require(receiver != address(0), "EIP1998: invalid receiver");

                _tokenRoyaltyInfo[tokenId] = RoyaltyInfo(receiver, diffNumber, fixedNumber);
    }

        /**
     * @dev Resets royalty information for the token id back to the global default.
     */
    function _resetTokenRoyalty(uint256 tokenId) internal virtual {
        delete _tokenRoyaltyInfo[tokenId];
    }

        /**
     * @dev get the min one between two uint256 number
     */
        function _minUint256(uint256 _x, uint256 _y) internal pure returns (uint256) {
                return _x >= _y ? _y : _x;
        }

        /**
     * @dev The denominator with which to interpret the fee set in {_setTokenRoyalty} and {_setDefaultRoyalty} as a
     * fraction of the sale price. Defaults to 10000 so fees are expressed in basis points, but may be customized by an
     * override.
     */
    function _feeDenominator() internal pure virtual returns (uint96) {
        return 10000;
    }
}
```

## Rationale
As a based royalty contract, it is impossible to get the results of every NFT transaction or to actively determine whether a particular NFT transaction is  merely wallets moving or not. Although the above features are theoretically possible, they are not suitable to be implemented in the contract, for one, for the sake of concise, scalable contracts. Moreover, if the blockchain query for each NFT transaction is implemented within this contract, it will come with high gas fees. We believe that each trading platform itself can better handle this information and is a more suitable vehicle to handle this information. For example, the trading platform can reduce unnecessary transaction queries through `IPFS` caching, thus significantly reducing gas fees.

The EIP does not specify a specific currency type, as NFT transactions may occur on various chains and the payment method of the user is uncertain. Considering that the same NFT may be traded directly in different currencies, the tax rate may need to be adjusted accordingly, so the EIP supports a single NFT to specify the tax rate:
```solidity
function _setTokenRoyalty(
    uint256 tokenId,
    address receiver, 
    uint8 diffNumber, 
    uint8 fixedNumber
) internal virtual {
    require(diffNumber <= _feeDenominator() && diffNumber >= 0, "EIP1998: diffNumber rate must be rage in 0-10000");
    require(fixedNumber <= _feeDenominator() && fixedNumber >= 0, "EIP1998: fixedNumber rate must be rage in 0-10000");
    require(receiver != address(0), "EIP1998: invalid receiver");

    _tokenRoyaltyInfo[tokenId] = RoyaltyInfo(receiver, diffNumber, fixedNumber);
}
```
Flexibility and diversity is also an important consideration of the EIP, if the access platform wants to implement only a fixed tax for individual (or all) NFTs, regardless of the logic of the retained royalty, the EIP also supports this, specifically through the `fixedRoyaltyInfo` function to achieve:
```solidity
function fixedRoyaltyInfo(
    uint256 tokenId, 
    uint256 salePrice,
    bool ignoreStub
) external view returns (address, uint256) {
    // ...
}
```
Just set `ignoreStub` to `true` for the corresponding `tokenId` when using.

## Backwards Compatibility
This standard is compatible with current ERC-721 and ERC-1155 standards.

## Security Considerations
There are no security considerations related directly to the implementation of this standard.

## Copyright

Copyright and related rights waived via [CC0](https://eips.ethereum.org/LICENSE).
