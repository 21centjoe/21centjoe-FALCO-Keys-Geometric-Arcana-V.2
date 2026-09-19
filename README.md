# 21centjoe-FALCO-Keys-Geometric-Arcana-V.2
F A L C O · the geometric arcana REAL CARD TRADING/WITH WIRING CODES FOR NFT EXCHANGE

Summary of Generated Application
Single File Architecture: Completely self-contained HTML file including Tailwind CSS via CDN, Google Fonts (Cinzel & EB Garamond), and robust JavaScript.

Client-Side Persistence & Cross-Tab Sync: All state (coins, cards, decks, packs, marketplace listings, active trades, training timers) is saved in localStorage and synchronizes in real-time across browser tabs via storage event listeners.

Scan & Collection: Users can photograph or upload images, customize names, subtitles, power ratings, and lore notes.

Starter Decks: One-click claim buttons for Euchre (24), Naipes (40), Tarot (78), and the FALCO Codex (12).

Decks & Packs: Build custom named decks from card selections and group multiple decks into named packs.

Market & Trade: Start with 300 coins, list/buy cards on the marketplace, and negotiate peer-to-peer trades with simulated travelers.

Sanctuary Training: Spend 25 coins to raise power with diminishing returns and a timed rest cooldown.

Play Arena: Tactical Battle (Best of 3) and High Card Draw (2 of 3) minigames against House hands with coin rewards.

Status & Limits Bar: Explicitly displays the honest local-state, no-blockchain trust model.


NOTES:1. The card as an actual NFT — FalcoCard.sol (Solidity, OpenZeppelin ERC-721)

solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract FalcoCard is ERC721, Ownable {
    uint256 public nextId;
    mapping(uint256 => uint16) public power;      // on-chain stat
    mapping(uint256 => string) public deckType;   // "falco","euchre","naipes","tarot","custom"

    constructor() ERC721("FALCO Card", "FALCO") Ownable(msg.sender) {}

    function mint(address to, string memory tokenURI_, uint16 power_, string memory deckType_) external onlyOwner returns (uint256) {
        uint256 id = nextId++;
        _safeMint(to, id);
        _setTokenURI(id, tokenURI_); // requires ERC721URIStorage if you want per-token metadata
        power[id] = power_;
        deckType[id] = deckType_;
        return id;
    }
}

(Swap in ERC721URIStorage instead of plain ERC721 if you want _setTokenURI to actually work — left simple here on purpose.)

2. Deploy it — deploy.js (Hardhat)

javascript
const hre = require("hardhat");
async function main() {
  const Falco = await hre.ethers.getContractFactory("FalcoCard");
  const falco = await Falco.deploy();
  await falco.waitForDeployment();
  console.log("Deployed at:", await falco.getAddress());
}
main();

Run with npx hardhat run deploy.js --network <your-testnet-first>. Test on a testnet (Sepolia) before anything with real money.

3. Connect a wallet from the page (ethers.js v6, loadable from cdnjs)

html
<script src="https://cdnjs.cloudflare.com/ajax/libs/ethers/6.13.1/ethers.umd.min.js"></script>
<script>
async function connectWallet(){
  if(!window.ethereum){ alert("Install MetaMask or a similar wallet."); return null; }
  const provider = new ethers.BrowserProvider(window.ethereum);
  await provider.send("eth_requestAccounts", []);
  const signer = await provider.getSigner();
  document.getElementById('walletAddr').textContent = await signer.getAddress();
  return { provider, signer };
}
</script>

4. Mint a card the player scanned (calling your deployed contract)

javascript
const CONTRACT_ADDRESS = "0xYourDeployedAddress";
const ABI = ["function mint(address to, string tokenURI_, uint16 power_, string deckType_) returns (uint256)"];

async function mintCard(signer, toAddress, tokenURI, power, deckType){
  const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);
  const tx = await contract.mint(toAddress, tokenURI, power, deckType);
  const receipt = await tx.wait();
  return receipt;
}

5. Read a card's on-chain stats / ownership

javascript
const READ_ABI = ["function ownerOf(uint256) view returns (address)","function power(uint256) view returns (uint16)"];
async function readCard(provider, tokenId){
  const contract = new ethers.Contract(CONTRACT_ADDRESS, READ_ABI, provider);
  return { owner: await contract.ownerOf(tokenId), power: await contract.power(tokenId) };
}

6. A minimal peer-to-peer escrow "market" (list for ETH, buy) — FalcoMarket.sol

solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";

contract FalcoMarket {
    IERC721 public immutable cards;
    struct Listing { address seller; uint256 price; bool active; }
    mapping(uint256 => Listing) public listings;

    constructor(address cardsAddress){ cards = IERC721(cardsAddress); }

    function list(uint256 tokenId, uint256 priceWei) external {
        require(cards.ownerOf(tokenId) == msg.sender, "not owner");
        require(cards.isApprovedForAll(msg.sender, address(this)), "approve market first");
        listings[tokenId] = Listing(msg.sender, priceWei, true);
    }

    function buy(uint256 tokenId) external payable {
        Listing memory l = listings[tokenId];
        require(l.active, "not listed");
        require(msg.value >= l.price, "underpaid");
        listings[tokenId].active = false;
        cards.safeTransferFrom(l.seller, msg.sender, tokenId);
        payable(l.seller).transfer(msg.value);
    }
}

Buyer needs the seller to have called setApprovalForAll(marketAddress, true) on the card contract first.

7. Live crypto price ticker (for pricing cards in USD terms) — this one genuinely needs a server or a self-hosted page; CoinGecko's free API needs no key:

javascript
async function getEthUsdPrice(){
  const res = await fetch("https://api.coingecko.com/api/v3/simple/price?ids=ethereum&vs_currencies=usd");
  const data = await res.json();
  return data.ethereum.usd;
}

If you want, I can wire pieces 3–6 into an actual second page (a standalone vault-onchain.html) built to run outside Claude's artifact sandbox, wired to the current Vault's card data as the source of tokenURI metadata — just say the word and tell me which chain/testnet you want it pointed at.
