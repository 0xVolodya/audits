<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="https://repository-images.githubusercontent.com/614106114/7d126b02-af69-4eb2-814e-aa5ee8dab294" /></td>
        <td>
            <h1>Canto Audit Report</h1>
            <h2>Subprotocols for Canto Identity Protocol.</h2>
            <p>Prepared by: 0xVolodya, Independent Security Researcher</p>
            <p>Date: Mar 17 to Mar 20, 2023</p>
        </td>
    </tr>
</table>

# About Canto
The audit covers three subprotocols for the Canto Identity Protocol:

Canto Bio Protocol: Allows the association of a biography to an identity
Canto Profile Picture Protocol: Allows the association of a profile picture (arbitrary NFT) to an identity
Canto Namespace Protocol: A subprotocol for minting names from tiles (characters in a specific font).

# Summary of Findings

| ID          | Title                        | Severity | Fixed |
|-------------| ---------------------------- |----------| ----- |
| [H-01]| Users will be able to purchase fewer NFTs than the project had anticipated | High     |  ✓ |
| [M-01]| Bio NFT incorrectly breaks SVG lines and doesn't support more than 120 characters effectively | Medium   |  ✓ |

# Detailed Findings

## [H-01] Users will be able to purchase fewer NFTs than the project had anticipated

## Impact
Users will be able to purchase fewer NFTs than the project had anticipated. The project had expected that users would be able to purchase a range of variations using both text and emoji characters. However, in reality, users will only be able to purchase a range of variations using emoji characters.

For example, the list of characters available for users to choose from is as follows
![image](https://i.ibb.co/NjnD4Tf/Screenshot-from-2023-03-20-00-22-32.png)

For instance, if a user chooses to mint an NFT namespace using font class 2 and the single letter 𝒶, then theoretically all other users should be able to mint font class 0 using the first emoji in the list, font class 1 using the single letter "a," font class 3 using the single letter 𝓪, and so on, the first letter on every class will be. However, in reality, they will not be able to do so.

I consider this to be a critical issue because the project may not be able to sell as many NFTs as expected, potentially resulting in a loss of funds.

## Proof of Concept
This is a function that creates namespace out of tray.
```solidity
canto-namespace-protocol/src/Namespace.sol#L110
    function fuse(CharacterData[] calldata _characterList) external {
        uint256 numCharacters = _characterList.length;
        if (numCharacters > 13 || numCharacters == 0) revert InvalidNumberOfCharacters(numCharacters);
        uint256 fusingCosts = 2**(13 - numCharacters) * 1e18;
        SafeTransferLib.safeTransferFrom(note, msg.sender, revenueAddress, fusingCosts);
        uint256 namespaceIDToMint = ++nextNamespaceIDToMint;
        Tray.TileData[] storage nftToMintCharacters = nftCharacters[namespaceIDToMint];
        bytes memory bName = new bytes(numCharacters * 33); // Used to convert into a string. Can be 33 times longer than the string at most (longest zalgo characters is 33 bytes)
        uint256 numBytes;
        // Extract unique trays for burning them later on
        uint256 numUniqueTrays;
        uint256[] memory uniqueTrays = new uint256[](_characterList.length);
        for (uint256 i; i < numCharacters; ++i) {
            bool isLastTrayEntry = true;
            uint256 trayID = _characterList[i].trayID;
            uint8 tileOffset = _characterList[i].tileOffset;
            // Check for duplicate characters in the provided list. 1/2 * n^2 loop iterations, but n is bounded to 13 and we do not perform any storage operations
            for (uint256 j = i + 1; j < numCharacters; ++j) {
                if (_characterList[j].trayID == trayID) {
                    isLastTrayEntry = false;
                    if (_characterList[j].tileOffset == tileOffset) revert FusingDuplicateCharactersNotAllowed();
                }
            }
            Tray.TileData memory tileData = tray.getTile(trayID, tileOffset); // Will revert if tileOffset is too high
            uint8 characterModifier = tileData.characterModifier;

            if (tileData.fontClass != 0 && _characterList[i].skinToneModifier != 0) {
                revert CannotFuseCharacterWithSkinTone();
            }

            if (tileData.fontClass == 0) {
                // Emoji
                characterModifier = _characterList[i].skinToneModifier;
            }
            bytes memory charAsBytes = Utils.characterToUnicodeBytes(0, tileData.characterIndex, characterModifier);
...
```
[canto-namespace-protocol/src/Namespace.sol#L110](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Namespace.sol#L110)

There is a bug in this line of code where a character is retrieved from tile data. Instead of passing `tileData.fontClass`, we are passing `0`.

```solidity
            bytes memory charAsBytes = Utils.characterToUnicodeBytes(0, tileData.characterIndex, characterModifier);
```
Due to this bug, the names for all four different font classes will be the same. As a result, they will point to an existing namespace, and later, there will be a check for the existence of that name (token) using NameAlreadyRegistered.

```solidity
        string memory nameToRegister = string(bName);
        uint256 currentRegisteredID = nameToToken[nameToRegister];
        if (currentRegisteredID != 0) revert NameAlreadyRegistered(currentRegisteredID);
```

## Tools Used
Manual review, forge tests
## Recommended Mitigation Steps
Pass font class instead of 0
```diff
-            bytes memory charAsBytes = Utils.characterToUnicodeBytes(0, tileData.characterIndex, characterModifier);
+            bytes memory charAsBytes = Utils.characterToUnicodeBytes(tileData.fontClass, tileData.characterIndex, characterModifier);
```

## [M-01] Bio NFT incorrectly breaks SVG lines and doesn't support more than 120 characters effectively

## Impact
Bio NFT incorrectly breaks SVG lines and doesn't support more than 120 characters effectively.

## Proof of Concept
According to the docs
> Any user can mint a Bio NFT by calling Bio.mint and passing his biography. It needs to be shorter than 200 characters.

Let's take two strings and pass them to create an NFT. The first one is 200 characters long, and the second one is 120 characters long.
`aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaW`
`aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaWaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaW`

This is how they will look like. As you can see they look identical.
![image](https://i.ibb.co/Cvw62Rz/Screenshot-from-2023-03-19-12-20-26.png)

Next, lets take this text for which we create nft. I took it from a test and double. `012345678901234567890123456789012345678👨‍👩‍👧‍👧012345678901234567890123456789012345678👨‍👩‍👧‍👧`

Here is on the left how it looks now vs how it suppose to be. As you can you line breaking doesn't work. I did enlarge
viewBox so you can see the difference.

![image](https://i.ibb.co/XDNxLWx/Screenshot-from-2023-03-19-12-28-42.png)

The problem is in this part of the code, where `(i > 0 && (i + 1) % 40 == 0)` doesn't handle properly because you want
to include emojis, so length will be more than 40 (`40 + length(emoji)`)
```solidity
canto-bio-protocol/src/Bio.sol#L56
       for (uint i; i < lengthInBytes; ++i) {
           bytes1 character = bioTextBytes[i];
           bytesLines[bytesOffset] = character;
           bytesOffset++;
           if ((i > 0 && (i + 1) % 40 == 0) || prevByteWasContinuation || i == lengthInBytes - 1) {
               bytes1 nextCharacter;
```
[canto-bio-protocol/src/Bio.sol#L56](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-bio-protocol/src/Bio.sol#L56)
Lastly, the NFT doesn't center-align text, but I believe it should. I took text from a test and on the left is how it currently appears, while on the right is how I think it should be.
![image](https://i.ibb.co/8r1jMhc/Screenshot-from-2023-03-18-13-43-21.png)

Here is the code. dy doesn't apply correctly; it should be 0 for the first line.
```solidity
canto-bio-protocol/src/Bio.sol#L104
       for (uint i; i < lines; ++i) {
           text = string.concat(text, '<tspan x="50%" dy="20">', strLines[i], "</tspan>");
       }
```
[canto-bio-protocol/src/Bio.sol#L104](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-bio-protocol/src/Bio.sol#L104)
## Tools Used
Manual review
## Recommended Mitigation Steps
Enlarge viewBox so it will support 200 length or restrict to 120 characters.
Here is a complete code with correct line breaking and center text. I'm sorry that I didn't add `differ` to code because there will be too many lines. It does pass tests and fix current issues
```solidity
   function tokenURI(uint256 _id) public view override returns (string memory) {
       if (_ownerOf[_id] == address(0)) revert TokenNotMinted(_id);
       string memory bioText = bio[_id];
       bytes memory bioTextBytes = bytes(bioText);
       uint lengthInBytes = bioTextBytes.length;
       // Insert a new line after 40 characters, taking into account unicode character
       uint lines = (lengthInBytes - 1) / 40 + 1;
       string[] memory strLines = new string[](lines);
       bool prevByteWasContinuation;
       uint256 insertedLines;
       // Because we do not split on zero-width joiners, line in bytes can technically be much longer. Will be shortened to the needed length afterwards
       bytes memory bytesLines = new bytes(80);
       uint bytesOffset;
       uint j;
       for (uint i; i < lengthInBytes; ++i) {
           bytesLines[bytesOffset] = bytes1(bioTextBytes[i]);
           bytesOffset++;
           j+=1;
           if ((j>=40) || prevByteWasContinuation || i == lengthInBytes - 1) {
               bytes1 nextCharacter;
               if (i != lengthInBytes - 1) {
                   nextCharacter = bioTextBytes[i + 1];
               }
               if (nextCharacter & 0xC0 == 0x80) {
                   // Unicode continuation byte, top two bits are 10
                   prevByteWasContinuation = true;
                   continue;
               } else {
                   // Do not split when the prev. or next character is a zero width joiner. Otherwise, 👨‍👧‍👦 could become 👨>‍👧‍👦
                   // Furthermore, do not split when next character is skin tone modifier to avoid 🤦‍♂️
🏻
                   if (
                       // Note that we do not need to check i < lengthInBytes - 4, because we assume that it's a valid UTF8 string and these prefixes imply that another byte follows
                       (nextCharacter == 0xE2 && bioTextBytes[i + 2] == 0x80 && bioTextBytes[i + 3] == 0x8D) ||
                       (nextCharacter == 0xF0 &&
                           bioTextBytes[i + 2] == 0x9F &&
                           bioTextBytes[i + 3] == 0x8F &&
                           uint8(bioTextBytes[i + 4]) >= 187 &&
                           uint8(bioTextBytes[i + 4]) <= 191) ||
                       (i >= 2 &&
                           bioTextBytes[i - 2] == 0xE2 &&
                           bioTextBytes[i - 1] == 0x80 &&
                           bioTextBytes[i] == 0x8D)
                   ) {
                       prevByteWasContinuation = true;
                       continue;
                   }
               }

               assembly {
                   mstore(bytesLines, bytesOffset)
               }
               strLines[insertedLines++] = string(bytesLines);
               bytesLines = new bytes(80);
               prevByteWasContinuation = false;
               bytesOffset = 0;
               j=0;
           }
       }
       string
           memory svg = '<svg xmlns="http://www.w3.org/2000/svg" preserveAspectRatio="xMinYMin meet" viewBox="0 0 400 100"><style>text { font-family: sans-serif; font-size: 12px; }</style>';
       string memory text = '<text x="50%" y="50%" dominant-baseline="middle" text-anchor="middle">';
       text = string.concat(text, '<tspan x="50%" dy="0">', strLines[0], "</tspan>");// center first line and than add dy
       for (uint i=1; i < lines; ++i) {
           text = string.concat(text, '<tspan x="50%" dy="20">', strLines[i], "</tspan>");
       }
       string memory json = Base64.encode(
           bytes(
               string.concat(
                   '{"name": "Bio #',
                   LibString.toString(_id),
                   '", "description": "',
                   bioText,
                   '", "image": "data:image/svg+xml;base64,',
                   Base64.encode(bytes(string.concat(svg, text, "</text></svg>"))),
                   '"}'
               )
           )
       );
       return string(abi.encodePacked("data:application/json;base64,", json));
   }

```